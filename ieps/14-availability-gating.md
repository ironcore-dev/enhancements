---
title: Introducing ServerReadinessRule to ensure Server readiness

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
    - [ServerReadinessRule CRD](#serverreadinessrule-crd)
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

Gating is enforced via `NoClaim` taints on the `Server` object. A server can only be claimed when all `NoClaim` taints are gone. There are two categories of taint:

- **Bootstrap taint** (`metal.ironcore.dev/not-ready`) — set by metal-operator at trigger points, removed by metal-operator once no other `NoClaim` taints remain
- **Gate taints** (e.g. `metal.ironcore.dev/network-not-ready`) — set and removed by external gate controllers, one taint key per gate type

Each controller has exclusive ownership of its taint key. They never interact with each other's taints.

### Bootstrap taint

Metal-operator sets the bootstrap taint automatically — no feature flag is required. At each trigger point, metal-operator checks whether any `ServerReadinessRule` matches the server. If at least one does, the bootstrap taint is set. Servers with no matching rules are unaffected and proceed normally.

| Trigger | Behavior |
|---|---|
| `creation` | Set `metal.ironcore.dev/not-ready: NoClaim` at `Initial` → `Discovery` |
| `maintenance` | Re-set the taint on `Maintenance` exit |
| `release` | Re-set the taint when a server transitions from `Reserved` back to `Available` |

The bootstrap taint carries the trigger reason as its `value` field:

```yaml
taints:
  - key: metal.ironcore.dev/not-ready
    value: "creation"   # or: "maintenance", "release"
    effect: NoClaim
```

Gate controllers can inspect the taint value to fast-forward checks where appropriate. A disk sanitization controller seeing `value: "creation"` knows no prior workload ran on the server and can immediately mark its condition `True`. Seeing `value: "release"` it performs the full sanitization. This is a hint — gate controllers are not required to act differently per trigger value.

Metal-operator removes the bootstrap taint once no other `NoClaim` taints remain on the server. The expected flow is that external gate controllers apply their gate taints early in Discovery (which takes several minutes), so by the time Discovery completes the gate taints are in place. If no gate controllers target the server, the bootstrap taint is removed at the end of Discovery and the server proceeds normally.

**`NoClaim` taints do not affect `Reserved` servers.** The taint is only evaluated at claim time — it prevents a new `ServerClaim` from binding but does not evict an existing reservation. Taints added while a server is `Reserved` (e.g. hardware degradation detected during a workload's lifetime) are preserved and not cleaned up on reservation. When the server is released and transitions back to `Available`, the `release` trigger fires: metal-operator re-sets the bootstrap taint and all matching gate controllers re-evaluate. The server becomes claimable again only after all taints are cleared.

### External gate controllers

External gate controllers are independent components, one per gate type. Each gate controller:

- Watches `Server` objects via a label selector defined in the matching `ServerReadinessRule`
- Applies its own gate taint to matching servers
- Runs its check and writes its condition to `Server.status.conditions`
- Removes its own gate taint when its condition passes

Gate controllers do not interact with the bootstrap taint or with each other.

**Re-arm contract:** when a gate controller re-arms at a trigger point (i.e. it applies its gate taint because the bootstrap taint appeared on a matching server), it must reset its condition to `False` before running its check. This ensures a passing condition from a previous cycle does not carry over and cause the gate to be skipped.

Servers are brought into scope via the `spec.serverSelector` in the `ServerReadinessRule` — this is the gate controller's targeting configuration. Any labels already present on the server (hardware type, region, zone, etc.) can be used; no dedicated gate labels are required.

### ServerReadinessRule CRD

A `ServerReadinessRule` is a cluster-scoped resource, created once by whoever deploys a gate controller. It declares which servers the gate controller targets, which condition it monitors, and which taint it manages. Gate controllers read rules to determine which servers to target and which taint to manage. Metal-operator watches rules solely for lifecycle management: it adds a finalizer at creation and removes orphaned gate taints on deletion or selector change.

**Rule lifecycle managed by metal-operator:**

- At creation, metal-operator adds `metal.ironcore.dev/serverreadinessgc` as a finalizer.
- On deletion (`deletionTimestamp` set), metal-operator removes the rule's taint from all servers that carry it, then releases the finalizer. The gate controller being unavailable at deletion time does not block cleanup.
- On `spec.serverSelector` change, metal-operator removes the rule's taint from servers that no longer match the updated selector.

This gives a clean separation: gate controllers own gating logic; metal-operator owns rule lifecycle and taint GC.

```yaml
apiVersion: metal.ironcore.dev/v1alpha1
kind: ServerReadinessRule
metadata:
  name: network-readiness
  finalizers:
    - metal.ironcore.dev/serverreadinessgc
spec:
  serverSelector:
    matchLabels:
      example.com/hardware-type: server-gen2
      topology.kubernetes.io/region: region-a
  condition:
    type: NetworkTopologyVerified
    maxAge: 4h
  taint:
    key: metal.ironcore.dev/network-not-ready
    effect: NoClaim
---
apiVersion: metal.ironcore.dev/v1alpha1
kind: ServerReadinessRule
metadata:
  name: disk-readiness
  finalizers:
    - metal.ironcore.dev/serverreadinessgc
spec:
  serverSelector:
    matchLabels:
      example.com/hardware-type: server-gen2
  condition:
    type: DiskSanitized
  taint:
    key: metal.ironcore.dev/disk-not-ready
    effect: NoClaim
```

The optional `condition.maxAge` field marks a gate as recurring. Metal-operator monitors `lastTransitionTime` on the corresponding server condition and re-applies the gate taint if the condition has been `True` for longer than `maxAge`, signalling the gate controller to re-run. Gate controllers for recurring gates are expected to periodically re-evaluate and re-patch their condition to keep `lastTransitionTime` current. If `maxAge` is absent the gate is treated as one-shot: the condition persists until the gate controller explicitly changes it or a trigger point resets it.

**Taint key uniqueness** is enforced by a validating admission webhook. On create and update of a `ServerReadinessRule`, the webhook lists all existing rules and rejects the request if any other rule declares the same `spec.taint.key`. Each taint key must be owned by exactly one rule.

**Reserved condition types** are also enforced by the same webhook. Metal-operator defines a set of internal condition types (prefixed `metal.ironcore.dev/`). A `ServerReadinessRule` whose `spec.condition.type` matches any reserved internal type is rejected. External gate controllers must use their own unprefixed or distinctly prefixed condition types.

### Lifecycle example

**Server created, gate controllers apply their taints during Discovery:**
```yaml
# Server
metadata:
  name: compute-01
  labels:
    example.com/hardware-type: server-gen2
    topology.kubernetes.io/region: region-a
    topology.kubernetes.io/zone: region-a-1
spec:
  taints:
    - key: metal.ironcore.dev/not-ready          # set by metal-operator
      effect: NoClaim
    - key: metal.ironcore.dev/network-not-ready  # set by network gate controller
      effect: NoClaim
    - key: metal.ironcore.dev/disk-not-ready     # set by disk gate controller
      effect: NoClaim
status:
  conditions:
    - type: NetworkTopologyVerified
      status: "False"
      reason: Pending
      lastTransitionTime: "2026-04-27T10:00:00Z"
    - type: DiskSanitized
      status: "False"
      reason: Pending
      lastTransitionTime: "2026-04-27T10:00:00Z"
```

**Disk gate passes — network still pending:**
```yaml
spec:
  taints:
    - key: metal.ironcore.dev/not-ready
      effect: NoClaim
    - key: metal.ironcore.dev/network-not-ready
      effect: NoClaim
status:
  conditions:
    - type: NetworkTopologyVerified
      status: "False"
      reason: TopologyMismatch
      lastTransitionTime: "2026-04-27T10:01:00Z"
    - type: DiskSanitized
      status: "True"
      reason: Sanitized
      lastTransitionTime: "2026-04-27T10:05:00Z"
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
      lastTransitionTime: "2026-04-27T10:12:00Z"
    - type: DiskSanitized
      status: "True"
      reason: Sanitized
      lastTransitionTime: "2026-04-27T10:05:00Z"
```

### Re-gating

At each trigger point metal-operator re-sets the bootstrap taint with the corresponding trigger value. All gate controllers matching the server re-apply their gate taint and re-evaluate from scratch. Gate controllers may inspect the taint value to fast-forward checks where appropriate.

### Observability

Metal-operator exposes a per-server metric — one time series per server/taint combination, value `1` while active:

```
metaloperator_server_noclaim_taint{server="compute-01", namespace="default", taint="metal.ironcore.dev/not-ready"} 1
metaloperator_server_noclaim_taint{server="compute-01", namespace="default", taint="metal.ironcore.dev/network-not-ready"} 1
```

Aggregate counts can be derived via PromQL:

```promql
count by (taint) (metaloperator_server_noclaim_taint)
```

Gate controller conditions on `Server.status.conditions` include `lastTransitionTime`. Operators can alert on conditions that have remained `False` beyond a deployment-specific threshold (e.g. 2 hours) using a `kube_customresource_status_condition` query. A condition stuck `False` means the gate controller has not cleared it — either the check is genuinely failing or the gate controller has crashed. In either case the server correctly remains gated; the alert surfaces which servers need attention.

## Dependencies

| Responsibility | Owner |
|---|---|
| Bootstrap taint (set/remove) | metal-operator |
| Gating logic (apply/remove gate taint, report condition) | Gate controller |
| Rule lifecycle (finalizer, taint GC on deletion/selector change) | metal-operator |

- **metal-operator**: bootstrap taint management, `ServerReadinessRule` finalizer and taint GC
- **ironcore-dev/metal-operator#672**: Server taints

## Related

- ironcore-dev/enhancements#11 — original issue
- ironcore-dev/metal-operator#672 — Server taints
