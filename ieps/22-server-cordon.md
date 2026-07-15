---
title: Server Cordoning

iep-number: 22

creation-date: 2026-07-13

status: implementable

authors:

- "@afritzler"

reviewers:

- "@adracus"

---

# IEP-22: Server Cordoning

## Table of Contents

- [Summary](#summary)
- [Motivation](#motivation)
    - [Goals](#goals)
    - [Non-Goals](#non-goals)
- [Proposal](#proposal)
- [Alternatives](#alternatives)

## Summary

Introduce a first-class, typed **cordon** signal on the `Server` resource that
prevents new `ServerClaim`s from binding to it.

## Motivation

Today there is no single explicit signal indicating that a server should not
receive new claims. This state is implied by a combination of
`metal.ironcore.dev/*` labels and annotations together with
`ServerMaintenanceRef` and the server's state machine. Because the signal is
implicit and stringly-typed, different components, placement logic, the
maintenance controllers, and external workload managers, each independently
re-derive the same "is this server claimable?" answer. This leads to
inconsistency and drift between components.

A typed cordon field makes the signal explicit and observable in `status`, so
consumers can react without scraping labels, and a single authority (the
`ServerClaim` reconciler) decides claimability.

### Goals

* Introduce a typed cordon concept on `Server` that is independent of the
  existing maintenance label / annotation flow.
* Make cordon observable through `status` so consumers can react without
  scraping labels.
* Ensure placement and claim-binding logic honors the cordon signal: a
  cordoned `Server` MUST NOT receive new `ServerClaim` bindings (i.e. it becomes
  unclaimable).

### Non-Goals

* Redesigning the `Server` state machine
  (`Initial -> Discovery -> Available -> Reserved`). Cordon is orthogonal to
  phase progression.
* Eviction / draining of already-bound claims. Cordon only blocks **new**
  bindings.
* Replacing existing in-tree `ServerMaintenance` usage. A deprecation window
  is expected; cordon coexists with the current label / annotation flow
  during the transition.

## Proposal

Add a cordon field to `Server.spec`. The `ServerClaim` reconciler /
scheduler MUST NOT bind a new `ServerClaim` to a `Server` that is cordoned.
Already-bound claims are unaffected by cordon alone.

### API Change

We add `spec.unclaimable: bool` to `Server`. The field is the single source
of truth for "do not place new claims here". We deliberately choose a spec
field over a label / annotation: it gives schema validation, natural printer
columns, and avoids mirroring the stringly-typed approach we are moving away
from.

```go
type ServerSpec struct {
	// ... existing fields ...

	// Unclaimable, when true, prevents new ServerClaims from being bound to
	// this Server. Already-bound claims are unaffected.
	// +kubebuilder:default=false
	// +optional
	Unclaimable bool `json:"unclaimable,omitempty"`
}
```

### Placement and Claim Binding

The `ServerClaim` reconciler already resolves candidate servers via a label
selector and / or an explicit `serverRef`. The scheduler is extended to skip
any candidate whose `spec.unclaimable` is `true`:

* A claim with an explicit `serverRef` to a cordoned server stays `Pending`
  with a condition explaining why it is not bound.
* A claim using a `serverSelector` skips cordoned candidates when filtering.
  If no uncordoned candidate matches, the claim stays `Pending`.

Cordon does **not** affect an already-bound claim. The `serverClaimRef` on a
`Server` remains in place while the server is cordoned; cordon only prevents
**new** bindings.

### Interaction with the State Machine

Cordon is orthogonal to the `Initial -> Discovery -> Available -> Reserved`
state machine. It affects claimability, not phase progression. A server may be
cordoned in any state; in particular, a cordoned server in `Available` simply
will not be picked up by new claims until uncordoned.

### Who Can Set / Unset Cordon

Cordon is a plain `spec` field, so any subject with `update` permission on the
`Server` resource can toggle it. We do not introduce a dedicated subresource
for toggling cordon in this iteration. The intended writers are operators /
admins for manual maintenance and automated maintenance controllers.

## Alternatives

Keep cordon implicit and derived from the existing
`metal.ironcore.dev/maintenance-*` labels and annotations plus
`ServerMaintenanceRef`. This is the status quo: it works, but it forces every
consumer to re-derive claimability from a combination of stringly-typed
labels and annotations, which is the source of the inconsistency this proposal
sets out to remove. We explicitly move away from it.

Use a dedicated label such as `metal.ironcore.dev/unclaimable` instead of a
spec field. This mirrors today's approach and inherits its drawbacks: no
schema validation, no natural printer columns, and it keeps the signal in
labels, exactly what this proposal moves away from.

Introduce cordon as a separate CRD (e.g. `ServerCordon`). This adds an extra
object and a watch for no gain over a single boolean on the resource that
already represents the server. A spec field is simpler and sufficient for the
cordon-only (block-new-bindings) contract described here.