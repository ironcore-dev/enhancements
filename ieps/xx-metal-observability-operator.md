---
title: Introduce metal-observability-operator as the observability home for the metal stack

iep-number: XX

creation-date: 2026-07-29

status: implementable

authors:

- "@xkonni"

reviewers:

- "@TBD"

---

# IEP-XX: Introduce metal-observability-operator as the observability home for the metal stack

## Table of Contents

- [Summary](#summary)
- [Motivation](#motivation)
    - [Goals](#goals)
    - [Non-Goals](#non-goals)
- [Proposal](#proposal)
- [Alternatives](#alternatives)
    - [Alternative 1: Merge telemetry pipeline as-is into metal-maintenance-operator](#alternative-1-merge-telemetry-pipeline-as-is-into-metal-maintenance-operator)
    - [Alternative 2: Keep metrics in metal-operator](#alternative-2-keep-metrics-in-metal-operator)

## Summary

This IEP proposes creating `metal-observability-operator` as a dedicated home for observability concerns in the metal stack, and defines a clean split of responsibilities with `metal-maintenance-operator` for the ongoing BMC telemetry pipeline work.

`metal-maintenance-operator` manages BMC subscriptions — it knows the BMCs, holds the credentials, and reconciles subscription state. `metal-observability-operator` owns the inbound side: the Redfish event receiver, Prometheus sinks, and BMC service discovery. If `metal-observability-operator` is not deployed, `metal-maintenance-operator` simply does not create subscriptions. The system degrades gracefully.

## Motivation

Work is underway to add a Redfish telemetry pipeline to the metal stack (see [metal-maintenance-operator#110](https://github.com/ironcore-dev/metal-maintenance-operator/pull/110)). The current plan places the entire pipeline — BMC subscription reconciliation, an inbound HTTP event receiver on `:9092`, Prometheus sinks, and BMC service discovery — inside `metal-maintenance-operator`.

The BMC subscription reconciler is a reasonable fit for `metal-maintenance-operator`: creating and managing subscriptions on a BMC is the same pattern as `BMCSetting` reconciliation. However, the event receiver is a different kind of component. It is an externally-reachable HTTP listener with its own port, service, and availability requirements — a different operational profile from a reconciliation loop. Co-locating the two means `metal-maintenance-operator` must be operated, scaled, and secured as both a reconciler and a network service simultaneously.

A dedicated `metal-observability-operator` gives the listener, Prometheus sinks, and service discovery a proper home with a clear name and independent operational lifecycle.

The bootstrapping cost is real but bounded: `metal-maintenance-operator` was recently created from scratch, so the team has a clear template and recent experience to draw from.

### Goals

- Establish `metal-observability-operator` as the canonical home for the Redfish event receiver, Prometheus sinks, and BMC service discovery (`/sd/bmcs`).
- Keep BMC subscription management in `metal-maintenance-operator`, where it fits alongside existing BMC lifecycle concerns.
- Allow `metal-observability-operator` to be deployed and scaled independently from `metal-maintenance-operator`.
- Provide a coherent, extensible home for future observability additions without scope creep into `metal-maintenance-operator`.

### Non-Goals

- Redefining the scope or responsibilities of `metal-maintenance-operator` beyond subscription management.
- Implementing dashboards, alerting rules, or scrape configuration — those belong in deployment-specific configuration.
- Resolving the `CriticalEventReceived` condition write — deferred to a follow-up enhancement.

## Proposal

Create `metal-observability-operator` and split the telemetry pipeline as follows:

**`metal-maintenance-operator`** reconciles BMC subscriptions. It is configured with the endpoint URL of `metal-observability-operator`'s event receiver. If that URL is not configured (i.e. `metal-observability-operator` is not deployed), subscription creation is skipped. No subscriptions means no events — the system degrades gracefully without entering a broken state.

**`metal-observability-operator`** owns:
- The Redfish event receiver (`:9092`) — inbound HTTP listener for BMC-pushed events
- Prometheus sinks — event counters and metric gauges
- `/sd/bmcs` service discovery endpoint

The coupling between the two operators is explicit and unidirectional: `metal-maintenance-operator` depends on `metal-observability-operator`'s address, not the other way around.

The bootstrapping cost is bounded: `metal-maintenance-operator` was recently created from scratch and serves as the template.

The `CriticalEventReceived` condition write on `Server` objects is out of scope for this IEP and will be addressed in a follow-up, once the operational boundary is established.

## Alternatives

### Alternative 1: Merge telemetry pipeline as-is into metal-maintenance-operator

Place the entire pipeline — subscription reconciler, event receiver, Prometheus sinks, service discovery — in `metal-maintenance-operator`.

The event receiver on `:9092` is an externally-reachable HTTP listener with a different operational profile from a reconciliation loop. Running both in the same binary means `metal-maintenance-operator` must be operated, scaled, and secured as both a reconciler and a network service.

### Alternative 2: Keep metrics in metal-operator

Revert the migration and continue exposing metrics from `metal-operator` directly.
