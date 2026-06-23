---
title: An eBPF (XDP + tc-BPF) Dataplane for dpservice — a Cilium-inspired alternative

iep-number: 21

creation-date: 2026-06-23

status: draft

authors:

- "@trevex"

reviewers:

- ""

---


# IEP-21: An eBPF (XDP + tc-BPF) Dataplane for dpservice — a Cilium-inspired alternative

> [!IMPORTANT]
> This side-quest was caused by a discussion during business travel last week and the PoC of the idea was heavily "vibe-coded" and is in no way production ready.
> Some of the ideas were also discussed with some IronCore Maintainers a long time ago, so hopefully this can also start a discussion in regards to which direction the dataplane should head. If I remember correctly there were also discussions in regards to Flat IPv6 and taking a Cilium inspired approach back then.

> **Purpose of this document.** This is a *discussion-starting* enhancement proposal, not a request to replace anything. It documents a working proof-of-concept that re-implements the IronCore L3 SDN dataplane (`dpservice`) in **eBPF** — using **native XDP on the uplink and tc-BPF on the guest edge**, the same split Cilium and Calico use — and tries to lay out an honest comparison with the existing DPDK dataplane so maintainers and contributors can decide whether this direction is worth pursuing, and if so, in what form. **It explicitly does not argue that DPDK is the wrong choice.** The goal here is to surface the *practical and operational* trade-offs and ask: is there a deployment profile where a kernel-native dataplane is the better fit, and is maintaining it worth the cost?

---

## Table of Contents

- [Summary](#summary)
- [Motivation](#motivation)
    - [What dpservice is today](#what-dpservice-is-today)
    - [Why explore an alternative at all](#why-explore-an-alternative-at-all)
    - [Goals](#goals)
    - [Non-goals](#non-goals)
- [Background: the two dataplane models](#background-the-two-dataplane-models)
    - [DPDK / dpservice (userspace poll + HW offload)](#dpdk--dpservice-userspace-poll--hw-offload)
    - [eBPF XDP + tc-BPF (kernel-native)](#ebpf-xdp--tc-bpf-kernel-native)
- [Proposal: the eBPF dataplane (what was built)](#proposal-the-ebpf-dataplane-what-was-built)
    - [The hybrid: native XDP uplink + tc-BPF guest edge](#the-hybrid-native-xdp-uplink--tc-bpf-guest-edge)
    - [Why the guest edge must be tc, not native XDP (a key finding)](#why-the-guest-edge-must-be-tc-not-native-xdp-a-key-finding)
    - [Composable, host-tested core (engineering note)](#composable-host-tested-core-engineering-note)
    - [Wire compatibility](#wire-compatibility)
- [State of the PoC](#state-of-the-poc)
- [DPDK vs eBPF/XDP — a balanced comparison](#dpdk-vs-ebpfxdp--a-balanced-comparison)
    - [At a glance](#at-a-glance)
    - [Performance — the honest version](#performance--the-honest-version)
    - [Hardware offload — clearing up a common confusion](#hardware-offload--clearing-up-a-common-confusion)
    - [Operational differences (where eBPF is strongest)](#operational-differences-where-ebpf-is-strongest)
- [DPUs / SmartNICs — how both interact](#dpus--smartnics--how-both-interact)
    - [The shared truth](#the-shared-truth)
    - [DPDK + DPU](#dpdk--dpu)
    - [eBPF/XDP + DPU](#ebpfxdp--dpu)
    - [Why this matters to the decision](#why-this-matters-to-the-decision)
- [Future improvements / roadmap](#future-improvements--roadmap)
- [Is it worth doing? (evaluation)](#is-it-worth-doing-evaluation)
- [Open questions for maintainers & contributors](#open-questions-for-maintainers--contributors)
- [Alternatives considered](#alternatives-considered)
- [Appendix A — PoC repositories & artifacts](#appendix-a--poc-repositories--artifacts)
- [Appendix B — Glossary](#appendix-b--glossary)

## Summary

We built a wire-compatible eBPF re-implementation of `dpservice`: it speaks the same `DPDKironcore` gRPC contract (so `metalnet` drives it unchanged), implements the full guest-facing dataplane (ARP/ND, DHCPv4/v6, IPv4/IPv6 overlay encap, NAT64, conntrack, NAT, VIP, LB, VNI, firewall, metering), and **passes the existing `dpservice` conformance suite 93/93**. It runs in a forked ironcore-in-a-box with DHCP + VM↔VM connectivity validated end-to-end.

Architecturally it follows the **Cilium model**: **native XDP on the physical/uplink device** (the offloadable, fast-path place) and **tc-BPF (clsact) on the guest endpoint device** (the GSO-safe, encryption-ready place). This split is not a stylistic choice — it is forced by a kernel reality we document below (native XDP cannot intercept a VM's egress on a vhost-backed tap).

The proposal asks three questions:
1. Are the **operational** differences (no hugepages/CPU-pinning, kernel-native tooling, simpler debugging, easier encryption, Rust memory safety) valuable enough to IronCore's users to justify a second dataplane?
2. Where does this sit relative to DPDK on **performance** and **hardware offload** — honestly?
3. How do **both** approaches interact with **DPUs/SmartNICs**, where much of this is heading?

---

## Motivation

### What dpservice is today

`dpservice` is IronCore's L3 SDN dataplane: a DPDK application that binds **SR-IOV Virtual Functions** as virtual ports, terminates an IP-in-IPv6 overlay, and answers ARP/ND/DHCP, does conntrack, NAT, VIPs, load-balancing, NAT64 and metering in userspace at line rate, offloading established flows to the NIC via `rte_flow`. It is driven by `metalnet` over the `DPDKironcore` gRPC API. It is mature, fast, and in production.

### Why explore an alternative at all

Three recurring themes motivate looking at a kernel-native option:

- **Operational surface.** DPDK requires hugepages, dedicated/pinned CPU cores, a specific NIC PMD + driver binding, and tracks the DPDK release cadence. Standard Linux tooling (`tcpdump`, `tc`, `ss`, `bpftool`, conntrack tooling) does not see DPDK-owned traffic. For smaller nodes, edge sites, or operators without deep DPDK expertise, this is real friction. We particularly have issues with this in Fog/Edge infrastructure, where we still rely on `KubeVirt` instead.
- **The ecosystem has converged on eBPF for the SDN endpoint.** Cilium and Calico run their datapaths as **tc-BPF on the pod/VM endpoint device + XDP on the physical NIC**. The patterns, tooling, and operational knowledge are widely shared. An IronCore eBPF dataplane would be legible to that audience.
- **Encryption and kernel integration.** IPsec (XFRM, incl. NIC inline-crypto) and WireGuard live on the kernel skb path. A kernel-native dataplane composes with them; a userspace-bypass dataplane must re-implement or hand off.

None of these say "DPDK is wrong." They say "there may be a deployment profile where kernel-native wins, and it is now cheap enough to build and measure."

### Goals

- Demonstrate a **wire-compatible** dataplane (same gRPC contract, same conformance suite) so it is a true drop-in for evaluation.
- Quantify the **operational** differences concretely.
- Provide an **honest** performance/offload comparison, including unknowns.
- Map how **both** DPDK and eBPF interact with **DPUs**.
- Give maintainers enough to decide: pursue, shelve, or pursue-as-experimental.

### Non-goals

- Not proposing to remove or deprecate dpservice.
- Not claiming throughput parity with DPDK (not yet measured; likely lower).
- Not a finished, production-hardened artifact — it is a PoC at conformance parity.

---

## Background: the two dataplane models

### DPDK / dpservice (userspace poll + HW offload)

- **Ports:** SR-IOV VFs (or vhost-user for VMs); the guest's virtio rings are polled directly in userspace — no kernel skb, no per-packet syscalls.
- **Fast path:** software for the first packet, then `rte_flow` programs the NIC/eswitch flow tables so subsequent packets are switched in hardware.
- **Strengths:** highest throughput and lowest latency; mature HW-offload via `rte_flow`; deterministic performance under pinning.
- **Costs:** hugepages, isolated CPUs, PMD/driver binding, DPDK version coupling, opaque to kernel tooling.

### eBPF XDP + tc-BPF (kernel-native)

- **Ports:** kernel netdevs — the physical NIC/uplink, and a tap/veth per guest.
- **Fast path:** **XDP** on the physical device (pre-`skb`, near the driver) for the hot/offloadable path; **tc-BPF (clsact/tcx)** on the endpoint device for policy, encap, and responders.
- **Strengths:** no hugepages/pinning; standard kernel tooling and observability; composes with XFRM/WireGuard, conntrack, routing; Rust memory-safety in the control plane; gradual rollout (it's "just" a program attached to a device).
- **Costs:** generally lower peak throughput than DPDK; the kernel verifier constrains program complexity; **native XDP hardware offload is effectively dead** (see [Hardware offload](#hardware-offload--clearing-up-a-common-confusion)); maturity gap.

---

## Proposal: the eBPF dataplane (what was built)

### The hybrid: native XDP uplink + tc-BPF guest edge

```
            ┌─────────────────────────── hypervisor (kernel) ───────────────────────────┐
  overlay   │   uplink netdev (dtap0)               guest tap (dtapvf_N)   ┌──────────┐ │
  (IP-in-   │   ┌───────────────────┐   redirect     ┌─────────────────┐   │   VM     │ │
   IPv6) ───┼──►│  uplink_rx (XDP)  │──────────────► │ tc_guest_tx     │◄──┤ virtio   │ │
            │   │  decap, LB, ct,   │                │ (tc / clsact)   │──►│ (vhost)  │ │
            │   │  NAT64 ingress    │                │ ARP/ND/DHCP,    │   └──────────┘ │
            │   └───────────────────┘                │ encap, ct, NAT, │                │
            │            ▲                           │ VIP, LB, NAT64  │                │
            │            └──────── redirect ─────────┴─────────────────┘                │
            └───────────────────────────────────────────────────────────────────────────┘
```

- **`uplink_rx` runs as native XDP** on the physical/uplink device — the correct, well-supported, fast place for XDP (this is where Cilium/Katran put it).
- **`tc_guest_tx` runs as tc-BPF on the guest tap's clsact ingress** — it sees the guest's egress as `skb`s and can both forward (encap → redirect to uplink) and inject replies (ARP/ND/DHCP) back to the guest.

### Why the guest edge *must* be tc, not native XDP (a key finding)

Native XDP on a **vhost-net-backed tap only runs on the "datacopy" fast path** — small, single-page, **non-GSO** frames. A stock virtio guest negotiates GSO/checksum offload, so vhost builds an `skb` and the native XDP hook (`tun_xdp_one`) is **never reached** — the program silently sees nothing. (Kernel-source-confirmed; the only NIC that ever offloaded XDP itself was Netronome NFP, now EOL.) **tc-BPF hooks after the `skb` exists**, so it sees 100% of guest traffic regardless of GSO — which is exactly why Cilium uses tc-BPF on endpoint devices. 

### Composable, host-tested core (engineering note)

The datapath is structured as a **pure, context-free core** (parsing, conntrack/NAT/VIP/LB math, packet serializers) that is **unit-tested on the host with no kernel**, plus thin per-attach-point glue for XDP vs tc. Where layout is fixed (ARP/ND/DHCPv4/encap) the core is pure and shared; where it is variable-offset (DHCPv6) or size-changing (NAT64) the byte logic stays in the eBPF crate, parameterized over a small `PktStore` trait so XDP and tc share one implementation. Written in **Rust + aya**, `#![no_std]` eBPF, verified on kernel 7.0.11 (attaches via `tcx`).

### Wire compatibility

The eBPF dataplane implements the **same `DPDKironcore` gRPC service** dpservice exposes. `metalnet` creates/deletes interfaces, routes, NATs, LBs, etc. against it unchanged. This is what makes it a genuine drop-in and lets the **existing conformance suite** run against it.

---

## State of the PoC

| Aspect | Status |
|---|---|
| Guest-edge features on tc | ARP · ND · DHCPv4 · DHCPv6 · IPv4 + IPv6 overlay egress · NAT64 ✅ |
| Ingress / uplink (XDP) | decap, delivery, LB (Maglev), NAT64 ingress, ICMP echo for VIPs ✅ |
| Control-plane features | conntrack, NAT, VIP, LB, VNI, firewall, metering, neighbor-NAT, dual-stack ✅ |
| **dpservice conformance suite** | **93 passed / 2 skipped — full parity with the XDP path AND the tc path** |
| End-to-end lab (ironcore-in-a-box) | Two VMs DHCP (`10.0.0.1`/`10.0.0.2`) and VM↔VM ping, **on tc, no SKB mode** ✅ |
| `serve` cutover | tc guest edge is the **default**; `XDP_DP_GUEST_TC=0` opts back to XDP `guest_tx` |
| Debug tooling | `XDP_DP_DEBUG` (aya-log) datapath tracing, behind a build feature |
| Host unit tests | pure core (DHCP/ARP/ND serializers, conntrack/NAT math) tested with `cargo test`, no root |

**What is NOT done / known gaps and risks:**
- **No throughput/latency benchmark vs DPDK.** Expectation: lower peak pps than DPDK; the gap and where it bites are unmeasured. This is the single most important open data point.
- **No hardware offload implemented.** "Offload-readiness" today means "the architecture is compatible with the tc-flower/switchdev offload path," not that offload exists.
- **Validated on a single-host kind lab.** Multi-host overlay, scale (many interfaces/flows), and failure/HA behavior under churn are not yet exercised beyond conformance.
- **Maturity gap.** dpservice is production-proven; this is a PoC at conformance parity. Verifier limits, map sizing, and corner cases at scale need real soak testing.
- **virtio/vhost coupling.** The PoC uses a kernel tap + vhost-net (via a small libvirt-provider change); production dpservice uses SR-IOV VFs, a different attach model (see [DPDK vs eBPF/XDP](#dpdk-vs-ebpfxdp--a-balanced-comparison) and [DPUs / SmartNICs](#dpus--smartnics--how-both-interact)).

---

## DPDK vs eBPF/XDP — a balanced comparison

> The honest one-liner: **DPDK wins on raw throughput and mature HW offload; eBPF wins on operational simplicity and kernel integration. They are different points on a curve, not better/worse absolutely.**

### At a glance

| Dimension | DPDK (dpservice) | eBPF (XDP + tc-BPF) |
|---|---|---|
| Peak throughput / latency | **Highest**, deterministic under pinning | Lower (kernel skb path on the endpoint); XDP fast on the uplink |
| HW offload | **Mature** via `rte_flow` (NIC/eswitch) | Via **tc-flower → switchdev** (same silicon path, *not yet implemented here*); native-XDP HW offload is dead |
| Hugepages / CPU pinning | Required | **None** |
| NIC binding / drivers | PMD + driver binding per NIC | Standard kernel drivers |
| Observability / tooling | Opaque to `tcpdump`/`tc`/conntrack | **Native**: `tcpdump`, `tc`, `bpftool`, `ss`, conntrack, drop monitor |
| Encryption (IPsec/WireGuard) | Re-implement / out-of-band | **Composes** with XFRM/WireGuard (kernel skb path) |
| Version coupling | DPDK release cadence | Kernel + libbpf/aya; programs are portable (CO-RE) |
| Memory safety (control plane) | C | **Rust** |
| Maturity | **Production** | PoC (conformance parity) |
| Rollout granularity | Process-level | Per-device program attach; hybrid/opt-in trivial |

### Performance — the honest version

DPDK's poll-mode + userspace bypass is built for line-rate small-packet workloads and is hard to beat on pps. The eBPF design puts the **uplink** on XDP (fast, pre-`skb`) but the **guest edge** on tc-BPF (post-`skb`), which carries `skb` allocation and (for bulk traffic) GSO super-frame handling. For control-plane-heavy and moderate-throughput overlay workloads this is typically fine; for sustained line-rate small-packet forwarding, DPDK is expected to lead. **We have not benchmarked this** — a head-to-head on representative traffic is the first thing any serious evaluation needs.

### Hardware offload — clearing up a common confusion

There are four distinct things often lumped as "offload":
1. **Native/driver XDP** — eBPF on the **host CPU** in the driver poll loop. Broadly supported (mlx5/ice/i40e/virtio). Fast *software*, **not** silicon offload.
2. **XDP *hardware* offload** (`XDP_FLAGS_HW_MODE`) — eBPF **on the NIC**. Netronome-NFP only, **EOL/dead**. Not a path.
3. **AF_XDP zero-copy** — userspace poll over a netdev's rings; host CPU runs the logic. Broadly supported.
4. **tc-flower / `rte_flow` offload** — match/action rules pushed to the **NIC eswitch** (switchdev). **This is the real silicon-offload path**, broadly supported (ConnectX, Intel, Broadcom), and is the basis of OVS HW-offload.

So: dpservice's `rte_flow` and a future eBPF dataplane's `tc-flower`/`ct`-offload target **the same hardware mechanism** (the NIC/DPU eswitch). The difference is the *control-plane language*, not the silicon. The eBPF PoC does **not** implement tc-flower offload yet — it is an architectural fit, not a delivered feature.

### Operational differences (where eBPF is strongest)

- **No hugepages / no isolated cores** → fits commodity and small nodes; no capacity carve-out.
- **Dynamic, pool-free port lifecycle (software ports)** → taps/veths are created on demand via netlink + `bpf_link`, with no pre-provisioned VF pool, hugepage reservation, or `vfio-pci` rebind; the saving is in day-0 host provisioning. *Caveat:* dpservice's gRPC port lifecycle is already dynamic — and under SR-IOV VF passthrough both stacks hit the same boot-time `sriov_numvfs` ceiling, so the on-the-fly property belongs to *software* ports (and to **SFs**, see [The shared truth](#the-shared-truth)), not to eBPF-vs-DPDK per se.
- **Standard tooling** → `tcpdump` on the tap, `tc filter`/`bpftool` to inspect attachments, conntrack tooling, drop-monitor — the things operators already know.
- **Kernel composition** → routing, XFRM, WireGuard, netfilter coexist; encryption is a kernel feature, not a re-implementation.
- **Contributor on-ramp** → the patterns mirror Cilium/Calico; Rust + aya is approachable; the pure core is host-unit-testable without a NIC.

---

## DPUs / SmartNICs — how both interact

This is where the comparison gets most interesting, because IronCore's production model already assumes NIC/DPU offload.

### The shared truth

On a modern **switchdev** SmartNIC/DPU (NVIDIA BlueField, Intel IPU, etc.), the offloadable dataplane is a **match/action eswitch** programmed via **VF or SF (subfunction) representors**. *Both* DPDK (`rte_flow`) and the kernel (`tc-flower` + `ct` offload) program that same eswitch. So at the offload layer, DPDK and eBPF **converge on the same hardware**; the divergence is what runs the *slow path / control plane*.

**Subfunctions (SFs)** are worth naming here as a port model: a lightweight way to carve one PCIe device into many independent functions over the auxiliary bus (mlx5), reaching far higher density than SR-IOV VFs (thousands vs the low-hundreds VF ceiling) — relevant for high-pod-density hosts. Each SF gets its own eswitch representor, so `rte_flow` and `tc-flower` program SFs **identically to VFs**; this is a shared capability of both dataplanes, not specific to either.

### DPDK + DPU

- dpservice (or its logic) runs as a DPDK app — either on the **host** binding VFs, or **on the DPU's ARM cores**, with `rte_flow` offloading flows to the DPU ASIC. This is the canonical "BlueField + DPDK" story and is mature.
- Strength: peak performance and a well-trodden vendor path.

### eBPF/XDP + DPU

- **tc-flower offload to the DPU eswitch** is the kernel-native equivalent of `rte_flow` — the same silicon, driven from the Linux tc/netfilter stack. This is what OVS-offload and Cilium's hardware integrations use.
- **Run the eBPF dataplane on the DPU's ARM Linux.** Because it is kernel-native, the *entire* tc/XDP dataplane can run on the DPU's own Linux, offloading the x86 host — a clean "dataplane on the DPU" model without a userspace-bypass runtime. Cilium has been moving in this direction.
- **AF_XDP on a VF** is the userspace-poll option if peak performance on the DPU host is needed without full DPDK.
- **VF attach changes our dataplane's role.** With a VF *passed through to the guest*, the host sees no per-packet traffic — only the **VF representor** (the eswitch miss path). Our eBPF dataplane then runs as an **eswitch control plane**: handle the first packet of a flow on the representor, decide policy/overlay/conntrack, install a `tc-flower` + `ct` offload rule, and let silicon switch the rest. That is architecturally the **same role dpservice plays in its offloaded mode** — only the offload API differs (`tc-flower`/`ct` vs `rte_flow`). With a *host-owned* VF (guest reached via tap/veth/vDPA) the software datapath stays, but the guest edge remains tc (the VF only replaces the PF as the uplink — see [Why the guest edge must be tc](#why-the-guest-edge-must-be-tc-not-native-xdp-a-key-finding)).
- **SF + vDPA is the cleanest forward target.** A subfunction (see [The shared truth](#the-shared-truth)) backing a **vDPA** (virtio Data Path Acceleration) device presents a *standard* **virtio-net** interface to the guest — in-tree driver, **live-migration-friendly, no vendor-driver lock-in** (unlike VF passthrough) — while the data path is offloaded to the DPU eswitch and the control path programs flows on the SF representor. That stacks four wins at once: **dynamic/dense/pool-free ports + hardware offload + virtio guest compatibility + a kernel-native (tc-flower) control plane** — the eBPF dataplane's natural endgame. Guardrails: it is **mlx5/vendor-led and newer** (portability is a bet, not a given); **vDPA is equally available to a DPDK control plane** (the differentiator is *which* control plane programs the eswitch, not vDPA itself); and the PoC has **validated none of it**.
- Caveat: **native XDP HW-offload to the DPU is not a path** (NFP-only). The DPU offload story for eBPF is **tc-flower → eswitch**, not "XDP on the DPU ASIC."

### Why this matters to the decision

If the long-term target is **flows offloaded to a DPU eswitch**, then the host-side dataplane is increasingly a **control plane + slow path**. At that point the question shifts from "DPDK vs eBPF throughput" to "which control plane is cheaper to operate and evolve" — and a kernel-native, Rust, tc-flower-programming control plane is a credible contender precisely because the heavy lifting is in silicon either way. This is the strongest strategic argument for the eBPF direction, and also the one most in need of validation on real DPU hardware (which the PoC has **not** done).

---

## Future improvements / roadmap

Roughly in order of value-for-effort:

1. **Benchmark vs dpservice** on representative traffic (pps, latency, CPU) — the decision input that matters most.
2. **tc-flower / conntrack offload** to switchdev NICs/DPUs — turn "offload-ready" into "offloaded," directly comparable to `rte_flow`.
3. **AF_XDP fast-path** option for the uplink/VF where peak host throughput is needed without DPDK.
4. **Encryption** — IPsec (XFRM, incl. inline-crypto offload) and/or WireGuard for the overlay, leveraging the kernel path.
5. **Scale + soak** — many interfaces/flows, map sizing, multi-host overlay, HA/restart behavior under churn.
6. **DPU validation** — run the dataplane on a BlueField/IPU ARM Linux with tc-flower offload to the eswitch.
7. **Native SR-IOV VF attach** (instead of tap+vhost) to match dpservice's production port model. The *uplink* VF drops straight into the native-XDP half (real wire frames, no vhost/GSO issue). The *guest* side depends on the model: a VF **passed through to the guest** removes the host datapath entirely — eBPF then runs only as the eswitch slow-path/control plane (see [eBPF/XDP + DPU](#ebpfxdp--dpu)) — whereas a **host-owned** VF keeps the software datapath but leaves the guest edge on tc (see [Why the guest edge must be tc](#why-the-guest-edge-must-be-tc-not-native-xdp-a-key-finding)). So VFs remove the vhost/GSO consideration only in the passthrough case, and only because they remove the host datapath there — not because native XDP becomes viable at the guest edge.
8. **SF + vDPA port model** — evaluate **subfunctions backing vDPA virtio devices** as the production attach: virtio guests (migration-friendly, no driver lock-in) + dynamic/dense/pool-free ports + eswitch offload + a kernel-native control plane (see [eBPF/XDP + DPU](#ebpfxdp--dpu)). The most promising long-term target, but vendor-dependent (mlx5-led) and entirely unvalidated — worth a focused spike, not a commitment.
9. **Production hardening** — observability/metrics parity, conformance in CI against the eBPF path, packaging.

---

## Is it worth doing? (evaluation)

A balanced read:

**Arguments for pursuing (at least as an experimental/alternative dataplane):**
- It already exists and **passes conformance 93/93** behind the same gRPC contract — the integration risk is largely retired.
- The **operational** wins (no hugepages/pinning, native tooling, kernel composition, Rust) are real and address genuine friction for a class of deployments.
- It is **legible to the Cilium/Calico ecosystem**, lowering the contributor and operator on-ramp.
- On a **DPU-offload future**, the host dataplane is mostly control plane + slow path, where kernel-native is competitive *and* the offload silicon is shared with DPDK.
- Adoption can be **gradual and opt-in** (per-deployment), not a forced migration.

**Arguments for caution (or for keeping DPDK primary):**
- **No performance data yet.** If peak pps matters for the target workloads, DPDK may simply be required, and the eBPF path is a non-starter for those.
- **dpservice is production-proven**; the eBPF path is a PoC with real maturity, scale, and HA unknowns.
- **HW offload is unimplemented** in the PoC; until tc-flower offload lands and is measured, the offload comparison is theoretical.
- **Two dataplanes is a maintenance cost** — CI, conformance, expertise, docs — that must be justified by adoption.
- The team's **DPDK expertise** is an asset; an eBPF dataplane asks for a partially different skill set.

**A possible framing for the decision (not a foregone conclusion):**
- Treat this as a **complementary, opt-in dataplane** for profiles where operational simplicity outweighs peak throughput (edge, smaller/commodity nodes, dev/test, environments standardized on eBPF), **behind the same gRPC contract**, gated by a flag.
- **Gate further investment on a benchmark** and a tc-flower-offload spike. If the numbers and the offload story hold up for a real deployment profile, graduate it from experimental; if not, keep it as a research/portability artifact and stay on DPDK.

This proposal does **not** recommend replacing dpservice. It recommends deciding, with data, whether a kernel-native option earns a place alongside it.

---

## Open questions for maintainers & contributors

1. **Target workloads:** what pps/latency/CPU envelope must the dataplane hit for IronCore's primary deployments? (Defines whether eBPF is even in scope.)
2. **DPU trajectory:** how central is DPU/eswitch offload to the roadmap? (If central, the host-dataplane-as-control-plane argument strengthens the eBPF case.)
3. **Port model (largely answered — confirmation wanted):** the `DPDKironcore` contract shows the production model is **SR-IOV VF passthrough**: `CreateInterface` "creates and configures a VF (hypervisor case)" and returns a `VirtualFunction` carrying the VF's **PCIe BDF + Linux netdev name**, which metalnet/the VMM then VFIO-passes into the guest; `DeleteInterface` "releases the VF" for reuse. So in steady state dpservice is the eswitch **slow-path + `rte_flow` offload** control plane, not a per-packet bump-in-the-wire. dpservice *also* supports a **vhost-user** software backend for VMs (no passthrough). The mapping for an eBPF dataplane: VF-passthrough → **tc-flower/`ct` on the VF representor** (eBPF as eswitch slow-path/control plane, see [eBPF/XDP + DPU](#ebpfxdp--dpu)); vhost-user → exactly this PoC's **tap + vhost-net + tc** path (see [Why the guest edge must be tc](#why-the-guest-edge-must-be-tc-not-native-xdp-a-key-finding)). *Open part:* is the eBPF "slow-path/control-plane on the representor" role acceptable as the production target, and is the vhost-user/tap software path in scope as a supported fallback (or dev-only)?
4. **Two-dataplane appetite:** is a second, opt-in dataplane behind the same gRPC contract acceptable, or is single-dataplane focus preferred?
5. **Encryption requirements:** is overlay encryption (IPsec/WireGuard) on the roadmap? (Strongly favors the kernel path.)
6. **What evidence would change minds?** A specific benchmark target + a tc-flower-offload PoC would make this concrete — is that worth funding?

---

## Alternatives considered

- **AF_XDP instead of XDP+tc:** userspace-poll over a netdev; near-DPDK performance while staying kernel-interoperable (OVS-AF_XDP, VPP). Viable, but reintroduces a userspace runtime and doesn't get the "just a program on a device" simplicity. Could be a fast-path *option* (see the AF_XDP fast-path item under [Future improvements / roadmap](#future-improvements--roadmap)).
- **Native XDP everywhere (incl. guest edge):** not possible on vhost-backed taps (see [Why the guest edge must be tc](#why-the-guest-edge-must-be-tc-not-native-xdp-a-key-finding)); would only work with SR-IOV VFs or by disabling guest offloads. tc-BPF on the endpoint is the correct, supported choice.
- **Stay DPDK-only:** the status quo; lowest additional maintenance, highest throughput, weakest operational/ecosystem story. Entirely defensible depending on the answers to the [open questions](#open-questions-for-maintainers--contributors).

---

## Appendix A — PoC repositories & artifacts

- **`xdp-dp`** — the eBPF dataplane (Rust + aya). tc-BPF guest edge is the default (`XDP_DP_GUEST_TC=0` for legacy XDP). Conformance suite runs against both paths (93/93).
- **`libvirt-provider-tap`** — small fork: a plain-tap (`type='ethernet'`) NIC binding with vhost-net, so a host eBPF dataplane can sit on the guest tap.
- **`ironcore-in-a-box-xdp`** — fork wiring the eBPF dpservice + plain-tap provider into the kind-based lab; deploys the published fork images by default.

## Appendix B — Glossary

- **XDP** — eXpress Data Path; eBPF hook at/near the driver RX, pre-`skb`.
- **tc-BPF / clsact / tcx** — eBPF on the tc ingress/egress hook, post-`skb`.
- **`rte_flow`** — DPDK's NIC flow-offload API.
- **switchdev / eswitch** — the NIC/DPU's in-silicon match/action switch, programmed via VF/SF representors.
- **VF / SF** — SR-IOV Virtual Function vs Subfunction (mlx5 auxiliary-bus); two ways to partition a device into representor-backed ports, SFs reaching much higher density.
- **vhost-net / vhost-user** — kernel vs userspace virtio datapaths for VMs.
- **DPU/IPU** — data/infrastructure processing unit; a SmartNIC with on-board ARM CPUs + a programmable eswitch.
