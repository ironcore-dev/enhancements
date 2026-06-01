---
title: Server Reclaim Taints

iep-number: tbd

creation-date: 2026-06-01

status: implementable

authors:

- "@adracus"

reviewers:

- "@ironcore-dev/metal-operator-maintainers"

---

# IEP-tbd: Server Reclaim Taints

## Table of Contents

- [Summary](#summary)
- [Motivation](#motivation)
    - [Goals](#goals)
    - [Non-Goals](#non-goals)
- [Proposal](#proposal)
- [Alternatives](#alternatives)

## Summary

`Server` objects can have taints applied once they return from being claimed.
Using this feature, custom workflows can be set up for dealing with
reclaimed `Server`s. This allows e.g. to clean and / or check a `Server` before
handing it out for claiming again.

## Motivation

Currently, when a `Server` is freed from a claim, it becomes claimable right
away again. This causes an issue, as e.g. the disk of the `Server` should be
cleaned or some checks should be run before handing the `Server` out for
claiming it again.

### Goals

- Define how servers are tainted after use

### Non-Goals

- (Continuously) monitor servers for readiness

## Proposal

Introduce `Server.spec.reclaimTaints`, allowing to specify a list
of taints to apply to the server once it gets reclaimed.

Every time a `Server` with `reclaimTaints` transitions from
`state: Reserved` to `state: Available`, the taints specified in
`spec.reclaimTaints` are added to `spec.taints`, overriding any
duplicate key with the values in `spec.reclaimTaints`.

```yaml
apiVersion: metal.ironcore.dev/v1alpha1
kind: Server
metadata:
  name: my-server
spec:
  reclaimTaints:
  - key: sanitization
    value: required
    effect: NoSchedule
  - key: performance
    value: check
    effect: PreferNoSchedule
```

The added taints can then be reconciled by an external controller,
allowing the external controller to remove the taints once e.g. a cleanup
and / or inspection operation has succeeded.

## Alternatives

- Use a `MutatingWebHookConfiguration` patching the `Server` when it transitions
  between phases. This however has lots of implicit logic and is not easily flexible
  per server.
- Always apply taints to servers once they are scheduled / dynamically add them
  once an update event happens. However, this allows for race conditions and suboptimal
  behavior.