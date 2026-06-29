---
title: MachinePool Eviction

iep-number: 21

creation-date: 2026-06-24

status: draft

authors:

- "@lukasfrank"

reviewers:

- "@main-reviewer-1"
- "@main-reviewer-2"

---

# IEP-21: MachinePool Eviction

## Table of Contents

- [Summary](#summary)
- [Motivation](#motivation)
    - [Goals](#goals)
    - [Non-Goals](#non-goals)
- [Proposal](#proposal)
    - [API Changes](#api-changes)
    - [Eviction Flow](#eviction-flow)
- [Alternatives](#alternatives)

## Summary

This IEP defines how `Machines` are evicted from a MachinePool.

We introduce a `NoExecute` taint effect. When set on a `MachinePool`, a taint eviction controller issues DELETE on all bound `Machine`s. 
The provider handles graceful VM shutdown via its existing finalizer before the object is fully removed.

## Motivation

Currently, when a bare-metal host backing a `MachinePool` needs maintenance, there is no way to signal this. The host gets shut down, VMs and workloads are killed, while the `Machine` objects in the API still appear as running.

There is no mechanism to gracefully shut down VMs in an ordered fashion before the host goes down.

### Goals

* Introduce `NoExecute` taint for the `MachinePool` that triggers deletion of all bound `Machine`s.

### Non-Goals

* Handle the unplanned failure case: when a pool becomes unhealthy (IEP-15 marks it `Ready=Unknown`), automatically evict machines after a configurable threshold.
* Drain orchestration (serial eviction, disruption budgets). Drain is a usage pattern on top: cordon (`NoSchedule`), then evict (`NoExecute`).
* Automatic rescheduling of evicted machines. `machinePoolRef` is immutable: Recreating a Machine on another pool requires a higher-level controller (or Gardener) which is out of scope.
* Toleration-based selective eviction. `NoExecute` evicts all machines unconditionally.
* Live migration of VMs between pools.
* Resource pressure eviction.
* Volume and NetworkInterface lifecycle on eviction. These follow existing ownership semantics and are not changed by this IEP.

## Proposal

### Comparison to Kubernetes

In Kubernetes, when a `NoExecute` taint is added to a Node, the taint eviction controller (part of the NodeLifecycleController) deletes all Pods that do not tolerate the taint. The kubelet sees the `deletionTimestamp` on each Pod, sends SIGTERM to containers, waits for the grace period, then reports back. The Pod object is removed once the kubelet confirms termination.

This IEP applies the same pattern to IronCore: a taint eviction controller deletes `Machine` objects when a `NoExecute` taint is set on a `MachinePool`. The poollet sees the `deletionTimestamp`, calls the provider to shut down the VM, and removes its finalizer to release the object.

Key differences from Kubernetes:
- No toleration matching: `NoExecute` evicts all machines unconditionally.
- Rescheduling is not built-in. Kubernetes has Deployments/ReplicaSets that recreate deleted Pods. IronCore does not yet have an equivalent higher-level controller.
- Kubernetes also has kubelet-initiated eviction: when a node runs low on resources (memory, disk, PIDs), the kubelet kills pods locally and marks them as `Failed` with reason `Evicted`. This is a separate mechanism from taint-based eviction. The kubelet acts autonomously without the control plane issuing a DELETE. Resource pressure eviction is explicitly out of scope for this IEP.

ref: [kubernetes taint eviction](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/tainteviction/taint_eviction.go#L147)

### API Changes

Extend the `TaintEffect` type in `api/common/v1alpha1/common_types.go`:

```go
const (
    TaintEffectNoSchedule TaintEffect = "NoSchedule"  (already present)
    TaintEffectNoExecute  TaintEffect = "NoExecute"
)
```

`NoSchedule` prevents new machines from being scheduled.
`NoExecute` additionally signals that all machines on the pool must be deleted.

### Eviction Flow

A new **taint eviction controller** watches `MachinePool` taint changes. When a `NoExecute` taint is added, it issues DELETE on all `Machine`s bound to that pool.

The poollet already sets a finalizer on every Machine:

```yaml
metadata:
  finalizers:
  - machinepoollet.ironcore.dev/machine
```

This finalizer ensures the object is not removed until the poollet confirms the VM is shut down.

1. NoExecute taint added to MachinePool
2. Taint eviction controller: DELETE Machine 
3. Machine gets deletionTimestamp (finalizer blocks removal)
4. Poollet reconciles machine
   1. Calls provider and deletes machine
   2. Removes finalizer
5. API server removes object


## Alternatives

### Keep Machine with an `Evicted` state instead of deleting

Instead of deleting the Machine object, it could be marked with a terminal state (e.g., `status.state = Evicted`) and left in the API. A higher-level controller could watch for evicted machines and recreate them, rather than watching for absence.

However, this adds complexity: dead objects accumulate and require garbage collection, the scheduler and poollet need to understand the new state, and it diverges from the Kubernetes taint eviction pattern which uses plain DELETE. Deletion is simpler and matches the established model.

### Clear machinePoolRef instead of deleting Machine

If `machinePoolRef` were mutable, eviction could mean "unbind and let the scheduler re-place." However, `machinePoolRef` is immutable by design.
