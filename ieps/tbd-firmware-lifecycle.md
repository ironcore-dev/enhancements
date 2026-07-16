---
title: Server firmware lifecycle - full inventory, declarative rollout, scheduled maintenance
iep-number: NNNN
creation-date: 2026-06-11
status: implementable
authors:
  - "@A-AKPINAR"
reviewers:
  - TBD
---

# IEP-NNNN: Server firmware lifecycle - full inventory, declarative rollout, scheduled maintenance

## Table of Contents
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Firmware inventory](#firmware-inventory)
  - [Declarative rollout strategy](#declarative-rollout-strategy)
  - [Scheduled maintenance window](#scheduled-maintenance-window)
- [Alternatives](#alternatives)

## Summary

The metal-operator already manages BIOS and BMC firmware versions (`BIOSVersion`, `BMCVersion` and
their `*Set` fleet variants) through a vendor-specific OEM layer (e.g. `bmc/redfish_dell.go`) that
handles staged-upgrade detection and Redfish task monitoring. Three lifecycle gaps remain for
operating large fleets: firmware versions are reported only for BIOS and BMC, fleet updates have no
rollout-safety controls, and maintenance is gated by state rather than time. This IEP closes those
three gaps as targeted extensions of the existing resources.

## Motivation

Operators running large bare-metal fleets need to see the complete firmware state of every
component, roll out firmware updates without taking down too many servers at once, and confine
disruptive updates to approved time windows. The building blocks exist today but stop short of
these capabilities, forcing manual coordination that does not scale to fleets of hundreds of
servers.

### Goals
- Surface complete per-server firmware inventory (BIOS, BMC, NIC, RAID, PSU) declaratively.
- Provide a declarative, safe rollout strategy for fleet firmware updates, including
  `maxUnavailable` and post-update verification.
- Enforce an image integrity (checksum/signature) check before a firmware image is applied.
- Automatically roll back to the previously installed firmware version when an update fails its
  post-apply verification.
- Allow maintenance actions to be scheduled into a defined time window.

### Non-Goals
- BIOS attribute configuration and drift remediation - already provided by `BIOSSettings`.
- Health/telemetry export - already provided by the existing event-based telemetry collector.
- Vendor-specific API surface: the proposed fields are vendor-neutral and implemented behind the
  existing BMC interface; vendor adapters (Dell, HPE, Lenovo, Supermicro) supply the OEM specifics.
- Hosting of the firmware repository itself.

## Proposal

### Firmware inventory

`SoftwareInventory` is already read internally for staged-upgrade checks but is not surfaced. We
add a read-only `firmwareInventory` list to `Server.status`, populated from
`UpdateService/FirmwareInventory`, covering NIC, RAID and PSU alongside BIOS/BMC. Inventory stays
co-located with the server it describes and reuses the existing read path.

```yaml
status:
  firmwareInventory:
    - component: BIOS
      currentVersion: "2.19.1"
      updateable: true
    - component: NIC.Integrated.1
      currentVersion: "22.5.7"
      updateable: true
    - component: RAID.Slot.3-1
      currentVersion: "51.16.0-4076"
      updateable: true
    - component: PSU.Slot.1
      currentVersion: "00.1D.7D"
      updateable: false
```

### Declarative rollout strategy

`BIOSVersionSet` already applies a version across a selected set of servers via `serverSelector` +
`biosVersionTemplate` (which carries `version`, `image`, and the existing
`serverMaintenancePolicy` / `retryPolicy` hooks). What is missing is fleet-level rollout safety
and image integrity. We add a `strategy` field on the set, and a `checksum` to the existing
`image` (`ImageSpec`) that must pass before the update is invoked. Shown on `BIOSVersionSet`; the
same fields apply to the other `*VersionSet` kinds.

```yaml
apiVersion: metal.ironcore.dev/v1alpha1
kind: BIOSVersionSet
spec:
  serverSelector:                     # existing: selects on user-applied server labels
    matchLabels:
      server-group: poweredge-prod
  biosVersionTemplate:                # existing
    version: "2.21.2"
    image:                            # existing (ImageSpec: URI, transferProtocol, secretRef)
      URI: "https://fw.example.com/dell/bios/BIOS_2.21.2.EXE"
      transferProtocol: HTTP
      checksum:                       # NEW: verified before apply
        algorithm: SHA256
        value: "9f2c…"
  strategy:                           # NEW: fleet rollout safety
    maxUnavailable: 10%               # rollout limit across the selected fleet
    verifyVersionAfterApply: true     # post-update version gate before continuing
    rollbackOnFailure: true           # NEW: revert to the previous version if verification fails
```

When `verifyVersionAfterApply` fails, `rollbackOnFailure` reverts the affected server to its
previously installed version before the rollout continues; that version is recorded in the
resource status as the rollback target. The iDRAC retains the prior firmware image, which makes
the revert possible.


### Scheduled maintenance window

`ServerMaintenance` gates updates by state today. We add an optional `window` (cron schedule +
duration) so an enforced maintenance applies only inside an approved window, mapped onto the
Redfish deferred apply-time semantics (`AtMaintenanceWindowStart`) where the BMC supports them.

```yaml
apiVersion: metal.ironcore.dev/v1alpha1
kind: ServerMaintenance
spec:
  serverRef:                  # existing (required)
    name: server-sample
  policy: Enforced            # existing
  window:                     # NEW
    schedule: "0 2 * * SAT"   # cron: Saturdays at 02:00
    duration: 4h
```

## Alternatives

- **A dedicated firmware-policy CRD** declaring target versions per component plus rollout
  strategy. Rejected as the default because the `*VersionSet` kinds already perform fleet updates;
  extending them avoids a parallel, overlapping API surface. Worth reconsidering if maintainers
  prefer to decouple rollout policy from version sets.
- **A separate `ServerFirmwareInventory` resource** per server instead of `Server.status`.
  Rejected: inventory is intrinsic to a server and benefits from being co-located, not a separate
  object.
- **An external scheduler** that creates `ServerMaintenance` references on a timer, instead of a
  `window` field. Rejected: keeps scheduling out of the API and duplicates state outside the
  cluster.
- **Do nothing / manual coordination.** Rejected: does not scale to large fleets and provides no
  rollout safety.