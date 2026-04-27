---
title: Evolve ServerClaim into the Universal Boot Primitive

iep-number: 20

creation-date: 2026-04-27

status: draft

authors:

- "@maxmoehl"
- "@afritzler"

reviewers:

- "@main-reviewer-1"
- "@main-reviewer-2"

---

# IEP-20: Evolve ServerClaim into the Universal Boot Primitive

## Table of Contents

- [Summary](#summary)
- [Motivation](#motivation)
    - [Goals](#goals)
    - [Non-Goals](#non-goals)
- [Proposal](#proposal)

## Summary

Collapse the many different types that make up the metal-automation into two
core types: `Server` and `ServerClaim`.

## Motivation

The metal automation has undergone a period of organic growth. This has resulted
in an API that reflects that growth which could be simplified if it would've
been designed from the ground up with what we know today. This proposal aims to
describe that API while keeping in mind the backward compatibility with the API
we have today to enable an evolution of it.

### Goals

* Simplify the API surface and operational logic of the metal automation.
* Reduce the number of kinds / components needed.

### Non-Goals

* Create a new metal automation that is not backwards compatible.

## Proposal

### API Changes

This section lists all additions, removals and changes of API types. Any other
API types remain as-is.

Delete types:
* `ServerBootConfiguration`
* `HTTPBootConfig`
* `IPXEBootConfig`

#### Taint & Toleration

New. These mirror the taints and tolerations on nodes and pods. The effect
exists to tell metal-operator whether it should forcefully remove an existing
claim which does not tolerate a taint (`Evict`) or leave them alone until they
eventually drop the claim and just prevent other claims from binding the server
(`NoBind`).

```go
type TaintEffect string

const (
	TaintEffectNoBind = "NoBind"
	TaintEffectEvict = "Evict"
)

type Taint struct {
	Key   string `json:"key"`
	Value string `json:"value,omitempty"`
	Effect TaintEffect `json:"effect"`
}
```

```go
type TolerationOperator string

const (
	TolerationOperatorEqual = "Equal"
	TolerationOperatorExists = "Exists"
)

type Toleration struct {
	Key      string             `json:"key"`
	Operator TolerationOperator `json:"operator,omitempty"`
	Value    string             `json:"value,omitempty"`
	Effect   TaintEffect        `json:"effect,omitempty"`
}
```

Well known taint keys are:
* `metal.ironcore.dev/maintenance`
* `metal.ironcore.dev/initial`

#### Server

Add:
* `Taints`

Remove:
* `BootConfigurationRef`
* `MaintenanceBootConfigurationRef`

```go
type ServerSpec struct {
	// SystemUUID is the unique identifier for the server.
	// +required
	SystemUUID string `json:"systemUUID"`

	// SystemURI is the unique URI for the server resource in REDFISH API.
	SystemURI string `json:"systemURI,omitempty"`

	// Power specifies the desired power state of the server.
	// +optional
	Power Power `json:"power,omitempty"`

	// IndicatorLED specifies the desired state of the server's indicator LED.
	// +optional
	IndicatorLED IndicatorLED `json:"indicatorLED,omitempty"`

	// ServerClaimRef is a reference to a ServerClaim object that claims this server.
	// +kubebuilder:validation:Optional
	// +optional
	ServerClaimRef *ImmutableObjectReference `json:"serverClaimRef,omitempty"`

	// ServerMaintenanceRef is a reference to a ServerMaintenance object that maintains this server.
	// +optional
	ServerMaintenanceRef *ObjectReference `json:"serverMaintenanceRef,omitempty"`

	// BMCRef is a reference to the BMC object associated with this server.
	// +optional
	BMCRef *v1.LocalObjectReference `json:"bmcRef,omitempty"`

	// BMC contains the access details for the BMC.
	// +optional
	BMC *BMCAccess `json:"bmc,omitempty"`

	// BootOrder specifies the boot order of the server.
	// +optional
	BootOrder []BootOrder `json:"bootOrder,omitempty"`

	// BIOSSettingsRef is a reference to a biossettings object that specifies
	// the BIOS configuration for this server.
	// +optional
	BIOSSettingsRef *v1.LocalObjectReference `json:"biosSettingsRef,omitempty"`

	// Taints control which claim can be bound to a server based on whether it
	// has the corresponding tolerations.
	Taints []Taint `json:"taints,omitempty"`
}
```

#### ServerClaimSpec

Add:
* `Tolerations`


```go
type ServerClaimSpec struct {
	// Power specifies the desired power state of the server.
	// +required
	Power Power `json:"power"`

	// ServerRef is a reference to a specific server to be claimed.
	// +kubebuilder:validation:Optional
	// +kubebuilder:validation:XValidation:rule="self == oldSelf",message="serverRef is immutable"
	// +optional
	ServerRef *v1.LocalObjectReference `json:"serverRef,omitempty"`

	// ServerSelector specifies a label selector to identify the server to be claimed.
	// +kubebuilder:validation:Optional
	// +kubebuilder:validation:XValidation:rule="self == oldSelf",message="serverSelector is immutable"
	// +optional
	ServerSelector *metav1.LabelSelector `json:"serverSelector,omitempty"`

	// IgnitionSecretRef is a reference to the Secret object that contains
	// the ignition configuration for the server.
	// +optional
	IgnitionSecretRef *v1.LocalObjectReference `json:"ignitionSecretRef,omitempty"`

	// Image specifies the boot image to be used for the server.
	// +required
	Image string `json:"image"`

	// Tolerations control whether this claim can be bound to a server that is
	// tainted in a certain way.
	Tolerations []Toleration `json:"tolerations,omitempty"`
}
```

#### ServerClaimStatus

Add:
* `Conditions`

```go
type ServerClaimStatus struct {
	// Phase represents the current phase of the server claim.
	// +optional
	Phase Phase `json:"phase,omitempty"`

	// Conditions report the current observations of the claim, such as
	// failures encountered while pulling or booting the configured image.
	// +optional
	Conditions []metav1.Condition `json:"conditions,omitempty"`
}
```

#### Phase

Modeled after the pod lifecycle: a claim is either `Pending` (created but not
yet bound to a server) or `Bound` (bound to a server). All other observations
about the claim, including failures, are surfaced via conditions.

Remove:
* `Unbound`

```go
type Phase string

const (
	// PhasePending indicates that the server claim has been created but is
	// not yet bound to a server.
	PhasePending Phase = "Pending"
	// PhaseBound indicates that the server claim is bound to a server.
	PhaseBound Phase = "Bound"
)
```

### Taint Effects

There are two distinct effects a taint can have: `NoBind` and `Evict`.

When a server is tainted with `NoBind` no new claims can be bound to that server
unless they have a toleration for the taint that is causing this effect.
Existing claims which are bound to a server remain even if they don't have a
toleration. The expectation is that the manager of that claim prepares to
release the server eventually.

The `Evict` effect forcefully removes any bound claim that does not tolerate it.
This is done by metal-operator as soon as the taint is picked up by deleting
the offending claim and clearing the claim reference on the server, returning
it to available. It is the responsibility of the claim owner to observe the
deletion and re-create the claim as necessary. The `Evict` taint implies the
`NoBind` effect.

### Claim Flow

To claim a server, a `ServerClaim` must be created. The claim contains a URL to
an IronCore compliant OCI image or manifest. A new claim starts out in the
`Pending` phase.

To bind a claim to a server, metal-operator checks the taints and tolerations
to select suitable servers for a claim. Once a match is found the claim is
bound to the server, the server transitions to the `Reserved` state, and the
claim moves to `Bound`. metal-operator then configures the server and turns it
on.

The image is not validated up-front; the actual pulling and verification
happens later as part of the boot process, similar to how a kubelet pulls a
container image only when starting a pod. If the pull fails — for example
because the image cannot be retrieved or is not in the expected format — the
component responsible for the pull surfaces this on the claim via a condition.
The exact mechanism for reporting such failures is out of scope for this
proposal.

### Onboarding Flow

When onboarding a new server, it is created with its state set to `Initial`.
When metal-operator first picks up the server it adds the
`metal.ironcore.dev/initial` taint with the `NoBind` effect to prevent claims
from getting bound to this server. It also creates a claim which references that
server and has a toleration for the initial taint to boot the metal probe and
collect the server metadata, the server state is set to `Discovery`. Once the
server metadata has been reported the server is turned off again by removing the
claim.  Once the server is off again the initial taint is removed and the state
is set to `Available` allowing the server to be claimed by any claim.

### Maintenance Flow

In addition to the existing maintenance flow where a maintenance gets bound to a
server object, the maintenance controller may choose to assign a taint to the
server to evict any current claim or prevent new claims from binding to the
server. The `metal.ironcore.dev/maintenance` taint key is well-known for this
kind of taint. Once the maintenance is done, the maintenance controller clears
any taints it previously set and the server can be used as before.

### Boot-Operator

With the core boot logic moved to metal-operator, boot-operator retains the
functionality to render ignitions from server claims. Though, with the metaldata
proposal we are also transitioning away from that towards a generic user-data
construct. The remaining image proxy logic could then be carved out and retained
as a separate component with boot-operator becoming deprecated.

### Migration Path

#### 1. Adjust Workload Managers

Components which manage server claims need to be adjusted to support a claim
which has been forcefully evicted. Since eviction deletes the claim, workload
managers should treat the disappearance of a claim they own as a signal that
the claim was evicted and, if it is still needed, recreate it.

Any server which they have a claim bound to should be monitored for taints. If a
taint appears the workload manager should make an effort to unbind the claim
unless the claim explicitly tolerates the taint.

#### 2. Add New API Components and Deprecate Old Fields

* Add taints & tolerations.
* Replace existing claim phases with `Pending` / `Bound`.
* Add conditions to claim status.

Together with the API changes metal-operator will start to honor taints in
addition to the existing logic for maintenances.

#### 3. Migrate Discovery Flow

Metal-operator will be adjusted to no longer rely on internal handling via
server boot configurations to instead use the initial taint and a claim to
perform the discovery boot.

#### 4. Migrate Maintenance Flow

Instead of relying on the approval flow based on labels / annotations the
maintenance controller should wait for the claim to be removed from the server
before starting the maintenance instead.

#### 5. Remove Deprecated Fields

Once the migration is done, metal-operator will stop creating boot
configurations which will stop most of the boot-operator automation.
Maintenances will also no longer be executed from their references. The
deprecated references are removed from the server spec:

* `BootConfigurationRef`
* `MaintenanceBootConfigurationRef`

#### 6. Remove Deprecated Types

Finally, the now unused types are deleted:

* `ServerBootConfiguration`
* `HTTPBootConfig`
* `IPXEBootConfig`
