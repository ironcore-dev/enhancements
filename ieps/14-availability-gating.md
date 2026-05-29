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
    - [ReadinessGateController](#readinessgatecontroller)
    - [ServerReadinessRule CRD](#serverreadinessrule-crd)
    - [Enforcement modes](#enforcement-modes)
    - [Lifecycle examples](#lifecycle-examples)
    - [Observability](#observability)
- [Dependencies](#dependencies)
- [Related](#related)

## Summary

Introduce a generic availability gating mechanism that prevents servers from being claimed until all declared readiness gates pass. A `ServerReadinessRule` declares which servers to target, which condition to monitor, and which taint to manage. The `ReadinessGateController` inside metal-operator watches rules and `Server` conditions and manages taint add/remove. Any entity with the appropriate permissions can write conditions on `Server` objects — they need not be aware that a rule exists.

## Motivation

Servers entering the fleet may fail pre-availability checks (e.g. physical wiring, unclean disks). Without a gating mechanism, a server can reach `Available` state and get claimed before these checks pass, leading to issues at workload runtime.

The driving use cases are network topology verification (an external controller compares discovered interface data against an authoritative source) and disk sanitization (prior workload data must be wiped before the server is re-claimable). Other gate types follow the same pattern without requiring changes to metal-operator.

### Goals

- Block servers from being claimed until all declared readiness gates pass
- Support multiple concurrent gate types owned by independent controllers
- Work for both managed and auto-discovered servers
- Be transparent to the workload scheduling layer
- Support re-gating when a server returns from maintenance

### Non-Goals

- Defining the logic run by individual external controllers
- Replacing existing maintenance workflows
- Automatic timeout or force-removal of taints for stuck gates — operators should alert on long-gated servers via the provided metrics
- Eviction of already-reserved servers based on gate state

## Proposal

### Overview

The taints of `Server` objects can be manipulated via `ServerReadinessRule` objects. These rules monitor the `.status.conditions` of a `Server` for a specific condition type and status.

If the specific condition type is not present in the expected status, a taint specified in the rule is applied to the `Server`.

Once the condition is present in the expected status, the taint is removed.

This makes a `Server` unclaimable to `ServerClaim`s that don't have the required tolerations.

### ReadinessGateController

The `ReadinessGateController` runs inside metal-operator and:

- Watches all `ServerReadinessRule` objects and all `Server` objects
- At each trigger point, evaluates which rules match a server and applies the corresponding taints
- Watches `Server.status.conditions` and removes a taint when its condition is satisfied
- Manages rule lifecycle: adds a finalizer at rule creation, removes orphaned taints on rule deletion or selector change

Any entity with the appropriate permissions can write conditions on `Server.status.conditions`. They do not need to apply or remove taints and need not be aware of any `ServerReadinessRule` that references their condition type.

### ServerReadinessRule CRD

A `ServerReadinessRule` is a cluster-scoped resource that declares:

- Which servers to target (`spec.serverSelector`)
- Which condition to monitor (`spec.condition.type`)
- Which taint to manage (`spec.taint`)
- When to enforce the rule (`spec.enforcement`)

```yaml
apiVersion: metal.ironcore.dev/v1alpha1
kind: ServerReadinessRule
metadata:
  name: disk-sanitization
  finalizers:
    - metal.ironcore.dev/serverreadinessgc
spec:
  serverSelector:
    matchLabels:
      example.com/hardware-type: server-gen2
  enforcement:
    mode: bootstrap
    triggers:
      - creation
      - maintenance
  condition:
    type: DiskSanitized
  taint:
    key: metal.ironcore.dev/disk-not-ready
    effect: NoClaim
---
apiVersion: metal.ironcore.dev/v1alpha1
kind: ServerReadinessRule
metadata:
  name: network-topology
  finalizers:
    - metal.ironcore.dev/serverreadinessgc
spec:
  serverSelector:
    matchLabels:
      example.com/hardware-type: server-gen2
  enforcement:
    mode: bootstrap
    triggers:
      - creation
      - maintenance
  condition:
    type: NetworkTopologyVerified
  taint:
    key: metal.ironcore.dev/network-not-ready
    effect: NoClaim
---
apiVersion: metal.ironcore.dev/v1alpha1
kind: ServerReadinessRule
metadata:
  name: hardware-inventory
  finalizers:
    - metal.ironcore.dev/serverreadinessgc
spec:
  serverSelector:
    matchLabels:
      example.com/hardware-type: server-gen2
  enforcement:
    mode: bootstrap
    triggers:
      - creation
  condition:
    type: HardwareInventoryVerified
  taint:
    key: metal.ironcore.dev/hardware-not-verified
    effect: NoClaim
```

**Taint key uniqueness** is enforced by a validating admission webhook. On create and update of a `ServerReadinessRule`, the webhook rejects the request if any other rule declares the same `spec.taint.key`. Each taint key must be owned by exactly one rule.

**Reserved condition types** are enforced by the same webhook. `ServerReadinessRule` objects whose `spec.condition.type` uses the `metal.ironcore.dev/` prefix are rejected — that namespace is reserved for metal-operator internal conditions.

**Rule lifecycle managed by ReadinessGateController:**

- At creation, adds `metal.ironcore.dev/serverreadinessgc` as a finalizer.
- On deletion, removes the rule's taint from all servers that carry it, then releases the finalizer.
- On `spec.serverSelector` change, removes the rule's taint from servers that no longer match the updated selector.

### Enforcement modes

#### `bootstrap`

The rule is enforced at specific trigger points only. At each listed trigger, the `ReadinessGateController` applies the taint to all matching servers. Once the condition passes and the taint is removed, the controller records completion via annotation and stops enforcing that rule for that server until the next trigger fires.

```yaml
metadata:
  annotations:
    metal.ironcore.dev/bootstrap-completed-disk-sanitization: "true"
```

If the condition later flips to `False` at runtime, nothing happens — the taint is not re-applied until the next trigger point.

Supported triggers:

| Trigger | When it fires |
|---|---|
| `creation` | Server enters Discovery for the first time |
| `maintenance` | Server exits Maintenance state |

Re-gating on server release (`Reserved` → `Available`) is out of scope for this proposal. It will be addressed by a dedicated server reclaim controller in a follow-up proposal.

#### `continuous`

The rule is always enforced regardless of trigger. If the condition is not satisfied at any point, the `ReadinessGateController` applies the taint immediately. When the condition recovers, the taint is removed. Suitable for checks that must hold throughout the server's lifetime.

### Lifecycle examples

#### Server created

All three rules apply at `creation`. `ReadinessGateController` applies all three taints immediately.

```yaml
# Server at creation
metadata:
  name: compute-01
  labels:
    example.com/hardware-type: server-gen2
spec:
  taints:
    - key: metal.ironcore.dev/disk-not-ready          # applied by ReadinessGateController
      effect: NoClaim
    - key: metal.ironcore.dev/network-not-ready       # applied by ReadinessGateController
      effect: NoClaim
    - key: metal.ironcore.dev/hardware-not-verified   # applied by ReadinessGateController
      effect: NoClaim
status:
  conditions: []
```

Hardware inventory controller runs first and sets its condition `True`. `ReadinessGateController` removes the hardware taint and annotates bootstrap completion.

```yaml
metadata:
  annotations:
    metal.ironcore.dev/bootstrap-completed-hardware-inventory: "true"
spec:
  taints:
    - key: metal.ironcore.dev/disk-not-ready
      effect: NoClaim
    - key: metal.ironcore.dev/network-not-ready
      effect: NoClaim
status:
  conditions:
    - type: HardwareInventoryVerified
      status: "True"
      reason: Verified
      lastTransitionTime: "2026-05-29T10:03:00Z"
```

Disk sanitization controller sets its condition `True`. `ReadinessGateController` removes the disk taint.

```yaml
spec:
  taints:
    - key: metal.ironcore.dev/network-not-ready
      effect: NoClaim
status:
  conditions:
    - type: HardwareInventoryVerified
      status: "True"
      reason: Verified
      lastTransitionTime: "2026-05-29T10:03:00Z"
    - type: DiskSanitized
      status: "True"
      reason: Sanitized
      lastTransitionTime: "2026-05-29T10:05:00Z"
```

Network topology controller sets its condition `True`. `ReadinessGateController` removes the last taint. Server is claimable.

```yaml
metadata:
  annotations:
    metal.ironcore.dev/bootstrap-completed-hardware-inventory: "true"
    metal.ironcore.dev/bootstrap-completed-disk-sanitization: "true"
    metal.ironcore.dev/bootstrap-completed-network-topology: "true"
spec:
  taints: []
status:
  conditions:
    - type: HardwareInventoryVerified
      status: "True"
      reason: Verified
      lastTransitionTime: "2026-05-29T10:03:00Z"
    - type: DiskSanitized
      status: "True"
      reason: Sanitized
      lastTransitionTime: "2026-05-29T10:05:00Z"
    - type: NetworkTopologyVerified
      status: "True"
      reason: TopologyMatch
      lastTransitionTime: "2026-05-29T10:12:00Z"
```

#### Server transitions from `Reserved` to `Available`

Re-gating on server release is handled by a dedicated server reclaim controller, which will be described in a follow-up proposal.

### Observability

Metal-operator exposes a per-server metric — one time series per server/taint combination, value `1` while active:

```
metaloperator_server_noclaim_taint{server="compute-01", taint="metal.ironcore.dev/disk-not-ready"} 1
```

Additional metrics may be added as the implementation matures.

## Dependencies

| Responsibility | Owner |
|---|---|
| Taint add/remove | ReadinessGateController (metal-operator) |
| Rule lifecycle (finalizer, taint GC) | ReadinessGateController (metal-operator) |
| Condition reporting | Any entity with write access to `Server.status.conditions` |

## Related

- ironcore-dev/enhancements#11 — original issue
- ironcore-dev/metal-operator#878 — Server taints
