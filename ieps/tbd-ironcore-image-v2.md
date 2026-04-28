---
title: IEP Title

iep-number: tbd

creation-date: 2026-04-27

status: draft

authors:

- "@adracus"

reviewers:

- "@main-reviewer-1"
- "@main-reviewer-2"

---


# IEP-NNNN: IronCore Image V2

## Table of Contents

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
- [Alternatives](#alternatives)

## Summary

Create well-defined formats for our various image types and restructure
UKI-like booting.

## Motivation

Our current image definition and structure has evolved largely over time.
While working fine for our implementation so far, there are various important
shortcomings:

### Boot Modes

Within our single manifest and configuration type, we house multiple
different ways to boot:

* [UKI](https://github.com/ironcore-dev/ironcore-image/blob/f89db68d5e9323ed5d6024ba70897e077504c25b/image.go#L21)
* [ISO](https://github.com/ironcore-dev/ironcore-image/blob/f89db68d5e9323ed5d6024ba70897e077504c25b/image.go#L22)
* [Direct kernel + initramfs](https://github.com/ironcore-dev/ironcore-image/blob/f89db68d5e9323ed5d6024ba70897e077504c25b/image.go#L19)
* [Squashfs](https://github.com/ironcore-dev/ironcore-image/blob/main/image.go#L20) 

There's even a 'missing' way to boot: 'Raw' EFI executables.

### Lack of Layer Deduplication

For UKI booting, since we allow specifying the entire UKI, we cannot make use
of OCI image layer deduplication: A UKI, whilst having structure internally,
is specified as a single image layer.

### Composability of initramfs

For kernel + initramfs booting, we allow passing in a squashfs layer that
can be appended to the initramfs during boot. However, we could actually allow
an arbitrary number of initramfs layers, as these can be concatenated, allowing
for higher reuse and again layer deduplication.

### Installer Images

While gaining more and more experience building images, there is the notion of
'installer images': Images that know how to write an OS to disk and then, after
successful write to disk, want to reboot from disk. Currently, these installer
images are mostly handcrafted although there is potential for extracting this
functionality using e.g. initramfs layering.

### Goals

* Create the following cleanly separated boot modes, each having their own
  config media and manifest type:
  * UKI-like boot mode
  * Disk boot mode
  * Raw EFI executable boot mode
* Don't specify UKIs as UKI binary blobs anymore. Instead,
  specify their components (kernel, initrd, stub etc.) as distinct layers,
  allowing for layer deduplication. If someone still chooses to e.g. specify
  a UKI as raw EFI executable, no deduping / further processing will happen.
* Create facilities for installer images: Namely, composability of initramfs
  and simple reboot to disk after finishing disk writing.

### Non-Goals

* None identified yet

## Proposal

Create the following boot types and specs:

### UKI component booting

UKI component booting is for booting with the components that
make up a UKI. To keep names concise, `uki` is used as name.
Though being slightly technically inaccurate (the UKI OCI image is
not a UKI yet but rather its components), `uki` is a short identifier
that gives everyone the right mental model.

#### Manifest

**Media Type**

`application/vnd.ironcore.image.uki.v2+json`

#### Config

**Media Type**

`application/vnd.ironcore.config.uki.v2+json`

**Structure**

```json
{
  "cmdline": "optional kernel command line string",
  "osRelease": "os-release description",
  "rebootMethod": "optional string, can currently only be 'disk' to indicate to boot from disk after successful boot"
}
```

#### Kernel

A kernel binary, required:

**Media Type**

`application/vnd.ironcore.kernel.efi`

#### Stub

EFI stub. Optional.

**Media Type**

`application/vnd.ironcore.stub.efi`.

#### Initrd

Initial RAM disk, 1 to N times required. Optionally compressed
using either `gzip`, `zstd`, `xz` or `lz4`.

**Media Type**

`application/vnd.ironcore.initrd.cpio` with the compression
being appended via default media type extension (e.g. `+xz`).

### EFI executable booting

EFI executable booting just allows specifying a raw EFI
executable to boot.

#### Manifest

**Media Type**

`application/vnd.ironcore.image.efi.v2+json`

#### Config

**Media Type**

`application/vnd.ironcore.config.efi.v2+json`

**Structure**

```json
{}
```

#### EFI Executable

The EFI executable, required.

**Media Type**

`application/vnd.ironcore.efi.executable`

### Disk booting

Disk booting specifies a disk to boot from.

**Media Type**

`application/vnd.ironcore.config.disk.v2+json`

**Structure**

```json
{}
```

#### Disk

The disk image to boot.

**Media Type**

Currently, only `raw` files (that could be directly `dd`ed on disk) via

`application/vnd.ironcore.raw`

## Alternatives

None so far.