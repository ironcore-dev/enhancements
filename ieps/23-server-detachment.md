---
title: Server Parked State for Out-of-Band Day-2 Operations

iep-number: 23

creation-date: 2026-07-27

status: implementable

authors:

- "@afritzler"

reviewers:

- "@adracus"

---

# IEP-23: Server Parked State for Out-of-Band Day-2 Operations

## Table of Contents

- [Summary](#summary)
- [Motivation](#motivation)
    - [Goals](#goals)
    - [Non-Goals](#non-goals)
- [Proposal](#proposal)
    - [Lifecycle](#lifecycle)
- [Alternatives](#alternatives)
- [Open Questions](#open-questions)

## Summary

Introduce a first-class `Parked` state on the `Server` that parks it out of the
`ServerClaim` lifecycle so that an external component can take a server out of
the machinery, power it down, and run any out-of-band **day-2 operation** (a
firmware or BIOS/BMC update, hardware rework, diagnostics, low-level storage
reconfiguration, or any other procedure) that must not be fought by the
reconcilers. In particular the `ServerClaim` and `Server` reconcilers must not
boot the server back from disk via `BootConfigurationRef` during intermediate
restarts the procedure may trigger.

This is a deliberately **small, operational** mechanism: it adds one state,
triggered by `metal.ironcore.dev/operation: park`; the status state is the
acknowledgment.

## Motivation

Today the `Server` and `ServerClaim` reconcilers own the server's power and
boot state: a bound claim writes a `BootConfigurationRef` onto the `Server`
(which the `Server` reconciler boots from) and reconciles the server's power
against the claim. The `Server` reconciler also honors the out-of-band
`metal.ironcore.dev/operation` annotation for a fixed set of *power* verbs and
a generic `ignore` value that skips one object.

The gap: there is **no first-class "park me" state** that makes both
reconcilers stand down — stopping the `ServerClaim` reconciler from
(re)applying the boot configuration or reverting power, and the `Server`
reconciler from booting the claim's image — while still leaving the
`ServerClaim` object in place (bound), so that on recovery the claim
controller can take over again without re-scheduling.

### Concrete use case

A typical day-2 procedure has this shape:

1. The external actor takes the server out of the machinery and powers it
   down. (Whether it is allowed to do so on a claimed server is decided by an
   approval flow that is out of scope for this proposal and lives outside the
   operator.)
2. It performs the procedure, which may cause **intermediate restarts** of
   the server.
3. After the procedure, it brings the server back and signals the operator to
   resume.

The critical requirement: **during step 2, neither reconciler may boot the
claim image** on an intermediate restart nor "heal" the power state back to
what the claim wants; both controllers stand down until an explicit "bring
back" signal is received.

### Goals

- Add a `Parked` overlay state to the `Server` state machine that suspends
  normal progression, boot, and power healing while active.
- Provide a new `park` value on the existing out-of-band operation annotation
  (`metal.ironcore.dev/operation: park`) that an external component can set to
  request parking.
- Keep the `ServerClaim` bound during parking so it can resume ownership on
  recovery without re-scheduling.

### Non-Goals

- Defining or implementing the approval flow that decides whether a claimed
  server may be parked; that lives outside the operator.
- Performing or orchestrating the actual day-2 procedure (firmware updates,
  diagnostics, hardware rework, etc.).

## Proposal

Add a new `ServerStateParked` to the `Server` state machine, as an
**overlay state**: while `Parked`, normal state-machine progression, boot,
and power healing are suspended.

### Lifecycle

1. **Request.** The external actor sets the `metal.ironcore.dev/operation`
   annotation to `park` on the `Server`:

   ```yaml
   metadata:
     annotations:
       metal.ironcore.dev/operation: park
   ```
2. **Park.** The `Server` reconciler observes the request and:
   - powers the server **off** (idempotent; only if not already off),
   - ensures the claim's `BootConfigurationRef` is **not** on the active boot
     path so an intermediate restart cannot boot the claim image,
   - sets `status.state = Parked`.
3. **Stand down.** While `status.state == Parked`:
   - the `Server` reconciler returns early, before any power healing, boot,
     or state-machine progression,
   - the `ServerClaim` reconciler returns early, so the claim does not
     re-apply the boot configuration or revert power. The `ServerClaim` stays
     bound; its phase is unchanged.
4. **Resume.** The external actor removes the `metal.ironcore.dev/operation`
   annotation (or clears its `park` value). The next reconciliation re-enters
   the normal flow:
   - the `Server` reconciler refreshes system info (hardware or firmware
     state may have changed during the procedure),
   - transitions back to the pre-parking state: `Reserved` if the server
     has a `ServerClaimRef`, otherwise `Available` (or `Initial` if it had
     not yet been discovered),
   - the `ServerClaim` reconciler takes over again and re-applies the boot
     configuration and power state as before.

## Alternatives

- **A boolean spec field (`spec.parked`).** Rejected in favor of the operation
  annotation. The `metal.ironcore.dev/operation` grammar already exists as the
  established channel for an external actor to signal the operator to stand
  down, so reusing it keeps the API surface small and consistent with how power
  operations are already requested. A spec field would duplicate that channel
  without adding capability.
- **Reuse plain `metal.ironcore.dev/operation: ignore`.** Skips one object but
  does not power the server down or signal safe-to-proceed. `Parked` adds
  parking and a real observed state.

## Open Questions

- **Interaction with deletion.** If a `Server` is deleted while `Parked`, the
  deletion path must still be able to remove the finalizer and clean up the
  boot configuration. Confirm there is no deadlock when the server is parked.
