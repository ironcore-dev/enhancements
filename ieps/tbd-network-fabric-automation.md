---
title: "A Concept for Vendor-Agnostic, Incremental Switch Automation"

iep-number: tbd

creation-date: 2026-06-29

status: draft

authors:

- "@hardikdr"

reviewers:

- tbd

---

# IEP-tbd: A Concept for Vendor-Agnostic, Incremental Switch Automation

## Table of Contents

- [Summary](#summary)
- [Motivation](#motivation)
    - [Goals](#goals)
    - [Non-Goals](#non-goals)
- [Proposal](#proposal)
    - [Core API Layer](#core-api-layer)
    - [Device](#device)
    - [Switch](#switch)
    - [SwitchInterface](#switchinterface)
    - [Provider Interface](#provider-interface)
    - [Incremental Growth: Fabric API](#incremental-growth-fabric-api)
- [Workflows](#workflows)
    - [Pre-boot Sanity Checks](#pre-boot-sanity-checks)
    - [Adding a New Switch to the Fabric](#adding-a-new-switch-to-the-fabric)
    - [Replacing a Faulty Switch](#replacing-a-faulty-switch)
    - [Server-Side Changes](#server-side-changes)
- [Alternatives](#alternatives)

---

## Summary

This document proposes a conceptual framework for Kubernetes-native, vendor-agnostic
switch automation. It is not an API specification. The API examples shown here are
illustrative — they exist to make the concept concrete and are not the final design.
Detailed API decisions will be made during implementation.

The proposal is intentionally minimal. We start with a small, well-defined core
(`Device`, `Switch`, `SwitchInterface`) and grow the API surface organically as
real use cases demand it. Several of the foundational ideas in this document,
particularly the `Device`/`Switch` separation originate from the original Wire
proposal (https://github.com/ironcore-dev/enhancements/pull/21). This
document builds on that foundation.

---

## Motivation

Switches in a data center have a lifecycle. They get racked, they get configured,
they run for years, and eventually they get replaced or repurposed. Today, managing
this lifecycle involves a mix of ZTP templates, manual YAML, and tribal knowledge
about which switches are available, which are in use, and which are in maintenance.

The goal is to bring this under Kubernetes-native control: declarative, observable,
and automatable.

### Goals

- **Standardize the switch lifecycle.** Define onboarding, offboarding and configuration aspects of the switch.
  Every switch follows the same path regardless of vendor.

- **Vendor-agnostic by design.** The core API is neutral. Vendor differences live
  behind a provider interface, not in the API itself.

- **Automate configuration across a fleet.** A single intent expressed at the fabric
  level should be able to configure dozens of switches consistently, not one YAML at
  a time.

- **Start small and grow.** The initial object model is the minimum needed to be
  useful. New capabilities are added when real use cases justify them.

### Non-Goals

- **Solving for one vendor.** Any design choice that only makes sense for SONiC,
  Cisco, or any other specific platform is a non-goal at the core API level. Vendor
  specifics belong in provider extensions.

- **Modeling a specific topology.** CLOS, spine-leaf, flat L2 — the API should not
  be hardwired to one physical arrangement. Topology assumptions can live in higher-
  level controllers (e.g., Fabric), not the core objects.

- **Exposing every knob.** The goal is not a Kubernetes projection of gNMI or NETCONF.
  We are not building a general-purpose switch configuration tool.

- **Finalizing the API here.** This document is a concept. The actual field names,
  resource shapes, and versioning will be decided when we implement.

---

## Proposal

### Core API Layer

The entire initial API surface is three objects:

```
Device          (cluster-scoped)  — physical identity of a switch
Switch          (namespaced)      — desired configuration of a switch
SwitchInterface (namespaced)      — per-port desired state and observed status
```

Organic growth towards objects like `Server`, `ServerInterface`, `Link` will evolve later when foundation
is solid and use cases drive them.

```
                  ┌──────────────────────────────────────────────────┐
                  │           FeDHCP                                 │
                  └──────────────────┬───────────────────────────────┘
                                     │ creates
                                     ▼
                              ┌─────────────┐
                              │   Device    │  (cluster-scoped)
                              │  physical   │
                              │  identity   │
                              └──────┬──────┘
                                     │ bound by
                                     ▼
                              ┌─────────────┐          ┌──────────────────┐
                              │   Switch    │──────────▶│  (Secret: ZTP)   │
                              │  desired    │          └──────────────────┘
                              │   config    │
                              └──────┬──────┘
                                     │ auto-creates
                                     ▼
                        ┌────────────────────────┐
                        │    SwitchInterface     │  × N ports
                        │  per-port state        │
                        └────────────────────────┘
                                     │
                                     ▼
                           ┌──────────────────┐
                           │  Provider        │
                           │  (ApplySwitch)   │
                           └──────────────────┘
```

---

### Device

`Device` is the physical reality. It represents a switch that exists in a rack,
regardless of whether it has been configured yet. When switches are cabled in,
`Device` objects should appear automatically, this could be via FeDHCP or even
by a user initially.

`Device.spec` is minimal by design.

```yaml
# Created automatically by FeDHCP on first DHCP request
apiVersion: wire.ironcore.dev/v1alpha1
kind: Device
metadata:
  name: aa-bb-cc-dd-ee-ff        # MAC as stable, natural key
spec:
  maintenanceMode: false
status:
  ip: 10.0.0.1
  mac: aa:bb:cc:dd:ee:ff
  vendor: Edgecore
  model: AS9516-32D
  portCount: 32
  providerID: sonic://chassis-1   # set on first agent contact
  switchRef:
    namespace: network-prod
    name: leaf-01
  conditions:
  - type: Identified              # mac-db lookup succeeded
    status: "True"
  - type: Reachable               # management plane is up
    status: "True"
```
---

### Switch

`Switch` is the desired configuration for a device. It binds to a `Device` via
`deviceRef` (the only immutable field), and contains everything needed to
bootstrap and configure the switch. For the initial implementation, this is intentionally kept
to the basics only what is needed to bring a switch online.

The key design insight: a `Switch` object is the single blob handed to the
provider. The provider decides how to sequence and apply it. A SONiC provider
talks to the sonic-agent over gRPC. A Cisco provider talks to gNMI. The `Switch`
object does not care.

**ZTP bootstrap.** A `Switch` can reference a `Secret` containing a ZTP script.
That script's only job is minimal: install the agent, set up basic connectivity,
signal readiness. The operator then takes over from there with real configuration.
Alternatively, a `Switch` can reference a named bootstrap template for standard
pre-defined setups.

**Auto-create interfaces.** When `autoDiscoverInterfaces: true` (the default),
the operator creates `SwitchInterface` objects automatically once the agent
reports the port list. This means an operator never needs to know interface names
upfront, they appear in the cluster once the switch is alive.

```yaml
apiVersion: wire.ironcore.dev/v1alpha1
kind: Switch
metadata:
  namespace: network-prod
  name: leaf-01
spec:
  deviceRef:
    name: aa-bb-cc-dd-ee-ff      # immutable
  hostname: leaf-01
  ...
  prefixes:
  - 10.0.0.0/64
  vlans:
  - id: 1000
    prefix: 10.0.100.0/24
    ..  
  bgp:
    asn: 0001
    peerGroups:
      - name: leafs
        neighbors:
          - interfaceRef:
              name: spine-01-if-01
  autoDiscoverInterfaces: true
  initZTPScript:
    secretRef:
      name: leaf-ztp-script      # optional: inject a ZTP script via secret to install agent etc.
status:
  phase: Ready                   # Pending | Ready | Error
  observedGeneration: 4
  conditions:
  - type: DeviceBound 
    status: "True"
  - type: BootstrapComplete
    status: "True"
  - type: ConfigApplied
    status: "True"
    observedGeneration: 4
```

To reconfigure a switch, change VLAN ranges, update BGP ASN, update the `Switch` spec.
The controller detects the generation change and pushes the delta to the device.
To rebind a switch to different hardware, delete and recreate the `Switch` object.
This keeps reconfiguration explicit and auditable.

---

### SwitchInterface

`SwitchInterface` represents a single physical port. These are auto-created by
the operator after the agent reports interface discovery.

```yaml
apiVersion: wire.ironcore.dev/v1alpha1
kind: SwitchInterface
metadata:
  namespace: network-prod
  name: leaf-01-Ethernet4
spec:
  switchRef:
    name: leaf-01
  adminState: Up
status:
  operState: Up
  speed: 100G
  lldpNeighbor:
    systemName: host-01
    portDescription: eth0
  conditions:
  - type: AdminStateApplied
    status: "True"
```

---

### Provider Interface

The operator core has no vendor code. Providers implement a Go interface and ship
an agent that speaks the device's native protocol.
The core calls `ApplySwitch` with the full desired state; the provider decides
the apply sequence. This is the critical extensibility point.

```go
type SwitchRuntime interface {
    ApplySwitch(ctx context.Context, device string, cfg *SwitchConfig) error
    DeleteSwitch(ctx context.Context, device string) error
    ListInterfaces(ctx context.Context, device string) ([]*InterfaceInfo, error)
    SetInterfaceAdminState(ctx context.Context, device string, iface string, up bool) error
}
```

A SONiC provider connects to a sonic-agent gRPC endpoint running inside the
device. A Cisco provider connects to gNMI. The `Switch` spec is the same; the
wiring is different.

---

### Incremental Growth: Fabric API

Once the core layer is working, the next natural step is expressing intent at
the fabric level, configuring a whole set of switches with a single object
rather than one `Switch` at a time.

A `Fabric` object uses a label selector to claim a set of `Device` objects and
derives and applies a consistent, topology-aware configuration across all of them.
The `Fabric` controller creates the `Switch` objects underneath, operators do not
write Switch objects by hand when a Fabric is active.

```yaml
apiVersion: wire.ironcore.dev/v1alpha1
kind: FooFabric   # | IroncoreL2Fabric | ..
metadata:
  namespace: network-prod
  name: fra3-leaf-fabric
spec:
  immutabilityPolicy: Strict   # Strict | Relaxed 
  deviceSelector:
    matchLabels:
      fabric: fra3
      role: leaf
  topology: SpineLeaf
  vlanStrategy: PerPort        # one VLAN per server port, ID derived from port index
  bgpStrategy: Unnumbered
  asnBase: 4231000000          # per-leaf ASN = asnBase + leaf-id label
  loopbackPrefixBase: "2a10:cafe:e013"
  ...
```

#### Immutability as a fabric-level policy

Immutability is enforced on the `Switch` / `SwitchInterface` objects via Fabric level intent.

The core API does not bake in an opinion about whether switches should change after initial configuration,
because that question has a different answer depending on the topology.

For stable, well-understood topologies: VLAN-per-port, BGP unnumbered spine-leaf,
config changes after bootstrap are rare by design. In these environments, any
modification to a running switch should be deliberate, reviewed, and applied through
a controlled path. For these topologies, the `Fabric` object carries an
`immutabilityPolicy: Strict` that makes this explicit: the Fabric controller owns
the `Switch` objects it creates, and any direct modification is reconciled back
immediately. The `Switch` objects become the rendered output of the Fabric intent,
not something an operator edits by hand.

For more dynamic topologies, multi-tenant overlays, config does
change more frequently as workloads come and go. Here, `immutabilityPolicy: Relaxed`
gives operators the flexibility to update Switch objects directly or via
automation outside the Fabric.

The key point: **immutability is a property of the topology, expressed through the
Fabric.** The `Switch` object is the mechanism; the `Fabric` object is where
operational policy lives.

When a change is needed in a `Strict` fabric: planned switch replacement, topology
expansion, BGP policy update - the operator updates the `Fabric` spec. The Fabric
controller derives the new desired state and applies it in a controlled way across
all owned switches consistently.

---

## Workflows

The following scenarios walk through how the API handles real operational events.
These are sketches, not complete runbooks.

### Pre-boot Sanity Checks

**Scenario:** A batch of new switches has been racked. Before they go into
production, someone needs to verify that each one is reachable, the hardware is
what DCIM says it is, and the ports are all up.

With `Device` objects created by FeDHCP on first DHCP request, this becomes
observable immediately:

1. Operator checks `Device.status.conditions` — `Identified: True` means the
   MAC matched a mac-db entry; `Reachable: True` means the management plane
   responded.
2. `Device.status.vendor` and `.model` can be validated against DCIM records
   automatically by a simple script or controller.
3. No `Switch` object is needed yet. The device sits in a discovered-but-unclaimed
   state. This is a safe, observable holding state, not limbo.

Alternatively a dedicated Fabric object such as `PreBootSanityFabric` could even create
no-op switch objects and do the validation.
Pre-boot validation requires zero new API machinery. The `Device` object alone
is sufficient in most cases.

---

### Adding a New Switch to the Fabric

**Scenario:** The fabric is growing. A new leaf switch is being added to serve
additional servers.

1. Switch is racked and cabled. FeDHCP sees the DHCP request → `Device` object
   appears with MAC as name.
2. Infrastructure team labels the `Device` (e.g., `role: leaf`, `rack: r04`).
3. Wire operator creates a `Switch` object referencing the new `Device`,
   with the desired VLAN and BGP config for this leaf.
4. The operator bootstraps the switch (ZTP script from secret), the sonic-agent
   comes up, interfaces are discovered, `SwitchInterface` objects are created.
5. `Switch.status.phase` transitions to `Ready`.
6. If a `Fabric` object exists with a selector matching this device's labels,
   the fabric controller picks it up automatically and incorporates it, no
   manual `Switch` creation needed.

The addition is explicit (someone created the `Switch`) but as automated as
the operator chooses to make it.

---

### Replacing a Faulty Switch

**Scenario:** A leaf switch has a hardware fault and needs to be swapped out.
The replacement should take over with the same configuration, with minimal
disruption.

1. Operator sets `Device.spec.maintenanceMode: true` on the faulty device. This
   signals controllers that traffic should not be directed here; higher-level
   controllers can act on this (e.g., drain BGP peers, mark associated
   `SwitchInterface` objects `adminState: Down`).
2. Physical swap happens. New switch is racked with same cabling.
3. New switch sends a DHCP request → new `Device` object is created with the
   new MAC.
4. Fabric Operator identifies it.
5. Bootstrap runs, agent comes up, interfaces are discovered, configuration
   is applied. Old `Device` object is deleted.

---

### Server-Side Changes

**Scenario:** A server is being decommissioned and its switch ports should be
shut down. Or a new server is being connected and its port needs to come up.

Because `SwitchInterface` objects exist per port and LLDP neighbor info is in
`status`, a higher-level automation can:

1. Query `SwitchInterface` objects where `status.lldpNeighbor.systemName` matches
   the server being decommissioned.
2. Set `spec.adminState: Down` on those interfaces.
3. The operator pushes the change to the device via `SetInterfaceAdminState`.

For a new server connection:

1. Server is cabled. The `SwitchInterface` for that port will show the new LLDP
   neighbor in status within seconds of the server being powered on.
2. Operator (or automation) sets `adminState: Up` if it was previously down.

This means server lifecycle events can drive switch port state changes without
manual port lookups — the LLDP data tells you which port to touch.

---

## Alternatives

**General-purpose gNMI projection.**
Exposing every switch knob as a Kubernetes resource makes the API surface
unbounded and tightly couples the Kubernetes API to vendor-specific data models.
We want automation for specific, well-understood use cases not a Kubernetes
wrapper around the entire gNMI tree.
