---
title: First-class Dell ServiceTag identifier
iep-number: NNNN
creation-date: 2026-06-11
status: implementable
authors:
  - "@A-AKPINAR"
reviewers:
  - TBD
---

# IEP-NNNN: First-class Dell ServiceTag identifier

## Table of Contents
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
- [Alternatives](#alternatives)

## Summary

On Dell hardware the ServiceTag is the canonical asset identifier used for inventory, support and
correlation with external systems. Today it is only available implicitly, collapsed into
`Server.status.sku` / `serialNumber` with no explicit identifier semantics. This IEP surfaces the
ServiceTag as a first-class, queryable identifier.

## Motivation

Operators identify Dell servers by ServiceTag in every other tool (DCIM, support portals, asset
databases). Without a first-class field, correlating a `Server` resource with those systems means
parsing an overloaded `sku`/`serialNumber` field, which is brittle and not self-documenting.

### Goals
- Expose the Dell ServiceTag as an explicit `Server.status.serviceTag` field.
- Propagate it as a label for selection and filtering.

### Non-Goals
- Integration with Dell Warranty/Support APIs.
- ServiceTag handling for non-Dell vendors.

## Proposal

Add an optional `serviceTag` field to `Server.status`, populated from the Dell OEM data already read
during discovery (currently collapsed into `sku`). Propagate the value as a label so servers can be
selected by ServiceTag, following the `metadata.metal.ironcore.dev/*` namespace established for
physical device metadata in
[IEP-14](https://github.com/ironcore-dev/enhancements/blob/main/ieps/14-device-metadata.md).

```yaml
metadata:
  labels:
    metadata.metal.ironcore.dev/service-tag: "7XQ20H3"
status:
  serviceTag: "7XQ20H3"
```

```sh
kubectl get servers -l metadata.metal.ironcore.dev/service-tag=7XQ20H3
```

## Alternatives

- **Keep using `sku` / `serialNumber`.** Rejected: overloaded and not self-documenting; consumers
  must know it happens to contain the ServiceTag on Dell.
- **Label-only, no status field.** Rejected: a status field is self-documenting and stable, and
  matches how IEP-14 treats identity data (status first); the label is a convenience on top of it,
  not a replacement.
- **A vendor-neutral `assetTag` abstraction.** A possible future generalization, but out of scope
  here; this IEP targets the concrete Dell ServiceTag gap.
