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
    - [Taint model](#taint-model)
    - [Readiness gates and conditions](#readiness-gates-and-conditions)
    - [Gating controller pattern](#gating-controller-pattern)
    - [Lifecycle examples](#lifecycle-examples)
    - [Re-gating](#re-gating)
- [Dependencies](#dependencies)
- [Related](#related)

## Summary

Introduce a generic availability gating mechanism that allows external controllers to block servers from being claimed until all declared gates pass. Gates are configured cluster-wide via a ConfigMap, stamped as `NoBind` taints on the `Server` at creation and maintenance exit, tracked via `status.conditions`, and enforced until a gating controller confirms all conditions are satisfied.

## Motivation

Servers entering the fleet may fail pre-availability checks (e.g. physical wiring, firmware compliance). Without a gating mechanism, a server can reach `Available` state and get claimed before these checks pass, leading to issues at workload runtime.

The driving use case is network topology verification: an external controller compares discovered interface data against an authoritative source and only clears its gate condition once the topology matches. Other gate types (e.g. firmware validation) follow the same pattern without requiring changes to metal-operator or other gate controllers.

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
- Re-gating triggered by external events other than maintenance exit — gate controllers may re-add their own taints independently, but this is not orchestrated by metal-operator

## Proposal

### Enabling gating

Metal-operator gains a `--availability-gating` flag that accepts a comma-separated list of trigger points:

```
--availability-gating=creation,maintenance
```

| Value | Behavior |
|---|---|
| `creation` | Apply gate taints unconditionally at `Initial` → `Discovery` |
| `maintenance` | Re-apply gate taints unconditionally on `Maintenance` exit |

The default is empty (gating disabled). Existing deployments are unaffected unless the flag is explicitly set.

### Gate configuration

The set of gates to apply is configured via a ConfigMap in the metal-operator namespace:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: metal-operator-availability-gating
data:
  gates: |
    - metal.ironcore.dev/gate-network
    - metal.ironcore.dev/gate-firmware
```

Metal-operator watches this ConfigMap at runtime. At each trigger point it reads the current gate list and stamps the corresponding taints and `readinessGates` onto the Server. Adding or removing a gate from the ConfigMap takes effect for the next trigger (next server creation or maintenance exit) without restarting metal-operator.

### Taint model

Two categories of `NoBind` taints are used:

- `metal.ironcore.dev/uninitialized` — set by metal-operator at each trigger point, removed by the gating controller once all gate conditions are satisfied. This is the master lock that prevents claiming.
- `metal.ironcore.dev/gate-<type>` — one per gate, set by metal-operator at each trigger point, removed by the respective gate controller when its check passes.

Each controller has exclusive ownership of its taint key, eliminating write races.

### Readiness gates and conditions

Metal-operator writes `spec.readinessGates` from the ConfigMap at each trigger point, mirroring the gate list.

Each external gate controller owns a condition in `status.conditions` corresponding to its gate type. It creates the condition when it first reconciles the Server and updates it as its check progresses.

Metal-operator removes the `uninitialized` taint only when all conditions listed in `readinessGates` have `status: "True"`.

### Controller responsibilities

**metal-operator:**
- Sets `uninitialized` + all `gate-*` taints at each trigger point (from the ConfigMap gate list)
- Writes `readinessGates` on the Server from the ConfigMap gate list
- Watches `Server` objects and removes the `uninitialized` taint when all `readinessGates` conditions are `True`
- Does not implement any gate logic itself

**External gate controllers** (one per gate type):
- Watch `Server` objects independently
- Remove their own `gate-<type>` taint when their check passes
- Update their corresponding condition in `status.conditions`
- Do not interact with the `uninitialized` taint

New gate types are introduced by deploying a new gate controller and adding its key to the ConfigMap — no changes to metal-operator or existing gate controllers required.

### Lifecycle examples

**Server at creation — all gates pending:**
```yaml
spec:
  readinessGates:
    - conditionType: NetworkTopologyVerified
    - conditionType: FirmwareCompliant
  taints:
    - key: metal.ironcore.dev/uninitialized
      effect: NoBind
    - key: metal.ironcore.dev/gate-network
      effect: NoBind
    - key: metal.ironcore.dev/gate-firmware
      effect: NoBind
status:
  conditions:
    - type: NetworkTopologyVerified
      status: "False"
      reason: Pending
    - type: FirmwareCompliant
      status: "False"
      reason: Pending
```

**Firmware gate passes — network still pending:**
```yaml
spec:
  readinessGates:
    - conditionType: NetworkTopologyVerified
    - conditionType: FirmwareCompliant
  taints:
    - key: metal.ironcore.dev/uninitialized
      effect: NoBind
    - key: metal.ironcore.dev/gate-network
      effect: NoBind
status:
  conditions:
    - type: NetworkTopologyVerified
      status: "False"
      reason: TopologyMismatch
    - type: FirmwareCompliant
      status: "True"
      reason: VersionMatch
```

**All gates pass — metal-operator removes `uninitialized`, server can be claimed:**
```yaml
spec:
  readinessGates:
    - conditionType: NetworkTopologyVerified
    - conditionType: FirmwareCompliant
  taints: []
status:
  conditions:
    - type: NetworkTopologyVerified
      status: "True"
      reason: TopologyMatch
    - type: FirmwareCompliant
      status: "True"
      reason: VersionMatch
```

### Re-gating

When `maintenance` is included in `--availability-gating`, metal-operator re-applies all gate taints and resets `readinessGates` on `Maintenance` exit. External gate controllers re-evaluate their conditions from scratch, and metal-operator removes the `uninitialized` taint only once all gates pass again.

External gate controllers may also independently re-add their own `gate-<type>` taint at any time. Metal-operator honors any `NoBind` taint present on a Server regardless of who set it.

**`NoBind` taints do not affect `Reserved` servers.** A `NoBind` taint is only evaluated at claim time — it prevents a new `ServerClaim` from binding, but does not evict or invalidate an existing reservation. If a `Reserved` server is re-tainted, the taint takes effect when the reservation is released and a new claim attempt is made. Eviction of a live server is out of scope for this mechanism.

### Observability

Metal-operator exposes metrics for servers carrying `NoBind` taints.

Per-server metric — one time series per server/taint combination, value `1` while the taint is active:

```
metaloperator_server_nobind_taint{server="compute-01", namespace="default", taint="metal.ironcore.dev/uninitialized"} 1
metaloperator_server_nobind_taint{server="compute-01", namespace="default", taint="metal.ironcore.dev/gate-network"} 1
metaloperator_server_nobind_taint{server="compute-01", namespace="default", taint="metal.ironcore.dev/gate-firmware"} 1
```

Aggregate metric — count of servers per taint key:

```
metaloperator_server_nobind_taint_total{taint="metal.ironcore.dev/uninitialized"} 5
metaloperator_server_nobind_taint_total{taint="metal.ironcore.dev/gate-network"} 3
metaloperator_server_nobind_taint_total{taint="metal.ironcore.dev/gate-firmware"} 1
```

This allows operators to identify which specific servers are gated, which gate is the bottleneck, and track the overall fleet gating state.

## Dependencies

- **metal-operator**: `--availability-gating` flag, ConfigMap watch, taint + `readinessGates` stamping at trigger points
- **ironcore-dev/metal-operator#672**: Server taints

## Related

- ironcore-dev/enhancements#11 — original issue
- ironcore-dev/metal-operator#672 — Server taints
