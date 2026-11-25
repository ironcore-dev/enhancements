---
title: Confidential VM Support

iep-number: 26

creation-date: 2026-11-24

status: implementable

authors:

- "@SamiraBriongos"
- "@byteocean"

reviewers:

- "@main-reviewer-1"
- "@main-reviewer-2"

---

# IEP-26: Confidential VM Support

## Table of Contents

- [Summary](#summary)
- [Motivation](#motivation)
    - [Goals](#goals)
    - [Non-Goals](#non-goals)
- [Proposal](#proposal)
- [Security Considerations](#security-considerations)
- [Testing Strategy](#testing-strategy)
- [Alternatives](#alternatives)

## Summary

A confidential VM protects its memory and execution state with a hardware 
trusted execution environment (TEE). This proposal adds confidential VMs and 
nonce-based attestation to the IronCore `Machine` API. 

Users request confidential execution with `spec.confidential: true`. The
provider applies the required TEE configuration during VM creation. It assumes
the underlying hardware supports confidential computing and that it is enabled
in the BIOS. If reconciliation cannot provide confidential execution, IronCore
returns an error instead of reconciling an ordinary VM.

The initial implementation and hardware validation use Intel TDX. The common
Machine API, provider contract, and attestation challenge construction also
support AMD SEV-SNP by design. AMD hardware validation remains future work
until suitable hosts are available.

Attestation is a key part of the solution, so it should be possible to request 
and retrieve evidence about the launch status of the confidential VM.  
In order to do so, a user sets a nonce on the `Machine`. The provider
retrieves raw evidence from a guest attestation agent over vsock, and 
the controller publishes the result and challenge metadata in
`Machine.status`. Supported guest images include the required guest tools preinstalled.

An independent verifier could then check the evidence and make trust and key-release
decisions based on that. 

## Motivation

Confidential computing protects guest memory and execution state from the
host, hypervisor, and infrastructure administrators. Users should be able to
request it like other Machine capabilities, without provider-specific manifest
fields. 

Memory protection alone does not establish trust. An independent verifier
also needs evidence describing the TEE and VM launch, bound to a fresh
challenge to prevent replay. Exposing it through the Machine API lets the
verifier evaluate the VM before trusting it or releasing data to it.

### Goals

- Allow users to request hardware-backed confidential execution with
  `spec.confidential: true`.
- Report an error when reconciliation cannot provide the requested
  confidential execution; never reconcile the Machine as a regular VM.
- Allow users to supply immutable guest initdata at VM creation and bind it to
  the confidential VM's launch measurements.
- Allow users to request fresh attestation evidence with a nonce on the same
  `Machine`.
- Publish the latest raw evidence, challenge metadata, and retrieval result in
  `Machine.status`.
- Support different TEE technologies through the same Machine API and status
  model.
- Preserve the behavior of existing, non-confidential Machines.

### Non-Goals

- Implementing an attestation verifier in IronCore.
- Defining trusted launch measurements, TCB policies, certificate trust stores,
  or workload authorization policies.
- Implementing a key broker, key-release protocol, secret delivery mechanism,
  or workload identity system based on attestation.
- Automatically encrypting VM disks or network traffic as a consequence of
  `spec.confidential: true`.
- Claiming continuous attestation or runtime trustworthiness from a single
  evidence response.
- Requiring all MachinePool implementations to support confidential VMs.
- Supporting live migration of confidential VMs in the initial implementation.
- Reconciling or refreshing attestation evidence periodically without a user
  request.
- Provisioning hosts, enabling TDX or SEV-SNP in firmware, or installing host
  software. IronCore metal automation can automate this preparation separately
  in the future.
- Building guest images or installing attestation tools when a VM launches.

## Proposal

### API Changes

Add optional confidential execution and attestation fields to `Machine.spec`.
Omitting them preserves existing Machine behavior:

```go
type MachineSpec struct {
    // Existing fields are omitted.

    // Confidential requests hardware-backed confidential execution.
    // This field is immutable after creation.
    // +optional
    Confidential bool `json:"confidential,omitempty"`

    // ConfidentialInitData contains guest initialization data.
    // []byte is represented as base64 in YAML and JSON.
    // It may only be set when Confidential is true.
    // This field is immutable after creation.
    // +optional
    ConfidentialInitData []byte `json:"confidentialInitData,omitempty"`

    // AttestationNonce is a fresh, randomly generated evidence challenge.
    // []byte is represented as base64 in YAML and JSON.
    // It may only be set when Confidential is true.
    // +optional
    AttestationNonce []byte `json:"attestationNonce,omitempty"`
}
```

`spec.confidential` and `spec.confidentialInitData` are immutable. Changing the
boolean cannot convert an existing VM, and changing initdata cannot change the
launch identity of a running VM. API validation must reject changes to either
field.

Initdata lets the user supply guest initialization configuration that must be
bound to the launch measurements. The provider passes the supplied bytes to
its confidential launch stack, which defines their encoding and how they are
measured. API validation enforces a size limit so it is compatible with the 
different technologies.
A provider can supply a default when the field is omitted, but it must reject
initdata it cannot measure.

`spec.attestationNonce` is mutable. Setting a new value requests fresh
evidence without recreating, rebooting, or changing the VM's launch
configuration. When set, the value must be a cryptographically random nonce
of exactly 32 bytes.

Add observed confidential-computing and attestation information to
`Machine.status`:

```go
type MachineStatus struct {
    // Existing fields are omitted.

    // Confidential reports the technology observed by the provider.
    // It is absent until confidential execution has been established.
    // +optional
    Confidential *MachineConfidentialStatus `json:"confidential,omitempty"`
}

type MachineConfidentialStatus struct {
    // Technology is the observed confidential-computing technology.
    // The initial value is INTELTDX.
    Technology string `json:"technology"`

    // Attestation is the result for the nonce currently in spec.
    // +optional
    Attestation *MachineAttestationStatus `json:"attestation,omitempty"`
}

type AttestationPhase string

const (
    AttestationPhasePending AttestationPhase = "Pending"
    AttestationPhaseReady   AttestationPhase = "Ready"
    AttestationPhaseFailed  AttestationPhase = "Failed"
)

type MachineAttestationStatus struct {
    Phase AttestationPhase `json:"phase"`

    // NonceHash is the lower-case, hex-encoded SHA-256 digest of the
    // requested nonce, prefixed with "sha256:".
    NonceHash string `json:"nonceHash"`

    // Quote contains the unmodified evidence returned by the provider.
    // +optional
    Quote []byte `json:"quote,omitempty"`

    // QuoteFormat identifies the binary encoding of Quote.
    // +optional
    QuoteFormat string `json:"quoteFormat,omitempty"`

    // ReportDataFormat identifies how the challenge report data
    // included in Quote was constructed.
    // +optional
    ReportDataFormat string `json:"reportDataFormat,omitempty"`

    // ObservedAt is when the controller received the successful response.
    // +optional
    ObservedAt *metav1.Time `json:"observedAt,omitempty"`

    // Reason and Message describe pending or failed retrieval.
    // +optional
    Reason string `json:"reason,omitempty"`
    // +optional
    Message string `json:"message,omitempty"`
}
```

All of these fields belong to the `status` subresource and are writable only by
the responsible controller. `Quote` is a byte slice, so its YAML and JSON
representations are base64-encoded.

For Intel TDX, `Technology` is `INTELTDX`, `QuoteFormat` is `INTELTDXQuote`,
and `Quote` contains the raw TDX quote. An AMD SEV-SNP provider uses
`AMDSEVSNP`, `AMDSEVSNPAttestationReport`, and a raw SNP attestation report in
the same fields. 

### Example Machine

The following example requests a confidential VM and its first evidence
response. The other Machine resources and references follow the existing API.

```yaml
apiVersion: compute.ironcore.dev/v1alpha1
kind: Machine
metadata:
  name: confidential-vm
spec:
  machineClassRef:
    name: machineclass-sample
  confidential: true
  confidentialInitData: BASE64_ENCODED_INIT_DATA
  attestationNonce: Kj/T5bUOXCWZbbzZd2Iz0HydSUPGJwIX7jICn9+ZG9A=
  volumes:
    - name: rootdisk # The first disk is the root disk.
      volumeRef:
        name: my-volume
```

After successful evidence retrieval, its status looks like this:

```yaml
status:
  state: Running
  machineID: libvirt://example-confidential-machine
  confidential:
    technology: INTELTDX
    attestation:
      phase: Ready
      nonceHash: sha256:0b0156a81883b51b3ef7c9e49057d1b271d6d776492d6cf0f5dd77e2fbf79c8697a2
      quote: BASE64_ENCODED_TDX_QUOTE
      quoteFormat: INTELTDXQuote
      reportDataFormat: ironcore.dev/attestation/v1
      observedAt: "2026-11-24T12:00:00Z"
```

These values illustrate the response shape.

### Requesting Fresh Evidence

A caller requests fresh evidence by setting a new 32-byte cryptographically
random nonce in `spec.attestationNonce`. This requires `update` or `patch`
permission for the Machine. Read-only callers can inspect the latest evidence
in `Machine.status` but cannot request a fresh response. Before accepting a
result, the caller must verify that `nonceHash` matches its nonce.

`Pending` means retrieval is in progress. `Failed` contains a reason and
message and must not include a quote presented as the result of that request.
The caller may retry with a new nonce after a failure or timeout. Removing
`spec.attestationNonce` clears the result without changing the VM's
confidentiality or power configuration.

### Challenge Binding

The nonce must be incorporated into the evidence returned by the TEE. A status
timestamp alone is not a freshness guarantee, and a plain hash stored outside
the signed evidence does not provide that guarantee either.

For the initial TDX implementation, the common controller constructs the
64-byte TDX `REPORTDATA` field from a domain-separated digest:

```text
SHA-256(
    "ironcore.dev/attestation/v1" || 0x00 ||
    nonce ||
    string(machine.metadata.uid) ||
    machine.status.machineID
)
```

The digest starts `REPORTDATA`, and the remaining bytes are zero.
`reportDataFormat: ironcore.dev/attestation/v1` identifies the field order and
padding. The controller passes the constructed bytes to the guest attestation
agent.

The provider maps the common 64-byte report data to TDX `REPORTDATA` or SNP
`REPORT_DATA`. Both technologies can therefore use
`reportDataFormat: ironcore.dev/attestation/v1` without changing the common
challenge construction.

Initdata is separate from the freshness challenge. During VM creation, the
confidential launch stack commits its digest to a hardware launch measurement.
When initdata is supplied, a verifier reads the original bytes from Machine
desired state, derives the expected measurement according to the provider's
documented construction, and checks it against the signed quote. Requesting
fresh evidence with a new nonce does not change the initdata or launch
measurements.

The verifier checks that the quote's signed TDX `REPORTDATA` or SNP
`REPORT_DATA` matches the expected construction for the caller's nonce and the
queried Machine. The report-data construction alone does not define an
identity policy.

Any future change to the challenge construction requires a new format
identifier. 

### Controller and Provider Interaction

A nonce change causes the controller to clear the previous result, set
`phase: Pending`, and request fresh evidence through the provider. Attestation
requires a running VM with a provider machine ID and must not create or start
one. Success publishes the raw quote, its format, the report-data format, and
the observation time as `Ready`. Failure publishes a reason and message as
`Failed`.

The nonce hash and phase distinguish completed requests from new ones.
Controller restarts, status writes, and unrelated reconciliation must not
repeat successful requests, generate unsolicited evidence, or replace a newer
result with stale evidence.

### Guest Agent

The initial implementation uses an attestation agent inside the guest,
reachable by the provider over vsock. The provider sends the constructed
report data to this agent. Supported Linux guest images include a compatible
kernel and the preinstalled agent.

## Security Considerations

A provider can support confidential execution only when its host prerequisites,
VM launch support, and evidence retrieval work with supported guest images.

A controller must not accept `confidential: true` and then ignore it because
its provider lacks support. The provider must return an explicit error.

A quote alone does not prove that a VM is trustworthy. An independent
verifier must validate signatures, certificates, TCB claims, launch
measurements, and the nonce binding before using the result to trust the VM.
The nonce creates freshness only if the verifier generates a fresh random value
and checks its binding in the quote. `observedAt` and `nonceHash` are useful for
correlating controller state, but neither substitutes for checking the signed
report data. Secret material must not be embedded in a nonce, `confidentialInitData`,
or the signed report data.

A host can still delay requests and deny service. This affects availability without
compromising evidence validity.

Confidential memory does not implicitly protect disks, networking, cloud-init
or ignition data, or data sent to an unverified endpoint. Volume encryption and
authenticated networking continue to use their own mechanisms. Users must not
release secrets to a VM solely because `spec.confidential` was set or
`status.phase` is `Ready`; those conditions are not verification decisions.

## Testing Strategy

### API and Controller Tests

- Verify existing Machine compatibility, field validation, immutability, and
  size limits.
- Verify nonce hashing and the common report-data construction for TDX
  `REPORTDATA` and SNP `REPORT_DATA`.
- Verify status handling and format identifiers for `INTELTDX` and `AMDSEVSNP`
  with fake evidence.
- Verify attestation phase transitions, request correlation, and rejection of
  stale evidence.
- Verify that an unsupported confidential request or launch failure returns an
  error without creating an ordinary VM.

### Provider Tests

Use a fake confidential launch stack, guest attestation agent, or vsock
transport to verify that:

- The provider passes the exact initdata to the launch stack and rejects
  initdata it cannot measure.
- The provider resolves the correct runtime VM and sends the exact report data
  supplied by the controller.
- The provider returns the raw quote without provider-specific re-encoding.
- Other provider status calls do not cause evidence generation.

Provider tests initially cover Intel TDX. Common controller tests cover the
TDX and SNP API representations with fake evidence.

### Hardware Integration Tests

On machines that support TDX, use a valid guest image with the attestation
agent preinstalled and verify that:

- A confidential Machine launches with the expected initdata measurement and
  reports its observed technology.
- Successive nonce updates produce fresh evidence for an independent verifier
  without changing the initdata measurement.
- The verifier accepts valid evidence and rejects altered evidence, unexpected
  challenges, and unexpected initdata measurements.
- Existing Machines continue to run without exposing attestation evidence.

AMD SEV-SNP launch and quote validation require an SNP provider and suitable
hardware before end-to-end SNP support can be claimed.

## Alternatives

### Separate Attestation Request Object

A `MachineAttestationRequest` CRD would provide a one-shot request lifecycle
and avoid Machine generation changes. It also introduces another public API
and another object for users and controllers to manage. Since the requested
API already permits nonce updates on an existing Machine, the additional object
is unnecessary for the initial implementation.

### Dedicated Attestation RPC

A synchronous RPC could return evidence without changing Machine desired state
or generation. It would require a new public API with its own routing and
authorization. Keeping evidence in Machine status would also require polling
or a status cache. Updating the existing Machine keeps the interaction within
the Kubernetes API.

### Nonce in an Annotation

An annotation would avoid adding a typed spec field, but would provide weaker
validation, less discoverability, and an unclear reconciliation contract. A
typed field better documents the operation and its restrictions.

### Periodic Attestation Refresh

Periodic updates would provide recent-looking evidence but would consume guest
and provider resources, cause Machine status churn, and still lack the
cryptographic freshness supplied by a verifier's nonce. Evidence stays
unchanged until a new nonce arrives or an outstanding request needs a retry.

### Integrated Verification or Secret Release

IronCore could verify evidence and release workload secrets itself. This
would require launch policy, TCB, certificate, and key-lifecycle management.
Keeping these concerns external matches the Machine API's role as an
infrastructure abstraction and lets users choose their own verification policies.
