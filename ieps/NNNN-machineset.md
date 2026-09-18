---
title: MachineSet

iep-number: NNNN

creation-date: 2026-09-17

status: implementable

authors:

- "@shawnsarwar"

reviewers: []

---

# IEP-NNNN: MachineSet

## Table of Contents

- [Summary](#summary)
- [Motivation](#motivation)
    - [Goals](#goals)
    - [Non-Goals](#non-goals)
- [Proposal](#proposal)
    - [API resources and controllers](#api-resources-and-controllers)
    - [MachineSet API](#machineset-api)
    - [MachineSetMember API](#machinesetmember-api)
    - [Example](#example)
    - [Resource identity and ownership](#resource-identity-and-ownership)
    - [Placement compatibility](#placement-compatibility)
    - [Provisioning and power state](#provisioning-and-power-state)
    - [Compute rollout](#compute-rollout)
    - [Scale-down and deletion](#scale-down-and-deletion)
    - [Volume updates and encryption](#volume-updates-and-encryption)
    - [Status and discovery](#status-and-discovery)
- [Alternatives](#alternatives)

## Summary

Introduce `MachineSet` to maintain a requested number of durable VM members from
one shared template. Each member retains its identity and assigned storage and
network resources while its underlying `Machine` can be replaced.

Users configure `MachineSet`, which coordinates membership and updates.
A controller-created `MachineSetMember` preserves each member's resource
associations and progress when its Machine is absent, including when scale-down
retains its resources for later use. Deleting the set releases its owned members
and resources. Members are read-only to workload consumers.

We propose an optional upstream IronCore controller component with associated
API resources. Packaging and integration decisions are left to maintainers.

## Motivation

IronCore's `Machine` is a replaceable execution resource. Existing consumers,
such as Gardener, provide their own lifecycle orchestration.
[IEP-21](https://github.com/ironcore-dev/enhancements/blob/main/ieps/21-machine-eviction.md)
defines eviction through Machine deletion and leaves recreation to an external
owner. Direct API consumers currently need to manage this lifecycle themselves.

A long-running VM needs continuity beyond the lifetime of an individual Machine.
When that Machine is replaced, the workload should retain its root and data
disks, network identity, and declared configuration.

Users also may need to change a VM's compute size, for example to increase its CPU
or memory allocation. In IronCore, this means selecting another supported
MachineClass. Because a Machine's class reference is immutable, applying that
change requires a replacement Machine that reuses the workload's existing disks
and network identity.

[Issue #65](https://github.com/ironcore-dev/enhancements/issues/65) discusses this
ownership boundary and a MachineSet abstraction. This proposal covers both a set containing a singleton VM with a persistent root disk and a homogeneous group, such as three broker VMs with the same provisioning configuration but independent disks and
addresses. A heterogeneous application would use multiple MachineSets;
coordinating those application components is outside this proposal.

### Goals

- Maintain a requested number of VM workloads whose configuration and resource
  associations survive Machine replacement.
- Provision independent disks and network interfaces for each member from one
  shared template.
- Allow CPU/RAM sizing changes through sequential Machine replacement, retaining
  the assigned disks and network identity.
- Avoid allocating ancillary resources for requested VMs while compute capacity is
  unavailable, and check suitable capacity before starting planned replacement.
- Make resource retention and deletion explicit during scale-down and when
  deleting MachineSet.
- Expose each member's current Machine, assigned resources, network addresses,
  and lifecycle progress through the API.
- Preserve access to encrypted Volumes across Machine replacement.
- Define the MachineSet API for future grow-only Volume updates.

### Non-Goals

- Live migration, CPU/memory hotplug, or preservation of the Machine UID.
- Snapshot management, backup/restore, reimaging, disk shrink, or automatic rollback.
- Application-aware preparation, quiescence, or health checks.
- Automatic replacement triggered by an unreachable-pool timeout or application
  health; host fencing and recovery from storage loss.
- Reserving compute capacity.
- Per-member configuration or heterogeneous sets.
- Storage or network migration to make an incompatible compute target usable.
- Owning encryption-key lifecycle management, including rotation, retention,
  and destruction.
- Taking over responsibility for Machines already managed by Gardener or other controllers.

## Proposal

### API resources and controllers

Introduce namespaced `MachineSet` and `MachineSetMember` API resources, using
`compute.ironcore.dev/v1alpha1` as the proposed group and version. This IEP
defines their API contract and controller responsibilities. API registration, serving, packaging, and integration decisions
are left open.

MachineSet manages VM lifecycles through IronCore's public resource APIs.
Provider-specific operations remain with the existing IronCore controllers,
poollets, and providers.

| Component | Responsibility |
| --- | --- |
| MachineSet controller | Membership, shared configuration, provisioning window, rollout order, and aggregate status. |
| MachineSetMember controller | Durable resource bindings, current execution, operation progress, and safe handoff. |
| Existing IronCore controllers and providers | Placement, provisioning, power reconciliation, attachment, and provider cleanup. |

The set selects the work; the member reconciles one logical VM and its retained
resources. The two reconcilers share the same controller component.
Field names and examples below are proposed API shapes, not final implementation
structures.

### MachineSet API

| Field | Meaning |
| --- | --- |
| `spec.replicas` | Desired member count; non-negative, default `1`. Powered-off members still count. |
| `spec.template` | Shared Machine template. Supported changes reconcile without another approval step; placement, tolerations, the bootstrap reference, and attachment layout are immutable as specified below. |
| `spec.volumeTemplates` | Named templates for standalone per-member Volumes. |
| `spec.networkInterfaceTemplates` | Named NIC templates instantiated separately for each member |
| `spec.resourceRetentionPolicy.whenScaledDown` | `Retain` (default) or `Delete` for members outside the desired range. |
| `spec.volumeUpdatePolicy` | `OnCreate` (default) or `ExpandExisting`; the latter is defined here but initially unsupported. |

The Machine template reuses the applicable Machine fields, including
`machineClassRef`, `power`, placement selectors, tolerations, and attachments.
For attachments, it additionally accepts `volumeTemplateRef` and
`networkInterfaceTemplateRef`. The controller resolves these into concrete
`volumeRef` and `networkInterfaceRef` values on each child Machine.

Template and attachment names are unique within their lists. Named templates
use MachineSet-specific validation: `metadata.name` identifies the template,
not a child resource. Each attachment selects exactly one source. Unknown
template references, controller-owned claim/binding fields, and conflicting
member identities are rejected.

Unsupported set sizes are rejected before provisioning; limits must not silently
omit member references or discard retained resources.

The following MachineSet fields have defined mutability and reconciliation behaviour
within the controller:

| Field | Mutability | Controller behaviour |
| --- | --- | --- |
| `spec.replicas` | Mutable | Provision or deactivate members; apply the scale-down retention policy. |
| `spec.template.spec.machineClassRef` | Mutable selection | Changing the class name requests a rollout; missing or replaced classes block dependent work. |
| `spec.template.spec.power` | Mutable | Change the existing Machines' desired power without replacing them. |
| `spec.resourceRetentionPolicy.whenScaledDown` | Mutable | `Delete` reclaims owned resources of unrequested members, including previously retained ones. `Retain` releases only compute |
| `spec.volumeTemplates[].spec.resources.storage` | Policy-dependent | Under `OnCreate`, changes affect new Volumes only. The deferred `ExpandExisting` policy grows assigned managed Volumes as described below. |
| `spec.volumeUpdatePolicy` | Staged support | Initially only `OnCreate` is supported; requests for `ExpandExisting` are rejected until expansion is implemented. |
| `spec.template.spec.machinePoolSelector`, `spec.template.spec.machinePoolRef` | Immutable | Preserve the user-declared compute placement constraints. |
| `spec.template.spec.tolerations` | Immutable | Preserve the declared compute-pool tolerations. |
| `spec.volumeTemplates[].spec.volumePoolSelector`, `spec.volumeTemplates[].spec.volumePoolRef`, `spec.volumeTemplates[].spec.tolerations` | Immutable | Preserve the declared storage placement constraints and tolerations. |
| `spec.networkInterfaceTemplates[].spec.networkRef` | Immutable | Preserve the selected Network. |
| `spec.template.spec.ignitionRef` | Immutable reference | The Secret reference and selected key are fixed. New Machines consume the externally managed content available when provisioning is prepared. |
| `spec.volumeTemplates[].spec.dataSource.osImage.image` | Immutable reference | Used to initialize new root Volumes. The content behind the reference may evolve externally. |
| `spec.template.spec.volumes`, `spec.template.spec.networkInterfaces` | Immutable | Preserve attachment names, layout, and source references; reject additions, removals, renaming, or retargeting. |

Immutability starts when MachineSet is created, including at zero replicas.
Supported template fields, source forms, and update behaviour must be explicitly
defined; underlying schema changes do not imply support. Controller-owned
claim/binding fields are rejected, but user-declared pool references remain
valid immutable placement inputs.

A singleton may reference externally owned Volumes and NetworkInterfaces.
Fixed exclusive inputs, including nested IP/prefix values, fixed Prefix
allocations, and VirtualIP references, are rejected when `replicas` exceeds
one, including on later scale-up.

Each member receives its own Volumes and NetworkInterfaces from the templates.
Members can share the Network, allocation-parent Prefix, classes, image, and Secrets.

Durable roots use standalone Volumes, not Machine-local or Machine-owned
ephemeral disks. The image reference is fixed, but its publisher may update the
content behind it. New root Volumes use the image resolved by the provider when
they are created.

Image-content changes do not trigger a rollout, and replacement Machines reuse
existing roots without reimaging them. MachineSet neither retains an image
snapshot nor requires a digest-pinned image.

The bootstrap reference stays fixed, but the Secret's owner may update its
content, for example to rotate credentials. Each new Machine, including a
replacement, uses the content available when provisioning is prepared.
MachineSet keeps no private copy and does not trigger a rollout or reconfigure
an existing guest when that content changes.

Missing or unreadable input blocks the dependent provisioning or planned
replacement operation before an existing Machine is stopped. Supplying bootstrap
input does not guarantee that the guest reruns first-boot logic on a retained
root disk.

### MachineSetMember API

A member occupies a zero-based position, or ordinal, within its originating
MachineSet, identified by that set's UID. An ordinal can be reused after a member
is permanently reclaimed. The member's own UID distinguishes a retained member
from a new member later created at that same position.

| Field | Meaning |
| --- | --- |
| `spec.machineSetRef` | Immutable originating MachineSet name and UID, in the same namespace. |
| `spec.ordinal` | Immutable logical position. |
| `spec.lifecycle` | Controller-owned intent: `Active`, `Retained`, or `Reclaiming`; see [Scale-down and deletion](#scale-down-and-deletion). |
| `spec.targetRevision` | Identity of the applicable execution configuration selected for this member. |
| `spec.template` | Controller-derived snapshot of the selected execution/resource templates, not a user override. |
| `status.appliedRevision` | Configuration whose execution has satisfied infrastructure readiness. |
| `status.machineRef` | Current Machine name and UID, when present. |
| `status.volumes`, `status.networkInterfaces` | Named, UID-bound resource associations and observed state. |
| `status.placement` | Observed compatibility restriction within immutable user placement constraints; does not itself enforce scheduling. |
| `status.operation` | Operation identity, target revision, phase, and predecessor identity across replacement. |
| `status.conditions` | Readiness, progress, and blocking information. |

A controller restart or failed API/provider call can interrupt replacement
after the old Machine has gone. The member therefore keeps the selected
configuration, predecessor identity, and operation progress independently of
that Machine, so reconciliation can resume without losing the intended change.

The set controller selects and records each operation before the member acts.
The member's status reports progress; it does not independently authorize
disruption. One template edit must not start all member replacements at once.

Workload consumers can get, list, and watch members. Creation and mutation of
member configuration, association labels, lifecycle state, and status are
reserved for the controllers through RBAC/admission. Unsupported mutations
must not be accepted and silently overwritten.

### Example

This set provisions three independent root Volumes and interfaces. Additional
named Volume templates provide data disks through the same attachment mechanism.
Class, image, Network, and parent Prefix names are illustrative existing inputs.

```yaml
apiVersion: compute.ironcore.dev/v1alpha1
kind: MachineSet
metadata:
  name: brokers
spec:
  replicas: 3
  template:
    spec:
      machineClassRef:
        name: broker-medium
      power: "On"
      volumes:
        - name: root
          volumeTemplateRef:
            name: root
      networkInterfaces:
        - name: primary
          networkInterfaceTemplateRef:
            name: primary
  volumeTemplates:
    - metadata:
        name: root
      spec:
        volumeClassRef:
          name: durable-block
        resources:
          storage: 40Gi
        dataSource:
          osImage:
            image: registry.example.org/images/broker:v1
  networkInterfaceTemplates:
    - metadata:
        name: primary
      spec:
        networkRef:
          name: application-network
        ipFamilies:
          - IPv4
        ips:
          - ephemeral:
              prefixTemplate:
                spec:
                  ipFamily: IPv4
                  prefixLength: 32
                  parentRef:
                    name: application-prefix
  resourceRetentionPolicy:
    whenScaledDown: Retain
  volumeUpdatePolicy: OnCreate
```

### Resource identity and ownership

The member records resource names and UIDs, their attachment roles, and whether
they were created for the member or supplied externally. Generated resources
carry protected associations with their member UID. Retrying a create must
recover the same resource, not allocate another bundle.

Resources belonging to a different owner must not be adopted. Recreating a
resource or a MachineSet with the same name does not transfer the old binding
to the new UID. A missing or conflicting binding is reported rather than
silently replaced with empty storage. Bound Volume and NetworkInterface UIDs
must be enforced at consumption, including for external resources; name-only
child references and a prior UID check are insufficient.

MachineSet manages resource bindings, lifecycle, retention, and the selected
capacity policy. It does not otherwise continually reset existing Volumes or
NetworkInterfaces to their provisioning templates. Their properties remain
governed by the underlying APIs and controllers, and MachineSet reports their
current state.

Generated durable resources may be owned by the member, but not by the
replaceable Machine. Members belong to the set's lifecycle: they can survive
Machine replacement or retained scale-down, but are cleaned up on set deletion.
Finalizers and lifecycle transitions must retire executions and release
attachments before deleting owned resources and member records. Direct consumer
deletion of a member is not a scale-down interface.

### Placement compatibility

The deployment establishes which compute pools can use each member's complete
storage and network bundle. These pools form the member's compatible target set.
MachineSet records that restriction and applies it together with the user's
immutable placement constraints during provisioning, replacement, and
reactivation. The actual successor must land within that set; unknown or
inconsistent membership blocks the start of planned replacement.

This uses deployment-maintained compatibility, not topology discovery or live
attachment probing. The integration must enforce compatible placement for
initial and retained resources; member status alone cannot do so. Unresolved,
stale, or empty target sets block the operation. Fresh readiness and capacity
observations are required.

A compatible set may contain several MachinePools and may span zones if the
retained resources are usable there. Neither a MachinePool nor a region/AZ label
universally establishes resource accessibility, failure-domain independence,
or data-residency compliance. Different members can have different compatible
subsets without different user-authored templates.

Lack of compatible capacity causes waiting, not migration or resource
substitution. External label or taint changes do not relax these constraints;
existing IronCore eviction remains independent.

### Provisioning and power state

Provision members in ascending ordinal order, with at most one unfinished
new-member resource bundle per set. Logical member records can exist before
allocation. With three of ten members fulfilled, only the fourth may hold
speculatively allocated disks/NICs; the remaining six stay queued.

Advance when the member reaches infrastructure readiness at its requested
power state. A blocked attempt keeps its provisioning slot across retries and
restarts; the controller must not bypass it by allocating another bundle.
The bound does not delete previously used resources. Reactivating a retained
member reuses its bindings. More aggressive provisioning is a future option.

`power: Off` means the member remains requested but powered off. It is not
scale-down and does not make its resources reclaimable. `power: On` delegates
convergence to the existing power-reconciliation path; guest shutdown does not
rewrite API intent. A power-only change updates the existing Machine rather
than replacing it. A compute-class change preserves desired power.

### Compute rollout

Changing `spec.template.spec.machineClassRef` requests a compute rollout.
Because a Machine's class reference is immutable, each affected execution is
replaced while its member and durable resources remain.

MachineSet remembers the selected MachineClass name and UID. Observed absence
or UID change blocks dependent provisioning and planned replacement, with a
persistent condition and Warning Event. Healthy Machines remain running.
Recovery requires selecting another `machineClassRef.name`; normal rollout
gates apply. Reapplying the same name does not authorize a recreated class.

Update one member at a time, highest ordinal first. If any requested member is
already unavailable, do not disrupt another member; recover the unavailable
members before advancing the rollout. This applies even if the unavailable
member is not next in ordinal order. Infrastructure readiness at the requested
power state, not application health, determines completion.

1. Record the target revision and survey the whole rollout's feasibility.
2. Require fresh positive capacity and compatibility for the next replacement.
3. Request predecessor shutdown/deletion through the existing lifecycle.
4. Confirm predecessor teardown and attachment/claim release; keep the Volumes, NetworkInterfaces, and their existing ownership.
5. Create the successor with the target class and retained resource references.
6. Observe infrastructure readiness, record the applied revision, and advance.

The initial survey reports unavailable members, outstanding replacements,
assessable capacity, constraints, and observation freshness. A shortfall or
incomplete assessment produces a persistent condition and Warning Event, but
need not block partial progress if the next-member gate passes.

Before every planned replacement, require one free eligible target while the
predecessor still exists. Check pool readiness, class/hardware support, placement,
taints/tolerations, committed and in-flight allocations, and retained-resource
compatibility. Unknown, stale, placeholder, or insufficient capacity does not
pass. Do not count capacity expected only after deleting the predecessor;
this can block a same-host resize. Reassess if intervening changes invalidate
the result, using shared placement/accounting logic rather than probe Machines.

Assessment is not reservation and is performed on a best effort basis: capacity can disappear afterward. Likewise,
source exclusion requires confirmed teardown, not merely a removed API object,
claim, or finalizer. No successor may use exclusive resources while the
predecessor's access remains uncertain.

| Situation | Behaviour |
| --- | --- |
| No eligible capacity before shutdown | Keep the predecessor; report blocked and reassess. |
| Failure after predecessor deletion | Preserve resources; recover the unavailable member before advancing. |
| Temporary dependency failure | Retry with capped backoff and react to dependency changes. |
| Unsuitable target configuration | Accept a corrected template; do not require the bad revision to become ready first. |
| Uncertain teardown or attachment release | Block successor activation; a configuration edit does not bypass exclusion. |

Conflicting actions on a member are serialized. Before shutdown starts, a newer
request can supersede the planned change. Once teardown starts, confirm
predecessor exclusion and attachment release before creating a successor using
the latest desired configuration. If an incompatible successor already exists,
retire it through the same sequence rather than requiring manual child deletion.

Do not create an execution for a member that is no longer requested or boot one
whose desired power is Off. Finish irreversible reclamation before provisioning
a fresh member at the same ordinal.

Retries use capped delays, dependency events, and periodic reassessment without
churning pending Machines. Desired intent does not expire; a stalled rollout may
resume days later or remain starved indefinitely. Report last actual progress,
not merely the latest retry. Reverting compute class requests another rollout,
not recovery of the old execution or disk contents.

External deletion, including IEP-21 eviction, and terminal execution failure
leave the member responsible for restoring execution under the same handoff
rules. Normal power-off is not terminal failure. An unreachable pool is reported
without timeout-triggered replacement. MachineSet does not wait for guest
acknowledgement or promise application consistency.

### Scale-down and deletion

Scale down highest ordinals first. The policy applies to members outside the
desired replica range, not to requested members that happen to be stopped,
pending, or undergoing replacement.

| Action | Result |
| --- | --- |
| Scale down with `Retain` | Remove executions after safe cleanup; retain members and assigned resources. |
| Scale up a retained ordinal | Reactivate that member using its resources and the current target configuration. |
| Scale down with `Delete` | Clean up executions, delete member-owned resources, then remove the member. |
| Change `Retain` to `Delete` | Also reclaim already-retained, unrequested members of this set. |
| Delete MachineSet | Retire all executions and delete every member and its owned resources, including previously retained members, regardless of scale-down policy. |

For example, reducing five members to three with `Retain` preserves ordinals
three and four. A later policy change to `Delete` reclaims their owned bundles
without another replica change or confirmation.

To release the whole compute set while preserving its durable resources, scale to zero with
`Retain`. Deleting the set instead requests permanent cleanup; there is no
retain-on-set-deletion policy in this proposal.

External resources and shared Networks, classes, images, and key Secrets are
not deleted by this policy. Deletion of a NIC's owned address-allocation
resources follows their existing lifecycle; shared infrastructure is untouched.
Nested ephemeral VIPs with `reclaimPolicy: Retain` are not NIC-owned;
define their lifecycle before accepting that source form.

Once MachineSet deletion begins, stop selecting new provisioning or successor
creation. Retire executions, establish attachment release, delete member-owned
resources, and then remove member records before allowing parent removal.
Previously scaled-down retained members are included in this cleanup. Externally
supplied resources are released but not deleted.

The deletion finalizer keeps the set present while cleanup is incomplete and
reports blocking reasons. Members must not restart executions during deletion.
Keep their identities and references available until their cleanup completes;
do not use unguarded garbage collection as a substitute for this ordering.

### Volume updates and encryption

**Volume expansion is designed here but deferred from the first implementation.**
Disk capacity is separate from MachineClass; compute replacement preserves
assigned disks and their sizes.

| Policy | Behaviour |
| --- | --- |
| `OnCreate` | Use the template's requested capacity when creating a Volume. Later capacity edits affect new Volumes only, not assigned ones. |
| `ExpandExisting` | Propagate increased template capacity to managed Volumes of requested members without replacing them. Initially reject this policy as unsupported. |

The future `ExpandExisting` behaviour is:

- Grow managed Volumes of requested members, including powered-off members.
  Evaluate inactive retained members on reactivation; external Volumes remain
  separately managed.
- Treat requested capacity as a minimum, retain larger existing disks, and
  reject decreases to an already requested expansion target. Grow the same
  Volume without migration or replacement.
- On explicit selection, evaluate current template capacities against assigned
  Volumes. Controller upgrades must neither change the default nor silently
  activate previously ignored requests.
- Returning to `OnCreate` stops forwarding new capacity edits, but submitted
  expansion requests continue and their progress remains visible.

Per-volume status records `requestedCapacity`, `observedCapacity` (backing size),
`attachmentCapacity`, and expansion conditions. Report backing and attachment
completion separately: absent attachment is not applicable yet; unobservable
capacity is unknown, including under `OnCreate`. Do not infer observed size from
the request. A later attachment must reflect the grown disk; guest filesystem
expansion remains external. Failed growth retains the disk, reports its stage,
and follows normal retry rules. Compute/storage edits are not an atomic rollback.

**Rotation compatibility is required; rotation ownership remains external.**
MachineSet uses the encryption configuration and status exposed by the Volume API
to support encrypted-volume replacement. API support needed for this
interoperability is in scope; responsibility for key lifecycle operations remains
with the Volume/storage layer and external key-management systems. This does not
introduce a separate MachineSet key-rotation API.

The operator or key-management system requests rotation and owns key-retention
policy; the Volume/poollet/provider layer rotates keys and resolves effective
attachment credentials. MachineSet preserves bindings, does not overwrite
rotation intent with provisioning defaults, and blocks handoff when safe
attachment cannot be established.

This depends on the shared attachment behaviour addressed by the
[Volume encryption-key rotation design](https://github.com/ironcore-dev/enhancements/pull/62).
A stale status reference alone is not sufficient. MachineSet does not copy raw
keys, destroy key Secrets, or use rotation as predecessor fencing.

### Status and discovery

MachineSet status contains aggregate observations and compact member references:

| Field | Meaning |
| --- | --- |
| `observedGeneration` | Set generation observed by its controller. |
| `readyReplicas`, `updatedReplicas`, `pendingReplicas` | Requested members meeting infrastructure readiness, target revision, or awaiting fulfillment, respectively. |
| `targetRevision`, `lastProgressTime` | Current configuration identity and last substantive progress. |
| `activeOperations` | Bounded provisioning/replacement operation identities, selected member references, and target revisions, owned by the set controller. |
| `conditions` | Readiness, rollout progress, capacity assessment, and blocking reasons. |
| `members` | Recorded members, including retained unrequested ones: ordinal plus Member name/UID; not duplicated resource inventories. |

`updatedReplicas` compares applied execution revisions with the set's current
target, independently of present readiness. Changing Volume-template capacity
under `OnCreate` does not make an existing execution outdated; resource bindings
and capacities are separate observations. No independent provisioning-revision
API is required.

Conditions such as `Ready`, `Progressing`, and `CapacityAvailable` use standard
fields and distinguish `False` from `Unknown`. Eight ready and four updated
members is a valid stalled-rollout state. Ready reflects observed infrastructure
health at the requested power state; target revision is reported separately.
It is not a copy of Machine Ready and does not establish application health.

A member status excerpt illustrates resource navigation:

```yaml
status:
  appliedRevision: revision-1
  machineRef:
    name: brokers-2-execution-1
    uid: "<machine-uid>"
  volumes:
    - name: root
      ownership: Managed
      volumeRef:
        name: brokers-2-root
        uid: "<volume-uid>"
  networkInterfaces:
    - name: primary
      ownership: Managed
      networkInterfaceRef:
        name: brokers-2-primary
        uid: "<interface-uid>"
      ips:
        - "10.0.0.23"
```

These example names are illustrative, not a name-derivation interface. Member
references survive Machine replacement. Include any associated VirtualIP
reference and observed address when present. References distinguish managed
from external resources and include namespace where required.

Clients can list a MachineSet's members directly using its UID and namespace,
without first retrieving the MachineSet. The result includes requested members
and members retained after scale-down, with their resource references and
observed state.

## Alternatives

### External orchestration only

Existing APIs allow consumers to implement resource retention and sequential
replacement themselves. This preserves the current responsibility boundary,
but repeats IronCore-specific lifecycle and failure handling in each consumer.
MachineSet offers a common lifecycle without changing existing consumers.

### StatefulSet-style bookkeeping without MachineSetMember

Kubernetes StatefulSet reconstructs logical slots from the set, Pods, PVCs, and
revision history. It demonstrates that ordinals, retained disks, and ordered
replacement do not require a separate member API.

Our retained bundle also records independent network resources, owned versus
external references, resource UIDs, and cross-execution progress. That inventory
must remain discoverable while a Machine is absent and after retained scale-down,
including scale-to-zero, until the member is reactivated or reclaimed.

Without a member API, equivalent retained inventory and progress tracking would
need to be maintained through set status and protected resource metadata.

We prefer one typed, controller-owned member record. This adds an API lifecycle,
RBAC, and garbage-collection obligations; it does not itself solve fencing,
rollback, or scalability. It is a representation trade-off, not a claim that
the StatefulSet pattern cannot be extended.

### Durable Machine identity

Keeping the Machine while changing its realization or placement would change
the existing immutable-placement and eviction model. A parent/member lifecycle
keeps Machine replaceable and leaves existing provider consumers unchanged.
