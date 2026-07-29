---
title: Introduce metal-observability-operator as home for metal metrics

iep-number: 24

creation-date: 2026-07-29

status: implementable

authors:

- "@xkonni"

reviewers:

- "@TBD"

---

# IEP-24: Introduce metal-observability-operator as home for metal metrics

## Table of Contents

- [Summary](#summary)
- [Motivation](#motivation)
    - [Goals](#goals)
    - [Non-Goals](#non-goals)
- [Proposal](#proposal)
- [Alternatives](#alternatives)

## Summary

Metrics are being moved out of `metal-operator`. This proposal argues that the migration target should be a new, dedicated `metal-observability-operator` rather than `metal-maintenance-operator`, which is the currently planned target.

## Motivation

Moving metrics out of `metal-operator` is the right direction. However, the current target — `metal-maintenance-operator` — is a convenience choice driven by availability rather than architectural intent. Maintenance and observability are distinct concerns; co-locating them produces a component with a blurred responsibility boundary that becomes harder to reason about, document, and evolve independently.

`metal-observability-operator` makes the responsibility boundary explicit and mirrors the approach taken in IEP-13 for the SONiC stack: observability as a first-class, named concern with its own operator.

### Goals

- Establish `metal-observability-operator` as the canonical home for metrics (and potentially other observability signals) originating from the metal stack.
- Keep maintenance and observability concerns in separate operators with clearly named responsibilities.
- Provide a coherent, extensible home for future observability additions (alerting rules, health checks, status reporting) without scope creep into `metal-maintenance-operator`.

### Non-Goals

- Defining the full set of metrics to be migrated — that work continues independently.
- Replacing or modifying `metal-maintenance-operator` beyond removing metrics from its scope.
- Implementing dashboards, alerting rules, or scrape configuration — those belong in deployment-specific configuration.

## Proposal

Create a new `metal-observability-operator` as the migration target for metrics currently in `metal-operator`. The operator follows the same Prometheus/ServiceMonitor patterns established elsewhere in the stack.

The name communicates purpose unambiguously. It also gives the observability surface room to grow (additional signals, new exporters, health-check aggregation) without widening the scope of an operator that exists for a different reason.

The bootstrapping cost is real but bounded: `metal-maintenance-operator` was recently created from scratch, so the team has a clear template and recent experience to draw from. The investment buys a structure that won't need untangling later.

Migration of the metrics themselves is out of scope for this IEP and proceeds as currently planned, with only the target operator changed.

## Alternatives

### Alternative 1: Move metrics to metal-maintenance-operator

The currently planned approach. `metal-maintenance-operator` exists and is available, which reduces upfront work.

**Rejected because:** availability is not an architectural reason. Maintenance and observability are independent concerns. Combining them in one operator makes the component harder to describe, harder to evolve independently, and sets a precedent for further scope creep. The short-term cost saving of not bootstrapping a new operator is outweighed by the long-term cost of untangling a mixed-responsibility operator.

### Alternative 2: Keep metrics in metal-operator

Revert the migration and continue exposing metrics from `metal-operator` directly.

**Rejected because:** the motivation for moving metrics out of `metal-operator` is sound and not revisited here.
