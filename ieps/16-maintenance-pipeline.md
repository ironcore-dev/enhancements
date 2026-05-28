---
title: MaintenancePipeline — Unified firmware and settings lifecycle for bare-metal fleets

iep-number: 0016

creation-date: 2026-05-15

status: draft

authors:

- "@nagadeesh-nagaraja"

reviewers:

- "@afritzler"
- "@ironcore-dev/metal-operator-maintainers"

---

# IEP-0016: MaintenancePipeline — Unified firmware and settings lifecycle for bare-metal fleets

## Table of Contents

- [Summary](#summary)
- [Motivation](#motivation)
    - [Goals](#goals)
    - [Non-Goals](#non-goals)
- [Proposal](#proposal)
    - [New API types](#new-api-types)
    - [MaintenancePipeline](#maintenancepipeline)
    - [MaintenancePipelineRun](#maintenancepipelinerun-auto-generated)
    - [Stage execution order](#stage-execution-order)
    - [BMC : Server — the 1:N relationship](#bmc--server--the-1n-relationship)
    - [Drift policy](#drift-policy)
    - [Drift recovery](#drift-recovery)
    - [Version hydration for version-agnostic stages](#version-hydration-for-version-agnostic-stages)
- [API changes and migration](#api-changes-and-migration)
    - [BMCSettings — settingsFlow replaces flat settings map](#bmcsettings--settingsflow-replaces-flat-settings-map)
    - [driftPolicy — new field on BMCSettings and BIOSSettings](#driftpolicy--new-field-on-bmcsettings-and-biossettings)
    - [BMC and Server settings refs — single ref becomes a list](#bmc-and-server-settings-refs--single-ref-becomes-a-list)
- [Alternatives](#alternatives)

## Summary

`MaintenancePipeline` is a new higher-level API that expresses the full desired maintenance lifecycle for a bare-metal fleet in a single authored resource. It orchestrates ordered execution of BMC and BIOS settings and upgrade stages across all servers matched by a label selector; it also handles the 1-BMC-to-N-servers relationship automatically, reuses drift detection and failure recovery from the existing controllers, and controls the order at a higher level.

## Motivation

The existing low-level APIs (`BMCSettings`, `BMCVersion`, `BIOSSettings`, `BIOSVersion`) require operators to:

1. Manually author multiple separate objects per cluster type and wire them together.
2. Manual intervention required to express ordering dependencies.
3. Manually Define and manage BMC firmware version upgrade hops as separate objects per hop.
4. Separately manage BIOS settings/versions (Server-scoped) and BMC settings/versions (BMC-scoped) with no unified view of the full lifecycle.

The 1 BMC → N Servers relationship is implicit and handled piecemeal:
- BMC-scoped resources (`BMCSettings`, `BMCVersion`) must be stamped once per BMC.
- Server-scoped resources (`BIOSSettings`, `BIOSVersion`) must be stamped once per Server.

There is no single object that represents "apply this full firmware + settings sequence to all servers of this type", and no mechanism for detecting and recovering from hardware regressions (factory resets, failed upgrades) after the initial run completes in a particular ordered workflow.

### Goals

- Single authored resource expresses the full desired maintenance lifecycle for a fleet.
- Explicit stage ordering via list position — stages execute strictly one at a time in list order.
- Version upgrade hops are explicit pipeline stages — one `BMCVersion`/`BIOSVersion` stage per hop, each targeting exactly one firmware version.
- BMC-scoped stages run once per unique BMC (de-duplicated across servers sharing that BMC).
- Server-scoped stages run independently per Server, gated on their BMC's stage completion.
- Fleet-level blast radius controlled via `maxConcurrent` (how many BMCs progress simultaneously).
- Drift re-reconciliation is first-class: hardware regression triggers re-execution of affected stages.

### Non-Goals

- Replacing or modifying the existing low-level APIs (`BMCSettings`, `BMCVersion`, `BIOSSettings`, `BIOSVersion`) — `MaintenancePipeline` orchestrates them, not replaces them.
- Other firmwares upgrades are not part of this design at this point in time.

## Proposal

### New API types

Two new CRDs are introduced:

| Type | Scope | Authored by | Description |
|---|---|---|---|
| `MaintenancePipeline` | Cluster | Operator | Desired lifecycle spec: server selector, stage list, concurrency, drift policy |
| `MaintenancePipelineRun` | Cluster | Controller | One per unique BMC matched by the pipeline; owns all child resources |

### MaintenancePipeline

```yaml
apiVersion: metal.ironcore.dev/v1alpha1
kind: MaintenancePipeline
metadata:
  name: ilo6-upgrade-to-v180
spec:
  # Selects Servers. Controller resolves server.spec.bmcRef to get the BMC.
  serverSelector:
    matchLabels:
      kubernetes.metal.cloud.sap/bmc-vendor: hpe

  # Max number of MaintenancePipelineRun objects (unique BMCs) progressing simultaneously.
  maxConcurrent: 3

  # What to do when hardware deviates from the desired state after completion.
  # Reconcile: re-run affected stages from the earliest dirty stage.
  # Observe: surface a DriftDetected condition but take no action.
  driftPolicy: Reconcile

  stages:
    # BMC-scoped stages — run once per BMC.
    - name: bmc-baseline
      kind: BMCSettings
      template:
        settingsFlow:
          - name: network-baseline
            priority: 1
            settings:
              SyslogServer: "10.0.0.5"
              SNMPv3AuthProtocol: SHA256

    # One stage per version hop — hardware already at or past a hop completes immediately.
    - name: bmc-fw-v160
      kind: BMCVersion
      template:
        version: "iLO 6 v1.60"
        image:
          URI: "http://firmware-repo.internal/ilo6/v160/ilo6_160.bin"

    - name: bmc-fw-v180
      kind: BMCVersion
      template:
        version: "iLO 6 v1.80"
        image:
          URI: "http://firmware-repo.internal/ilo6/v180/ilo6_180.bin"

    - name: bmc-post-upgrade
      kind: BMCSettings
      template:
        serverMaintenancePolicy: Enforced
        settingsFlow:
          - name: post-upgrade-tuning
            priority: 1
            settings:
              ProcTurbo: Enabled

    # Server-scoped stages — run once per Server in serverRefs.
    # version omitted — Run hydrates from server.status.biosVersion at stamp time.
    - name: bios-pre-upgrade
      kind: BIOSSettings
      template:
        serverMaintenancePolicy: Enforced
        settingsFlow:
          - name: pxe-prepare
            priority: 1
            settings:
              PxeBootToFirstInterface: Enabled

    - name: bios-fw-v250
      kind: BIOSVersion
      template:
        version: "U54 v2.50"
        image:
          URI: "http://firmware-repo.internal/bios/u54/v250/bios_v250.bin"

    - name: bios-post-upgrade
      kind: BIOSSettings
      template:
        version: "U54 v2.50"
        serverMaintenancePolicy: Enforced
        settingsFlow:
          - name: performance-tuning
            priority: 1
            settings:
              ProcHyperthreading: Enabled
              NUMAGroupSizeOpt: Clustered
```

### MaintenancePipelineRun (auto-generated)

One `MaintenancePipelineRun` is created per unique BMC matched by the pipeline's `serverSelector`. It owns all child resources for both the BMC and all Servers sharing it.

```yaml
apiVersion: metal.ironcore.dev/v1alpha1
kind: MaintenancePipelineRun
metadata:
  name: ilo6-upgrade-to-v180-x7k9f   # generated suffix; never constructed deterministically
  labels:
    metal.ironcore.dev/pipeline: ilo6-upgrade-to-v180
    metal.ironcore.dev/bmc: bmc-abc123
  ownerReferences:
    - kind: MaintenancePipeline
      name: ilo6-upgrade-to-v180
      controller: true
spec:
  bmcRef:
    name: bmc-abc123
  serverRefs:
    - name: server-a
    - name: server-b
    - name: server-c
status:
  phase: InProgress
  totalStages: 7
  pendingStages: 4
  activeStages: 1
  completedStages: 2
  failedStages: 0
  stages:
    - name: bmc-baseline
      phase: Completed
      childRef: {kind: BMCSettings, name: bmc-baseline-p4m2q}
    - name: bmc-fw-v160
      phase: Completed
      childRef: {kind: BMCVersion, name: bmc-fw-v160-n8rt3}
    - name: bmc-fw-v180
      phase: InProgress
      childRef: {kind: BMCVersion, name: bmc-fw-v180-m3kx9}
    - name: bmc-post-upgrade
      phase: Pending
    - name: bios-pre-upgrade
      phase: Pending
      servers:
        - serverRef: {name: server-a}
          phase: Pending
        - serverRef: {name: server-b}
          phase: Pending
    - name: bios-fw-v250
      phase: Pending
      servers: [...]
    - name: bios-post-upgrade
      phase: Pending
      servers: [...]
  conditions:
    - type: Progressing
      status: "True"
    - type: DriftDetected
      status: "False"
```

### Stage execution order

Stages execute **strictly in list order**. The Run controller stamps the child for stage `N` only after stage `N-1` reaches `Completed`. There is no parallelism within a single Run.

For Server-scoped stages (`BIOSSettings` / `BIOSVersion`), the Run stamps children **simultaneously for all servers** in `serverRefs`. Each server then advances through subsequent Server-scoped stages **independently** — a server moves to the next stage as soon as its own child completes, without waiting for other servers.

The aggregate phase for a Server-scoped stage is defined explicitly as follows:

| Aggregate phase | Meaning |
|---|---|
| `Pending` | No server child has started yet |
| `InProgress` | At least one server child is progressing |
| `Failed` | At least one server child failed |
| `Completed` | All server children completed |

Phase precedence is `Failed` > `InProgress` > `Pending` > `Completed` only when computing the aggregate from mixed server states; `Completed` means every server child finished successfully.

`maxConcurrent` caps **fleet-level** parallelism: how many `MaintenancePipelineRun` objects (unique BMCs) may be `InProgress` simultaneously. Stages within a single Run are always sequential.

### BMC : Server — the 1:N relationship

The pipeline controller creates **one `MaintenancePipelineRun` per unique BMC**. The Run owns and manages all child resources — both BMC-scoped and Server-scoped:

```
MaintenancePipeline
    ├── MaintenancePipelineRun/bmc-abc        (one Run per unique BMC)
    │       ├── BMCSettings/bmc-abc-baseline   (BMC-scoped, once per Run)
    │       ├── BMCVersion/bmc-abc-fw-v180     (BMC-scoped, once per hop per Run)
    │       ├── BIOSSettings/server-a-bios-pre (Server-scoped, once per Server)
    │       ├── BIOSSettings/server-b-bios-pre
    │       ├── BIOSVersion/server-a-bios-v250
    │       └── BIOSVersion/server-b-bios-v250
    └── MaintenancePipelineRun/bmc-xyz        (separate Run for a different BMC)
```

BMC firmware upgrade happens exactly once per BMC regardless of how many Servers share it. BIOS settings/versions are stamped per Server by the same Run.

### Drift policy

`driftPolicy` operates at two levels with three values:

| Value | On `MaintenancePipeline` | On child resource |
|---|---|---|
| `Reconcile` | Re-run from the earliest dirty stage on drift | — (not valid on children) |
| `Observe` | Surface `DriftDetected` condition, no action | Detects drift, no maintenance window, no hardware action |
| `Suspend` | — (not valid at pipeline level) | Completely frozen: no reconciliation, no drift detection |

All child types expose a `spec.driftPolicy` field that the Run patches after each stage completes:

| `spec.driftPolicy` | Child controller behaviour |
|---|---|
| `""` (empty, default) | **Active** — reconciles hardware, requests maintenance windows, applies changes |
| `Observe` | **Read-only** — detects drift and updates status conditions, but requests no maintenance windows and applies no changes |
| `Suspend` | **Frozen** — no reconciliation, no drift detection, no status updates, no hardware actions |

**Settings stages** (`BMCSettings` / `BIOSSettings`) are always patched to `Observe` after completion. Drift on a settings stage is always meaningful — there is no concept of a "superseded" settings state.

**Version stages** (`BMCVersion` / `BIOSVersion`) may appear multiple times as sequential upgrade hops. When a later stage of the same kind exists in the pipeline, the earlier hop is *superseded*: the hardware is expected to be at a higher version, so that child is patched to `Suspend` to silence false-positive drift alerts. Only the terminal version stage of each kind receives `Observe`.

For example, in a pipeline with BMC hops `v1.60 → v1.70 → v1.80`:
- `bmc-fw-v160` and `bmc-fw-v170` → `Suspend` (superseded — hardware at v1.80 ≠ v1.60 is not a regression)
- `bmc-fw-v180` → `Observe` (terminal — hardware regressing below v1.80 is real drift)

### Drift recovery

When an `Observe`-mode child detects hardware drift (desired state no longer satisfied), it transitions `Completed → Pending` without requesting a maintenance window. The Run's watch fires and:

1. Patches `driftPolicy: Observe` on any currently `InProgress` child (halts hardware actions).
2. Evaluates each stage against live state to find the earliest dirty stage (`Suspend` children are skipped — intermediate hops cannot be dirty by definition).
3. Patches `driftPolicy: ""` (re-activates) the existing child at the reset point. For version-agnostic settings stages, also patches `spec.version` from current hardware state.
4. Advances through stages sequentially from the reset point, reusing existing child objects throughout — no objects are deleted or re-created.

Only servers/stages that drifted are re-executed. A partial BIOS regression on `server-a` only resets `server-a`'s sub-status from the earliest dirty BIOS stage forward; BMC stages and `server-b`/`server-c` are untouched.

#### Re-execution in order

The following timeline shows a full factory reset scenario where BMC regressed to `iLO 6 v1.50` and all servers regressed to `U54 v2.10`. The Run re-activates each existing child object in list order — no objects are deleted or re-created:

```
[T+0]  Run patches spec.driftPolicy: "" on existing BMCSettings/bmc-baseline
         → controller applies DNS, Syslog, SNMP → Applied
         → Run patches driftPolicy: Observe  (settings stage → always Observe)

[T+1]  bmc-baseline Completed
         Run patches spec.driftPolicy: "" on existing bmc-fw-v160 child
           (v1.50 ≠ v1.60 → child re-executes, requests maintenance window, flashes)

[T+2]  BMCVersion v1.60 Completed
         → later same-kind stage exists → Run patches driftPolicy: Suspend
         Run patches spec.driftPolicy: "" on existing bmc-fw-v170 child

[T+3]  BMCVersion v1.70 Completed → driftPolicy: Suspend
         Run patches spec.driftPolicy: "" on existing bmc-fw-v180 child

[T+4]  BMCVersion v1.80 Completed
         → no later same-kind stage → Run patches driftPolicy: Observe
         Run patches spec.driftPolicy: "" on existing bmc-post-upgrade child

[T+5]  BMCSettings/bmc-post-upgrade Applied → driftPolicy: Observe
         Run re-hydrates spec.version: "U54 v2.10" (from server.status.biosVersion)
           and patches spec.driftPolicy: "" on bios-pre-upgrade children (× 3 servers)
           → BIOSSettings applies PxeBootToFirstInterface → Applied → driftPolicy: Observe

[T+6]  bios-pre-upgrade Completed (all servers)
         Run patches spec.driftPolicy: "" on existing bios-fw-v250 children (× 3)
           (v2.10 ≠ v2.50 → children re-execute → upgrade)

[T+7]  BIOSVersion v2.50 Completed per server → driftPolicy: Suspend
         Run patches spec.driftPolicy: "" on existing bios-fw-v250 children (× 3)

[T+8]  BIOSVersion v2.50 Completed per server → driftPolicy: Observe
         Run patches spec.driftPolicy: "" on existing bios-post-upgrade children (× 3)

[T+9]  BIOSSettings/bios-post-upgrade Applied on all servers → bios-post-upgrade Completed
         All stages Completed → Run phase: Completed
         DriftDetected condition cleared
```

Key observations:
- Each existing child object is reused — `spec.driftPolicy: ""` re-activates it in place. The child's condition history is preserved across recovery attempts.
- Intermediate version hops (`v1.60`, `v1.70`) are re-executed in sequence because the hardware must traverse every hop; the Suspend assignment only takes effect *after* the hop completes.
- The halting step at T+0 (patching `Observe` on any `InProgress` child) is a no-op in this scenario since all children were already `Completed` before drift was detected.

### Version hydration for version-agnostic stages

`BIOSSettings` and `BMCSettings` stages that apply settings independent of firmware version can omit `template.version`. The Run hydrates this field at child-stamp time from live hardware state (`server.status.biosVersion` / `bmc.status.firmwareVersion`). On drift recovery, the Run re-reads the current version and patches `spec.version` on the existing child before re-activating it.

## API changes and migration

This section describes the breaking and non-breaking changes to the existing child resource APIs introduced by this proposal, and the recommended migration path for operators with existing standalone resources.

### BMCSettings — settingsFlow replaces flat settings map

**Change.** The flat `spec.settings` map (`SettingsMap map[string]string`, JSON key `settings`) on `BMCSettings` is replaced by the `spec.settingsFlow` array, which groups settings into named, priority-ordered items (`SettingsFlowItem`). `BIOSSettings` already used `settingsFlow` before this proposal; no structural change there.

| Field | Old spec | New spec |
|---|---|---|
| `spec.settings` | `map[string]string` — flat key/value map | **Removed** (deprecated in v1, removed in v2) |
| `spec.settingsFlow` | — | `[]SettingsFlowItem` — named, priority-ordered groups |
||||

**v1 — both fields present, `settings` deprecated:**

```yaml
# OLD — still accepted in v1 but triggers a deprecation warning
spec:
  version: "iLO 6 v1.80"
  settings:
    SyslogServer: "10.0.0.5"
    SNMPv3AuthProtocol: SHA256
```

```yaml
# NEW — required in v2, recommended in v1
spec:
  version: "iLO 6 v1.80"
  settingsFlow:
    - name: network-baseline
      priority: 1
      settings:
        SyslogServer: "10.0.0.5"
        SNMPv3AuthProtocol: SHA256
```

**Migration strategy:**

- **v1 (this release):** Both `settings` (deprecated) and `settingsFlow` are accepted. If only `settings` is set, the controller wraps it internally as a single `settingsFlow` item with `priority: 1`. If both are set, `settingsFlow` takes precedence and `settings` is ignored. An admission webhook records a `deprecated-settings-map` event to help operators discover resources that need updating.
- **v2 (next release):** The `settings` field is removed from the CRD schema. Any resource still using `settings` will fail validation. A migration tool (`kubectl metal migrate bmcsettings`) will rewrite all existing objects to `settingsFlow` in place before the upgrade.


### driftPolicy — new field on BMC/BIOS Settings/Version

`spec.driftPolicy` is a new **optional** field added to `BMCSettingsSpec`, `BIOSSettingsSpec` etc. It defaults to `""` (empty string), which preserves the existing behaviour: the controller is fully active, reconciles hardware, and requests maintenance windows.

No migration is required. Existing standalone resources without this field behave exactly as before. The field is only patched by the `MaintenancePipelineRun` controller — operators must not set it manually on resources they manage outside a pipeline, as doing so would bypass the pipeline's ordering and drift-recovery logic.

### BMC and Server settings refs — single ref becomes a list

**What changes.** The current API tracks which settings object is active via a single back-reference:

- `BMC.spec.bmcSettingsRef` (`*LocalObjectReference`) — set by the `BMCSettings` controller to point to itself once it wins the ownership race for a given BMC.
- `Server.spec.biosSettingsRef` (`*LocalObjectReference`) — set by the `BIOSSettings` controller similarly for a given Server.

Today only one `BMCSettings` may reference a given BMC at a time, enforced by the admission webhook. A `MaintenancePipelineRun` creates **one `BMCSettings` per pipeline stage** for the same BMC — `bmc-baseline`, `bmc-post-upgrade`, etc. — so multiple objects for the same BMC will exist simultaneously. The single ref cannot track all of them and the uniqueness constraint blocks their creation entirely.

The fix mirrors the `settingsFlow` change: keep the old field as deprecated in v1, introduce the list alongside it, remove the old field in v2.


## Alternatives

### One pipeline object per BMC (no auto-generated Run)

The operator authors one `MaintenancePipeline` per BMC and the controller creates the child resources directly without a Run intermediary. This was rejected because it pushes the 1:N fan-out problem back onto the operator and removes the fleet-level `maxConcurrent` control.

### Inline version hops (array of versions per stage)

A single `BMCVersion` stage could accept a list of version hops and sequence them internally. This was rejected because it hides stage-level progress from the status and makes drift recovery significantly more complex — the `driftPolicyFor` rule relies on stage identity, which is ambiguous when multiple hops are collapsed into one stage.

### `ServerMaintenance` workflows in a pipeline context

When a pipeline is running, multiple child resources for the same Server can be active across adjacent stages — for example a `BIOSSettings` child completing and a `BIOSVersion` child starting immediately after. Each child independently creates and removes `ServerMaintenance` objects as it enters and exits the `InProgress` state. This means a Server may be put into and taken out of maintenance mode multiple times in quick succession within a single pipeline run, once per stage that requires it.

This is a known rough edge. The right fix is not in the pipeline controller but in the `ServerMaintenance` workflow itself — for example:

- A **maintenance window** mechanism that keeps a Server in maintenance mode for the duration of an entire pipeline run rather than per stage/controller.
- **Auto-approval** policies that allow `ServerMaintenance` requests to be granted immediately during a window, without waiting for an external approver on every stage transition.
- A **batch** or **coalescing** mode where consecutive maintenance requests from the same owner group are merged and gets one approval.

These improvements are out of scope for this proposal. 
