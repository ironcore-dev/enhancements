---
title: Community engagement rules for ironcore-dev GitHub org

iep-number: 16

creation-date: 2026-04-09

status: review

authors:

- "@gonzolino"

reviewers:

- "@maxmoehl"
- "@afritzler"
- "@peanball"
---

# IEP-16: Community engagement rules for ironcore-dev GitHub org

## Table of Contents

- [Summary](#summary)
- [Motivation](#motivation)
    - [Goals](#goals)
    - [Non-Goals](#non-goals)
- [Proposal](#proposal)
- [Alternatives](#alternatives)

## Summary

Rules of engagement for the IronCore community should be defined and
automatically enforced across the
[ironcore-dev](https://github.com/ironcore-dev) org on github.com.

## Motivation

The [ironcore-dev](https://github.com/ironcore-dev) org on github.com is
currently managed manually. As a result, settings for repos and teams vary and
responsibilities are unclear.

All members of the org have write access to all repositories by default,
regardless of their responsibilities.

Every repository specifies its own branch protection rules / rulesets and there
is a plethora of teams configured in the org with manually managed access to
repos.

Rules for the org must be written down and applied in the org automatically to
harmonize repository settings and rulesets, and apply permissions to org members
according to their responsibilities.

### Goals

- Create a repository with documentation describing the community engagement
  rules, e.g. how to become a contributor or maintainer and what
  responsibilities come with these roles.
- Specify initial set of contributors and maintainers.
- Specify which maintainer/contributor teams are responsible for which
  repositories.
- Setup automation to control maintainer/contributor teams, repository
  permissions and rulesets.

### Non-Goals

- Granular control of Github org settings.
- Control of repository settings that deviate from the standard.

## Proposal

### Repository with community documentation

Setup a `community` repository inside the
[ironcore-dev](https://github.com/ironcore-dev) org containing a `README.md` to
describe the community engagement rules. It should contain sections for
contributors and maintainers, each describing the tasks and responsibilities of
that role as well as stating the process how to become a maintainer/contributor.

Besides the `README.md`, the repository should contain a `teams.yaml` and a
`repositories.yaml` file with the following structure.

`teams.yaml` lists all maintainer/contributor teams with an authoritative list
of their members. To become a maintainer/contributor of a component, a PR must
be opened to adapt this file to add the user to the corresponding team (same for
removing a user from a team). There will be an additional `tsc` team for members
of the technical steering committee that serve as maintainers for org-wide
governance and tooling repos.

```yaml
- name: x-maintainers
  description: maintainers of component X
  members:
    - maintainer1
    - maintainer2

- name: x-contributors
  description: contributors of component X
  members:
    - contributor1
    - contributor2
    - contributor3
```

`repositories.yaml` lists all repositories of the org and specifies the
responsible maintainer/contributor teams with their permissions. The list of
collaborators is authoritative. All collaborator permissions to repositories are
assigned via teams. No individual users are allowed as collaborators.

```yaml
- name: x-repo
  description: Repository for component X
  collaborators:
    teams:
      - name: x-maintainers
        permission: push
      - name: x-contributors
        permission: pull
```

#### Documentation of contributor role

To become a contributor you should show interest in the respective area by engaging with the community via issues, PRs, and the community meetings. Once you are a contributor you are expected to contribute to discussions and help the project move forward by implementing accepted feature requests or fixing bugs.

Tasks and responsibilities of a contributor:
- Participate in issue discussions
- Review PRs
- Implement accepted issues

The following rules how to become a contributor should be codified:
- At least 5 interactions with repositories in the area in which the user wants
  to become a contributor in.
  Valid interactions are created issues, participation in issue discussions,
  created PRs or PR reviews.
- At least 1 of the interactions must be a created PR that got merged.
- An active maintainer must approve the user becoming a contributor.

The contributor role can be revoked if
- the contributor makes no active contributions for 6 months
- A simple majority of TSC members vote to revoke the contributor role

#### Documentation of maintainer role

To become a maintainer you must exhibit a strong technical understanding of most of the components of the respective area and have proven that you understand the direction of the project by driving discussions to their conclusion by reaching rough consensus with the existing maintainers. Once you are a maintainer you are expected to participate in the community meetings, triage issues, and merge pull requests.

Tasks and responsibilities of a maintainer:
- Same tasks as contributors
- Approve & merge PRs
- Triage issues

The following rules how to become a maintainer should be codified:
- Only contributors can become maintainer of an area.
- A majority of TSC members have to approve the contributor becoming a
  maintainer

The maintainer role can be revoked if
- the maintainer makes no active contributions for 6 months
- A simple majority of TSC members vote to revoke the maintainer role

### Setup for Teams and Repositories in the Org

The repositories of the org will be separated into four areas:

1. IaaS
2. Metal
3. Network
4. TSC

Each area will have a maintainers and a contributors github team. Teams will
share responsibility for repositories that are relevant to more than one area.
Repositories which do not fit into any area are out of scope for now.

Sections below list all org repositories, grouped into areas.

#### Area: IaaS

Compute, storage, virtualization, and the IronCore cloud API layer.

- [ceph-provider](https://github.com/ironcore-dev/ceph-provider) — Ceph provider for the IronCore storage interface
- [cloud-hypervisor-provider](https://github.com/ironcore-dev/cloud-hypervisor-provider) — cloud-hypervisor provider for the IronCore compute interface
- [cloud-provider-ironcore](https://github.com/ironcore-dev/cloud-provider-ironcore) — Kubernetes cloud controller for IronCore
- [FeOS](https://github.com/ironcore-dev/FeOS) — Rust-based init system for VMs/containers in clustered environments
- [gardener-extension-provider-ironcore](https://github.com/ironcore-dev/gardener-extension-provider-ironcore) — Gardener extension for IronCore cloud provider
- [ironcore](https://github.com/ironcore-dev/ironcore) — Core IaaS API and types
- [ironcore-csi-driver](https://github.com/ironcore-dev/ironcore-csi-driver) — Kubernetes CSI driver for IronCore
- [ironcore-image](https://github.com/ironcore-dev/ironcore-image) — OCI image spec, library and tooling
- [ironcore-in-a-box](https://github.com/ironcore-dev/ironcore-in-a-box) — All-in-one IronCore IaaS deployment
- [kubectl-ironcore](https://github.com/ironcore-dev/kubectl-ironcore) — kubectl plugin for IronCore IaaS types
- [libvirt-provider](https://github.com/ironcore-dev/libvirt-provider) — Libvirt provider for the IronCore compute interface
- [machine-controller-manager-provider-ironcore](https://github.com/ironcore-dev/machine-controller-manager-provider-ironcore) — MCM provider for IronCore

#### Area: Metal

Bare-metal server discovery, provisioning, firmware, and lifecycle.

- [boot-operator](https://github.com/ironcore-dev/boot-operator) — Controller for boot infrastructure (HTTPBoot, IPXEBoot)
- [cloud-provider-metal](https://github.com/ironcore-dev/cloud-provider-metal) — Kubernetes cloud controller for IronCore metal API
- [cluster-api-provider-ironcore-metal](https://github.com/ironcore-dev/cluster-api-provider-ironcore-metal) — Cluster API for IronCore bare metal
- [FeDHCP](https://github.com/ironcore-dev/FeDHCP) — DHCP service; serves PXE boot (metal) and network address assignment (network)
- [gardener-extension-provider-ironcore-metal](https://github.com/ironcore-dev/gardener-extension-provider-ironcore-metal) — Gardener extension for IronCore metal
- [ipam](https://github.com/ironcore-dev/ipam) — IP address management operator; allocates IPs for both bare-metal provisioning and network overlays
- [machine-controller-manager-provider-ironcore-metal](https://github.com/ironcore-dev/machine-controller-manager-provider-ironcore-metal) — MCM provider for bare metal
- [metal-load-balancer-controller](https://github.com/ironcore-dev/metal-load-balancer-controller) — Service IP announcement via metalbond; bridges metal infrastructure with network overlay
- [metal-operator](https://github.com/ironcore-dev/metal-operator) — Bare metal server discovery and provisioning
- [metal-token-rotate](https://github.com/ironcore-dev/metal-token-rotate) — Token delivery for metal-operator into Gardener
- [metaldata](https://github.com/ironcore-dev/metaldata) — Metadata service for bare metal

#### Area: Network

Dataplane, overlay networking, routing, and network device management.

- [dpservice](https://github.com/ironcore-dev/dpservice) — DPDK-based fast dataplane / L3 router / SDN enabler
- [ebpf-nat64](https://github.com/ironcore-dev/ebpf-nat64) — eBPF-based NAT64 for routers
- [ironcore-net](https://github.com/ironcore-dev/ironcore-net) — Provider-specific implementation of IronCore network types
- [metalbond](https://github.com/ironcore-dev/metalbond) — Route reflector for IronCore overlay network
- [metalnet](https://github.com/ironcore-dev/metalnet) — Kubernetes controller for dpservice networking resources
- [sonic-operator](https://github.com/ironcore-dev/sonic-operator) — Operator for SONiC switch devices

#### Area: IaaS + Metal

- [os-images](https://github.com/ironcore-dev/os-images) — Publishes OS images as OCI artifacts

#### Area: IaaS + Metal + Network

- [controller-utils](https://github.com/ironcore-dev/controller-utils) — Shared utility library for Kubernetes controllers
- [provider-utils](https://github.com/ironcore-dev/provider-utils) — Shared utility library for providers

#### Area: Org Governance (TSC)

Org-wide tooling, governance, documentation, and shared libraries.

- [.github](https://github.com/ironcore-dev/.github) — Org-level GitHub config and workflows
- [community](https://github.com/ironcore-dev/community) — This repository
- [enhancements](https://github.com/ironcore-dev/enhancements) — Enhancement proposals
- [ironcore-dev.github.io](https://github.com/ironcore-dev/ironcore-dev.github.io) — Project landing page and docs
- [openapi-extractor](https://github.com/ironcore-dev/openapi-extractor) — OpenAPI spec extraction tool
- [repository-template](https://github.com/ironcore-dev/repository-template) — Default repo template
- [roadmap](https://github.com/ironcore-dev/roadmap) — Cross-area roadmap tracking
- [steering](https://github.com/ironcore-dev/steering) — Technical Steering Committee

PRs in the `community` repository require approval from at least 4 members of the TSC.

### Automation via OpenTofu

To enforce the contents of the `teams.yaml` and `repositories.yaml`, the
`community` repository should contain an `automation` directory. Inside that
directory a project will read the YAML files and apply the settings
utilising the [github
provider](https://registry.terraform.io/providers/integrations/github) for
OpenTofu.

Apart from teams and repository collaborator permissions, OpenTofu should also
enforce org wide rulesets specifying requirements for merging PRs. It will also
set default member permissions to read.

OpenTofu will run in a GitHub action on every push to the `main` branch in the
`community` repo. Direct pushes to `main` are forbidden, so each push is
actually a merged PR that got tested and reviewed.

For state storage, the "local" backend will be used to store the state in a JSON
file. This file will be committed to git after each change. To prevent conflicts
with code changes happening in parallel to potential applies, the state file
will be committed to a dedicated `state` branch. Rulesets must be put in place
to ensure only valid state files are pushed (e.g. enforce linear history, pushes
only from github-actions).

To ensure OpenTofu plans get reviewed & approved before applying them, [GitHub
deployments](https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/review-deployments)
will be used. TSC members will be in charge of approving code changes and plans.

## Alternatives

- Keep manual management of org users and repos.
- A set of org owners are made responsible to apply the changes from the
  community YAML manually.
- Automation is done via alternative tool to OpenTofu (research required to
  identify tools)
- Automation is done via custom developed tooling
