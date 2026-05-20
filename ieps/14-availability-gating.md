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
    - [Enabling gating](#enabling-gating)
    - [Gate configuration](#gate-configuration)
    - [ServerReadinessGate CRD](#serverreadinessgate-crd)
    - [Controller responsibilities](#controller-responsibilities)
    - [Lifecycle examples](#lifecycle-examples)
    - [Re-gating](#re-gating)
    - [Observability](#observability)
- [Dependencies](#dependencies)
- [Related](#related)

## Summary

Introduce a generic availability gating mechanism that allows external controllers to block servers from being claimed until all declared gates pass. Gates are configured cluster-wide via a ConfigMap, enforced via a single `metal.ironcore.dev/not-ready: NoBind` taint on the `Server`, and tracked via a dedicated `ServerReadinessGate` CRD. Any external controller can own and reconcile its respective gate condition independently.

## Motivation

Servers entering the fleet may fail pre-availability checks (e.g. physical wiring, unclean disks). Without a gating mechanism, a server can reach `Available` state and get claimed before these checks pass, leading to issues at workload runtime.

The driving use case is network topology verification: an external controller compares discovered interface data against an authoritative source and only clears its gate condition once the topology matches. Other gate types (e.g. disk sanitization) follow the same pattern without requiring changes to metal-operator or other gate controllers.

### Goals

- Block servers from being claimed until all declared readiness gates pass
- Support multiple concurrent gate types owned by independent controllers
- Work for both managed and auto-discovered servers
- Be transparent to the workload scheduling layer
- Support re-gating when a server returns from maintenance

### Non-Goals

- Defining the logic run by individual gate controllers
- Replacing existing maintenance workflows
- Per-server or per-server-class gate configuration (cluster-wide only for now)
- Eviction of already-reserved servers based on gate state
- Automatic timeout or force-removal of the `not-ready` taint for stuck gates — operators should alert on long-gated servers via the provided metrics

## Proposal

### Enabling gating

Metal-operator gains a `--availability-gating` flag that accepts a comma-separated list of trigger points:

```
--availability-gating=creation,maintenance,release
```

| Value | Behavior |
|---|---|
| `creation` | Set `metal.ironcore.dev/not-ready: NoBind` taint and create `ServerReadinessGate` at `Initial` → `Discovery` |
| `maintenance` | Re-set the taint and re-create the `ServerReadinessGate` on `Maintenance` exit |
| `release` | Re-set the taint and re-create the `ServerReadinessGate` when a server transitions from `Reserved` back to `Available` |

The default is empty (gating disabled). Existing deployments are unaffected unless the flag is explicitly set.

**Trigger points are the only moments when the ConfigMap is evaluated.** The gate list is snapshotted into the `ServerReadinessGate` at trigger time and not updated mid-flight. A ConfigMap change takes effect at the next trigger — the next creation, maintenance exit, or reservation release, depending on which triggers are enabled. This means:

- A gate added to the ConfigMap while a server is in `Discovery` will not apply to that server until its next trigger.
- A gate added while a server is `Reserved` will be picked up at maintenance exit (if `maintenance` is enabled) or reservation release (if `release` is enabled).

### Gate configuration

The set of gates to apply is configured via a ConfigMap in the metal-operator namespace:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: metal-operator-availability-gating
data:
  gates: |
    - conditionType: NetworkTopologyVerified
      triggers: [creation, maintenance]
    - conditionType: DiskSanitized
      triggers: [release]
```

Metal-operator reads this ConfigMap at each trigger point and stamps only the gates whose `triggers` list includes the current trigger. Adding or removing a gate takes effect at the next trigger without restarting metal-operator.

### ServerReadinessGate CRD

Metal-operator creates a `ServerReadinessGate` object for each gated server at trigger time, populated from the ConfigMap snapshot. The `ServerReadinessGate` is owned by the `Server` object — deletion of the `Server` cascades to the gate object. On re-trigger, metal-operator deletes the existing `ServerReadinessGate` and creates a new one; gate controllers must detect the new object and re-evaluate their conditions from scratch. External gate controllers write their conditions to this object. Metal-operator removes the `not-ready` taint from the `Server` when all conditions are `True`.

A condition listed in `spec.gates` but absent from `status.conditions` is treated as blocking — metal-operator will not remove the `not-ready` taint until all declared conditions are explicitly present and `True`.

```yaml
apiVersion: metal.ironcore.dev/v1alpha1
kind: ServerReadinessGate
metadata:
  name: compute-01
  namespace: default
spec:
  serverRef:
    name: compute-01
  gates:
    - conditionType: NetworkTopologyVerified
    - conditionType: DiskSanitized
status:
  conditions:
    - type: NetworkTopologyVerified
      status: "False"
      reason: TopologyMismatch
    - type: DiskSanitized
      status: "True"
      reason: Sanitized
```

### Controller responsibilities

**Metal-operator:**
- Sets `metal.ironcore.dev/not-ready: NoBind` taint on the `Server` at each trigger point
- Creates the `ServerReadinessGate` object from the ConfigMap snapshot at each trigger point
- Watches `ServerReadinessGate` objects and removes the `not-ready` taint when all conditions are `True`
- Does not implement any gate logic itself

**External gate controllers** (one per gate type):
- Watch `ServerReadinessGate` objects independently
- Write their condition to `status.conditions` of the `ServerReadinessGate`
- Do not interact with the `Server` taint directly

New gate types are introduced by deploying a new gate controller and adding its condition type to the ConfigMap — no changes to metal-operator or existing gate controllers required.

### Lifecycle examples

**Server at creation — all gates pending:**
```yaml
# Server
spec:
  taints:
    - key: metal.ironcore.dev/not-ready
      effect: NoBind

# ServerReadinessGate
spec:
  serverRef:
    name: compute-01
  gates:
    - conditionType: NetworkTopologyVerified
    - conditionType: DiskSanitized
status:
  conditions:
    - type: NetworkTopologyVerified
      status: "False"
      reason: Pending
    - type: DiskSanitized
      status: "False"
      reason: Pending
```

**DiskSanitized gate passes — network still pending:**
```yaml
# Server
spec:
  taints:
    - key: metal.ironcore.dev/not-ready
      effect: NoBind

# ServerReadinessGate
status:
  conditions:
    - type: NetworkTopologyVerified
      status: "False"
      reason: TopologyMismatch
    - type: DiskSanitized
      status: "True"
      reason: Sanitized
```

**All gates pass — metal-operator removes `not-ready`, server can be claimed:**
```yaml
# Server
spec:
  taints: []

# ServerReadinessGate
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

At each trigger point metal-operator re-sets the `not-ready` taint and re-creates the `ServerReadinessGate` from the current ConfigMap snapshot. External gate controllers re-evaluate their conditions from scratch, and metal-operator removes the taint only once all gates pass again.

**`NoBind` taints do not affect `Reserved` servers.** The `not-ready` taint is only evaluated at claim time — it prevents a new `ServerClaim` from binding, but does not evict or invalidate an existing reservation. If a `Reserved` server is re-tainted, the taint takes effect when the reservation is released and a new claim attempt is made.

### Observability

Metal-operator exposes metrics for servers carrying the `not-ready` taint.

Per-server metric — one time series per server, value `1` while the taint is active:

```
metaloperator_server_nobind_taint{server="compute-01", namespace="default"} 1
```

Aggregate metric — total count of servers currently gated:

```
metaloperator_server_nobind_taint_total 5
```

Individual gate condition state is visible via `kubectl get serverreadinessgates`:

```
NAME          SERVER      NETWORKTOPOLOGYVERIFIED   DISKSANITIZED
compute-01   compute-01  False (TopologyMismatch)   True (Sanitized)
compute-02   compute-02  True (TopologyMatch)       False (Pending)
```

## Dependencies

- **metal-operator**: `--availability-gating` flag, ConfigMap watch, `ServerReadinessGate` CRD, taint management
- **ironcore-dev/metal-operator#672**: Server taints

## Related

- ironcore-dev/enhancements#11 — original issue
- ironcore-dev/metal-operator#672 — Server taints
