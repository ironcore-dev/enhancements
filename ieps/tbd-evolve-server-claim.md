---
title: Evolve ServerClaim into the Universal Boot Primitive

iep-number: tbd

creation-date: 2026-04-27

status: draft

authors:

- "@maxmoehl"
- "@afritzler"

reviewers:

- "@main-reviewer-1"
- "@main-reviewer-2"

---

# IEP-NNNN: Evolve ServerClaim into the Universal Boot Primitive

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
* `metal.ironcore.dev/needs-maintenance`
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

#### BootMethod

New. In addition to the type, metal-operator will have a global flag to set the
default, the flag itself will default to `PXE`.

```go
type BootMethod string

const (
	BootMethodHTTP = "HTTP"
	BootMethodPXE = "PXE"
)
```

#### ServerClaimSpec

Add:
* `Tolerations`
* `BootMethod`
* `Priority`


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

	// BootMethod configures via which method the server is booted.
	BootMethod BootMethod `json:"bootMethod,omitempty"`

	// Priority can be any value from -100 to 100. The default is 0. A higher
	// priority claim gets preferred when multiple claims try to claim the same
	// server. A claim referencing a specific server will always win over a
	// claim with a selector.
	// +kubebuilder:validation:Minimum=-100
	// +kubebuilder:validation:Maximum=100
	// +kubebuilder:default=0
	// +optional
	Priority int `json:"priority,omitempty"`
}
```

#### ServerClaimStatus

Add:
* `ImageURL`

```go
type ServerClaimStatus struct {
	// Phase represents the current phase of the server claim.
	// +optional
	Phase Phase `json:"phase,omitempty"`

	// ImageURL is the digest-pinned OCI image reference resolved from the
	// user-provided image. It contains the necessary artifacts for the
	// specified boot method as layers in the IronCore format.
	ImageURL string `json:"imageURL,omitempty"`
}
```

#### Phase

Add:
* `Initial`
* `Ready`
* `Invalid`
* `Evicted`

Remove:
* `Unbound`

```go
type Phase string

const (
	// PhaseInitial indicates that the server claim was just created and is not
	// yet ready to be bound.
	PhaseInitial Phase = "Initial"
	// PhaseReady indicates that the server claim is ready to be bound to a
	// server.
	PhaseReady Phase = "Ready"
	// PhaseInvalid indicates that the server claim failed validation checks and
	// cannot be bound to a server.
	PhaseInvalid Phase = "Invalid"
	// PhaseBound indicates that the server claim is bound to a server.
	PhaseBound Phase = "Bound"
	// PhaseEvicted indicates the server was forcefully evicted and cannot be
	// re-bound.
	PhaseEvicted Phase = "Evicted"
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
This is done by metal-operator as soon as the taint is picked up by removing the
claim reference on the server, setting it back to available, and putting the
claim into the `Evicted` phase with an event that it was evicted by a taint. It
is the responsibility of the claim owner to react to such an event and re-create
the claim as necessary. The `Evict` taint implies the `NoBind` effect.

### Claim Flow

To claim a server, a `ServerClaim` must be created. The claim contains a URL to
an IronCore compliant OCI image or manifest. A new claim starts out in the
`Initial` phase, when it is first picked up by metal-operator it verifies the
information provided by the creator is correct.

To do so it resolves the provided URL and resolves any manifests to ensure that
it contains the necessary artifacts for the given boot method. Once it gathered
this information it places the URL of the specific OCI image into the status
`ImageURL` field using the digest of the image to record what the tag resolved
to when it was checked and ensure the boot process uses the same image. The
claim transitions to the `Ready` phase and can now be bound. If the image does
not contain the necessary artifacts, e.g. not an IronCore image or only contains
PXE artifacts, but claim specifies HTTP, it enters the `Invalid` phase.

To bind a claim to a server, metal-operator checks the taints and tolerations to
select suitable servers for a claim. Once a match is found the claim is bound to
the server and the server transitions to the `Reserved` state while the claim
becomes `Bound`.

Depending on the boot method given in the claim metal-operator configures the
server and turns it on. The boot procedure for PXE remains as it is today. For
HTTP booting the metal-operator sets PXE boot as well but the DHCP server must
be configured to only reply to PXE or HTTP boot requests depending on how the
claim is configured.

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

To schedule a maintenance, the `metal.ironcore.dev/needs-maintenance` taint is
added to a server and a maintenance object referencing the server is created.
Depending on the effect of the taint, the maintenance is either executed
gracefully or forcefully, see [taint effects](#taint-effects) for details. In
the graceful case the information about this taint is propagated to the workload
manager which has to react to the taint. The workload is migrated off the server
and eventually the claim is released allowing the server to transition to the
`Maintenance` state and the maintenance to begin. If, as part of a maintenance,
the server must be booted a claim with the appropriate toleration must be
created with the necessary information.

### Boot-Operator

With the core boot logic moved to metal-operator, boot-operator retains the
functionality to render ignitions from server claims. Though, with the metaldata
proposal we are also transitioning away from that towards a generic user-data
construct. The remaining image proxy logic could then be carved out and retained
as a separate component with boot-operator becoming deprecated.

### Migration Path

#### 1. Adjust Workload Managers

Components which manage server claims need to be adjusted to support a claim
which has been forcefully evicted. They should consider an evicted claim as
something that needs to be cleaned up and, if it is still needed, should be
re-created.

Any server which they have a claim bound to should be monitored for taints. If a
taint appears the workload manager should make an effort to unbind the claim
unless the claim explicitly tolerates the taint.

#### 2. Add New API Components and Deprecate Old Fields

* Add taints & tolerations.
* New claim phases.
* Add boot method and priority to claim spec.
* Add image URL field to claim status.

Together with the API changes metal-operator will start to honor taints and
priority in addition to the existing logic for maintenances. With the addition
of the new claim phases metal-operator will also start validating the provided
OCI URL to ensure it's valid.

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
