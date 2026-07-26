# k8s-l3-dual-homing-lab

A containerlab lab that proves out **L3 dual-homing for Kubernetes worker nodes**:
each node eBGP-peers two Top-of-Rack switches (different ASNs), gets an ECMP default,
and advertises its Pod CIDRs / LoadBalancer VIPs into a spine-leaf Clos — with BFD
fast failover. K8s is a Kind cluster; CNI + BGP control plane is Cilium, with a
node-local FRR relaying routes to the fabric.

The payoff: a ToR can die and established TCP connections don't even reset. The
trade-offs, the relay design, and how it compares to MetalLB/frr-k8s are in
**[docs/architecture.md](docs/architecture.md)**.

## Routing-On-The-Host model

This lab follows a **Routing-On-The-Host** model: every Kubernetes worker runs its own
BGP daemon (FRR) and participates in the underlay fabric as a first-class router. Instead
of relying on L2 tricks (bonds, MC-LAG, or VLAN trunking) to hide the dual uplinks behind
a single logical link, each worker eBGP-peers both ToRs directly, receives an ECMP
default route, and advertises its Pod CIDRs and LoadBalancer VIPs into the spine-leaf
fabric — all from inside the node's own network namespace.

The CNI (Cilium) injects routes into the kernel; the node-local FRR picks them up via a
loopback eBGP session and re-advertises them out both physical legs. The host is not a
dumb endpoint — it is an active participant in the routing fabric, making per-packet
forwarding decisions across both uplinks.

## Full lab topology

```
                          ┌─────────────────┐         ┌──────────────────┐
                          │  spine-fabric    │         │  spine-border    │
                          │  AS 65200        │═10.99.0.8═│  AS 65201        │
                          │  (hub, default-  │  /31    │  (cp uplink +    │
                          │   origin, SNAT)  │         │   SNAT egress)   │
                          └──┬──┬──┬──┬──────┘         └────────┬─────────┘
                             │  │  │  │                         │
              ┌──────────────┘  │  │  └──────────────┐          │
              │  eth1            │  │  eth4           │  eth5    │ eth2
              │  /31             │  │  /31            │  /31     │ /31
        ┌─────┴──────┐   ┌─────┴──┴─────┐   ┌──────┴─────┐   ┌─┴────────────┐
        │rack1-tor-a │   │rack1-tor-b   │   │rack2-tor-a │   │rack2-tor-b   │
        │AS 65101    │   │AS 65102      │   │AS 65101    │   │AS 65102      │
        └──┬─────┬───┘   └──┬──────┬────┘   └──┬─────┬───┘   └──┬──────┬────┘
           │     │          │      │            │     │          │      │
           │eth1 │eth1      │eth1  │eth1        │eth1 │eth1      │eth1  │eth1
           │     │          │      │            │     │          │      │
      ┌────┴──┐  │     ┌────┴──┐   │       ┌────┴──┐  │     ┌────┴──┐   │
      │k8s-   │  │     │k8s-   │   │       │k8s-   │  │     │k8s-   │   │
      │worker │  │     │worker │   │       │worker2│  │     │worker2│   │
      │AS 65001│  │     │AS 65001│  │       │AS 65001│  │     │AS 65001│  │
      │(dual- │  │     │(dual- │   │       │(dual- │  │     │(dual- │   │
      │homed) │  │     │homed) │   │       │homed) │  │     │homed) │   │
      └───────┘  │     └───────┘   │       └───────┘  │     └───────┘   │
                 │eth2             │eth2               │eth2             │eth2
                 │/31              │/31                │/31              │/31
                 │                 │                   │                 │
           ┌─────┴─────┐    ┌─────┴──────┐      ┌─────┴─────┐    ┌─────┴──────┐
           │rack1-tor-b│    │rack1-tor-a │      │rack2-tor-b│    │rack2-tor-a │
           └───────────┘    └────────────┘      └───────────┘    └────────────┘

                            RACK 1                           RACK 2

  k8s-control-plane ──eth1/31──▶ spine-border:eth2
  (single-homed, AS 65001)
```

## Dual homing — how each worker connects

Each worker eBGP-peers two ToRs (different ASNs) over its two physical legs. FRR
on the node re-advertises Cilium-learned routes out both legs, giving the fabric
ECMP reachability back. BFD (~150 ms detection) triggers atomic ECMP reprogramming
on a leg failure — in-flight flows reroute without RST.

```
                        ┌──────────────── k8s-worker (AS 65001) ─────────────────┐
                        │                                                         │
  Cilium (AS 64512) ────┤ 127.0.0.1 BGP ──▶ FRR                                  │
                        │                    │                                    │
                        │  dummy0 10.99.255.1/32  (advertised node loopback)      │
                        │                    │                                    │
                        │         eth1                    eth2                     │
                        │        10.99.0.0/31            10.99.0.2/31             │
                        └────────────┼────────────────────────┼──────────────────┘
                                     │ eBGP                   │ eBGP
                                     ▼ (AS 65101)             ▼ (AS 65102)
                              ┌─────────────┐          ┌─────────────┐
                              │rack1-tor-a  │          │rack1-tor-b  │
                              │             │          │             │
                              └──────┬──────┘          └──────┬──────┘
                                     │                        │
                                     └───────────┬────────────┘
                                                 ▼
                                         spine-fabric (AS 65200)

  kernel default = ECMP { via rack1-tor-a, via rack1-tor-b }   ← active-active
  BFD 50ms×3 (~150ms) per leg; FRR atomic ECMP re-program on leg drop
  Pod CIDRs + LB VIPs advertised out BOTH legs → fabric ECMPs toward the node
```

## What's in here

| Path | What |
|------|------|
| `clab-dual-tor-kind.clab.yml` | the containerlab topology (6 FRR fabric nodes + Kind cluster) |
| `configs/<node>/` | per-node FRR config (`frr.conf`, `daemons`) — one dir per node |
| `images/kind-node/` | custom Kind node image (FRR baked in) |
| `configs/cilium-bgp.yaml`, `configs/kind-config.yaml` | Cilium BGP CRs; Kind cluster config |
| `Justfile` | every workflow (build / deploy / cutover / cilium / per-node queries) |
| `docs/` | architecture, findings, design specs, topology, decision log |

## How to use

Requires containerlab 0.77+, Docker, `kind` 0.32+, `just`, ~4 GB RAM.

One-shot end-to-end bring-up:

```sh
just deploy-all       # build -> deploy -> cutover -> cilium-install -> cilium-bgp
```

Or step by step:

```sh
just build            # build the Kind node image (FRR baked in)
just deploy           # bring up the topology (fabric + Kind cluster)
just cutover          # move the cluster onto the fabric: copy FRR configs, assign
                      # fabric IPs, start FRR, repoint kubelets, API-server proxy,
                      # write the kubeconfig, and drop eth0 (pure fabric mode —
                      # egress then flows via the spine SNAT)

just cilium-install   # CNI + BGP control plane + Pod IP pool
just cilium-bgp       # apply the Cilium BGP CRs
```

Verify / demo:

```sh
just sessions         # BGP sessions across the fabric
just routes           # routing / BGP tables
just cilium-bgp-status
just pods
just cp-pool-test     # (demo) prove multi-pool IPAM — CP node draws from cp-pool
```

> `just cutover` already writes the kubeconfig (pointed at the spine-border proxy).
> The standalone `just kubeconfig` is only for the pre-fabric flow — running it after
> cutover overwrites the working one with kind's local endpoint.

Per-node helpers: `just k8s-worker-sessions`, `just rack1-tor-a-routes`,
`just spine-fabric-cmd "<vtysh cmd>"`, etc. Run `just` (or `just --list`) for all.

## Resources

**Talks & Videos**
- [Building Layer 3 Only Baremetal Kubernetes Clusters](https://www.youtube.com/watch?v=7prUnxglfCk&t=4s) — Nokia, KubeCon talk covering BGP-based L3 dual-homing for K8s workers on bare metal

**Project Documentation**
- [Containerlab](https://containerlab.dev/) — The lab orchestration tool used to deploy this topology
- [Cilium BGP Control Plane](https://docs.cilium.io/en/stable/network/bgp-toc/index.html) — Cilium's built-in BGP speaker (GoBGP) for advertising Pod CIDRs and LB VIPs
- [MetalLB BGP Mode](https://metallb.universe.tf/concepts/bgp/) — MetalLB's BGP-based load balancing for bare-metal K8s
- [FRR-K8s](https://github.com/metallb/frr-k8s) — Kubernetes-native wrapper around FRR; allows multiple actors to share a single FRR instance per node
- [MetalLB Split FRR Proposal](https://github.com/metallb/metallb/blob/main/design/splitfrr-proposal.md) — Design proposal behind the FRR-K8s split (the pattern this lab's relay is compared to)
- [FRRouting](https://frrouting.org/) — The BGP daemon used by this lab (and by MetalLB/FRR-K8s)
