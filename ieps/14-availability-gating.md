---
title: Availability Gating for IronCore Servers

iep-number: 0014

creation-date: 2026-04-27

status: implementable

authors:

- "@xkonni"

reviewers:

- "@ironcore-dev/metal-operator-maintainers"

---

# IEP-0014: Availability Gating for IronCore Servers

## Table of Contents

- [Summary](#summary)
- [Motivation](#motivation)
    - [Goals](#goals)
    - [Non-Goals](#non-goals)
- [Proposal](#proposal)
    - [Overview](#overview)
    - [Bootstrap taint](#bootstrap-taint)
    - [External gate controllers](#external-gate-controllers)
    - [ServerReadinessRule CRD](#serverreadinesssrule-crd)
    - [ServerReadinessGate CRD](#serverreadinessgate-crd)
    - [Lifecycle example](#lifecycle-example)
    - [Re-gating](#re-gating)
    - [Observability](#observability)
- [Dependencies](#dependencies)
- [Related](#related)

## Summary

Introduce a generic availability gating mechanism that prevents servers from being claimed until all declared readiness gates pass. The design follows the upstream [Kubernetes Node Readiness Controller](https://kubernetes.io/blog/2026/02/03/introducing-node-readiness-controller/) pattern: metal-operator sets a bootstrap taint at server creation, external gate controllers independently manage their own gate-specific taints and report conditions on the `Server` object, and metal-operator removes the bootstrap taint once all gate taints are cleared.

## Motivation

Servers entering the fleet may fail pre-availability checks (e.g. physical wiring, unclean disks). Without a gating mechanism, a server can reach `Available` state and get claimed before these checks pass, leading to issues at workload runtime.

The driving use case is network topology verification: an external controller compares discovered interface data against an authoritative source and only clears its gate condition once the topology matches. Other gate types follow the same pattern without requiring changes to metal-operator.

### Goals

- Block servers from being claimed until all declared readiness gates pass
- Support multiple concurrent gate types owned by independent controllers
- Work for both managed and auto-discovered servers
- Be transparent to the workload scheduling layer
- Support re-gating when a server returns from maintenance

### Non-Goals

- Defining the logic run by individual gate controllers
- Replacing existing maintenance workflows
- Automatic timeout or force-removal of taints for stuck gates — operators should alert on long-gated servers via the provided metrics
- Eviction of already-reserved servers based on gate state

## Proposal

### Overview

Gating is enforced via `NoBind` taints on the `Server` object. A server can only be claimed when all `NoBind` taints are gone. There are two categories of taint:

- **Bootstrap taint** (`metal.ironcore.dev/not-ready`) — set by metal-operator at trigger points, removed by metal-operator once no other `NoBind` taints remain
- **Gate taints** (e.g. `metal.ironcore.dev/network-not-ready`) — set and removed by external gate controllers, one taint key per gate type

Each controller has exclusive ownership of its taint key. They never interact with each other's taints.

### Bootstrap taint

Metal-operator gains a `--availability-gating` flag that accepts a comma-separated list of trigger points:

```
--availability-gating=creation,maintenance,release
```

| Value | Behavior |
|---|---|
| `creation` | Set `metal.ironcore.dev/not-ready: NoBind` at `Initial` → `Discovery` |
| `maintenance` | Re-set the taint on `Maintenance` exit |
| `release` | Re-set the taint when a server transitions from `Reserved` back to `Available` |

The default is empty (gating disabled). Existing deployments are unaffected unless the flag is explicitly set.

The bootstrap taint carries the trigger reason as its `value` field:

```yaml
taints:
  - key: metal.ironcore.dev/not-ready
    value: "creation"   # or: "maintenance", "release"
    effect: NoBind
```

This allows external gate controllers to inspect the taint value and decide whether to act. A disk cleaning controller interested only in post-release cleaning watches for `value: "release"` and ignores other triggers. A network controller watching for `value: "creation"` and `value: "maintenance"` ignores release events.

Metal-operator removes the bootstrap taint once no other `NoBind` taints remain on the server. The expected flow is that external gate controllers apply their gate taints early in Discovery (which takes several minutes), so by the time Discovery completes the gate taints are in place. If no gate controllers target the server, the bootstrap taint is removed at the end of Discovery and the server proceeds normally.

**`NoBind` taints do not affect `Reserved` servers.** The taint is only evaluated at claim time — it prevents a new `ServerClaim` from binding but does not evict an existing reservation.

### External gate controllers

External gate controllers are independent components, one per gate type. Each gate controller:

- Watches `Server` objects via a label selector (servers are labeled by external systems to opt into specific gates)
- Applies its own gate taint to matching servers
- Runs its check and writes its condition to `Server.status.conditions`
- Removes its own gate taint when its condition passes
- Creates and updates a `ServerReadinessGate` object per server for observability

Gate controllers do not interact with the bootstrap taint or with each other.

Servers are brought into scope by adding the matching selector label. External systems are responsible for labeling servers appropriately — for example, a provisioning tool reading from an inventory source can set labels based on which checks are required for a given server.

### ServerReadinessRule CRD

A `ServerReadinessRule` is a cluster-scoped resource, created once by whoever deploys a gate controller. It declares which servers the gate controller targets, which condition it monitors, and which taint it manages. It is read by the gate controller and never touched by metal-operator.

```yaml
apiVersion: metal.ironcore.dev/v1alpha1
kind: ServerReadinessRule
metadata:
  name: network-readiness
spec:
  serverSelector:
    matchLabels:
      metal.ironcore.dev/gate-network: "true"
  condition:
    type: NetworkTopologyVerified
  taint:
    key: metal.ironcore.dev/network-not-ready
    effect: NoBind
  triggers: [creation, maintenance]
---
apiVersion: metal.ironcore.dev/v1alpha1
kind: ServerReadinessRule
metadata:
  name: disk-readiness
spec:
  serverSelector:
    matchLabels:
      metal.ironcore.dev/gate-disk: "true"
  condition:
    type: DiskSanitized
  taint:
    key: metal.ironcore.dev/disk-not-ready
    effect: NoBind
  triggers: [release]
```

The `triggers` field declares which bootstrap taint values the gate controller responds to. When metal-operator sets the bootstrap taint, the gate controller checks whether the taint value matches its declared triggers before applying its gate taint and running its check.

### ServerReadinessGate CRD

A `ServerReadinessGate` is created and updated by an external gate controller — one object per server/rule combination. It provides a single queryable surface for operators to inspect gate state across the fleet. Metal-operator never reads or writes these objects.

```yaml
apiVersion: metal.ironcore.dev/v1alpha1
kind: ServerReadinessGate
metadata:
  name: compute-01-network
spec:
  serverRef:
    name: compute-01
  ruleRef:
    name: network-readiness
status:
  condition:
    type: NetworkTopologyVerified
    status: "False"
    reason: TopologyMismatch
  taint: metal.ironcore.dev/network-not-ready
```

```bash
kubectl get serverreadinessgates
NAME                   SERVER       RULE               CONDITION                STATUS   REASON
compute-01-network     compute-01   network-readiness  NetworkTopologyVerified  False    TopologyMismatch
compute-01-disk        compute-01   disk-readiness     DiskSanitized            True     Sanitized
```

### Lifecycle example

**Server created, gate controllers apply their taints during Discovery:**
```yaml
# Server
metadata:
  name: compute-01
  labels:
    metal.ironcore.dev/gate-network: "true"
    metal.ironcore.dev/gate-disk: "true"
spec:
  taints:
    - key: metal.ironcore.dev/not-ready          # set by metal-operator
      effect: NoBind
    - key: metal.ironcore.dev/network-not-ready  # set by network gate controller
      effect: NoBind
    - key: metal.ironcore.dev/disk-not-ready     # set by disk gate controller
      effect: NoBind
status:
  conditions:
    - type: NetworkTopologyVerified
      status: "False"
      reason: Pending
    - type: DiskSanitized
      status: "False"
      reason: Pending
```

**Disk gate passes — network still pending:**
```yaml
spec:
  taints:
    - key: metal.ironcore.dev/not-ready
      effect: NoBind
    - key: metal.ironcore.dev/network-not-ready
      effect: NoBind
status:
  conditions:
    - type: NetworkTopologyVerified
      status: "False"
      reason: TopologyMismatch
    - type: DiskSanitized
      status: "True"
      reason: Sanitized
```

**All gates pass — metal-operator removes bootstrap taint, server claimable:**
```yaml
spec:
  taints: []
status:
  conditions:
    - type: NetworkTopologyVerified
      status: "True"
      reason: TopologyMatch
    - type: DiskSanitized
      status: "True"
      reason: Sanitized
```

### Re-gating

At each trigger point metal-operator re-sets the bootstrap taint with the corresponding trigger value. Gate controllers that declare that trigger value in their `triggers` field re-apply their gate taint and re-evaluate from scratch. Gate controllers that do not declare that trigger value ignore the event and do not re-gate.

### Observability

Metal-operator exposes metrics for servers carrying `NoBind` taints.

Per-server metric — one time series per server/taint combination, value `1` while active:

```
metaloperator_server_nobind_taint{server="compute-01", namespace="default", taint="metal.ironcore.dev/not-ready"} 1
metaloperator_server_nobind_taint{server="compute-01", namespace="default", taint="metal.ironcore.dev/network-not-ready"} 1
```

Aggregate metric — count of servers per taint key:

```
metaloperator_server_nobind_taint_total{taint="metal.ironcore.dev/not-ready"} 5
metaloperator_server_nobind_taint_total{taint="metal.ironcore.dev/network-not-ready"} 3
```

Individual gate state is visible via `kubectl get serverreadinessgates`.

## Dependencies

- **metal-operator**: `--availability-gating` flag, bootstrap taint management
- **ironcore-dev/metal-operator#672**: Server taints

## Related

- ironcore-dev/enhancements#11 — original issue
- ironcore-dev/metal-operator#672 — Server taints
