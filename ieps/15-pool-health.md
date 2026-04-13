---
title: Pool Health

iep-number: 15

creation-date: 2026-05-13

status: implementable

authors:

- "@adracus"

reviewers:

- "@afritzler"
- "@balpert89"

---

# IEP-15: Pool Health

## Table of Contents

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
- [Alternatives](#alternatives)

## Summary

To determine the availability and health of each pool (`compute`, `storage`),
we propose introducing heartbeats and status conditions of pools.

Similar to Kubernetes' node heartbeats & status conditions, there are 2
parties involved in this:

1. The pool itself, regularly reporting its status by patching the `*Pool`
   resource's status and acquiring / keeping a lease object within the
   `ironcore-<type>-lease` namespace (with `type` being either `machinepool`,
   `volumepool` or `bucketpool`).
   For references on leases, see the [leases Kubernetes documentation](https://kubernetes.io/docs/concepts/architecture/leases/).
2. A pool controller within the IronCore control plane that updates
   a pool object's status conditions depending on the heartbeats of the pool.

## Motivation

Currently, there are no means to determine whether a pool is healthy or ready
other than trying to interact with the pool. Additionally, features like
pool rolling and / or eviction need proper health information of pools to
function properly.

### Goals

* Require a well-defined lease per pool to be acquired / kept.
* Expose health information on a pool's `.status.condtions`.
* Extend / create pool reconcilers that monitor the pools and update
  pool status if pools fail to do their heartbeat.

### Non-Goals

* Handle eviction - Eviction is a subsequent process that depends on proper
  health information. It also needs to be discussed in which form machines
  / volumes can / should be evicted.
* Handle pool rolling - needs to be done subsequently as well.

## Proposal

For each IronCore installation, create the following additional namespaces:
* `ironcore-machinepool-lease` for `MachinePool` leases.
* `ironcore-volumepool-lease` for `VolumePool` leases.
* `ironcore-bucketpool-lease` for `BucketPool` leases.

Inside these namespaces, for each `<Type>Pool`, a pool controller
must acquire and keep a `coordination.k8s.io/Lease`:
For example, if there is a `MachinePool`
`my-machinepool`, the controller of this `MachinePool` must acquire and
keep a lease called `ironcore-machinepool-lease/my-machinepool`. The
recommended default for setting on the lease is 40 seconds.

```yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: my-machinepool
  namespace: ironcore-machinepool-lease
spec:
  holderIdentity: machinepoollet-07a5ea9b9b072c4a5f3d1c3702_0c8914f7-0f35-440e-8676-7844977d3a05
  leaseDurationSeconds: 3600
  renewTime: "2026-05-01T13:18:48.065888Z"
```

Additionally, each pool controller must regularly update a condition
called `Ready` in a `Pool`'s `.status.conditions`. By default, an
interval of 10 seconds is recommended (taken from Kubernetes' kubelet defaults).
The pool controller must set this condition to `True` if the pool is ready
for workload, otherwise it must explicitly set it to `False`.

Healthy example:

```yaml
apiVersion: compute.ironcore.dev/v1alpha1
kind: MachinePool
metadata:
  name: machinepool-sample
spec:
  providerID: ironcore://shared
status:
  conditions:
  - type: Ready
    status: True
    reason: HeartbeatReceived
    message: A heartbeat for the machine pool has been received within the allotted time.
    observedGeneration: 4
    lastTransitionTime: "2026-05-01T13:18:48.065888Z"
```

Unhealthy example:

```yaml
apiVersion: compute.ironcore.dev/v1alpha1
kind: MachinePool
metadata:
  name: machinepool-sample
spec:
  providerID: ironcore://shared
status:
  conditions:
  - type: Ready
    status: Unknown
    reason: NoHeartbeatReceived
    message: No heartbeat for the machine pool has been received within the past time.
    observedGeneration: 4
    lastTransitionTime: "2026-05-01T13:18:48.065888Z"
```

For each pool type (`MachinePool`, `VolumePool`, `BucketPool`), a controller is
created / reconciler amended to the `ironcore-dev` controllers (not the `*poollet*`
that monitors the pools of that type.

If a pool is unresponsive for a specific duration, its `Ready` condition
is marked as `Unknown`. A recommended default for the
`<type>pool-monitor-grace-period` is 50 seconds. A pool is deemed
unresponsive if there was no progress on neither the pool's status
nor the pool's lease.

> [!IMPORTANT] The pool controller should not inspect timestamps
> of the conditions / leases. This is prone to failure due to clock
> skew. Instead, pool controllers should check if there were updates
> since the last inspection and only if there were no updates for a
> specific duration mark the pool as non-ready.

`pool.status.state` will be deprecated in favor of the more fine-grained
`Ready` condition (and potentially other future conditions).

## Alternatives

* Keep it as-is and work via trial-and-error.
