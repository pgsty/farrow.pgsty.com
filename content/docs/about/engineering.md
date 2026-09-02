---
title: Engineering
description: Source layout, build and test gates, image normalization, release outputs, and evidence policy.
weight: 30
icon: fa-solid fa-screwdriver-wrench
---

## Repository boundary

The Farrow source repository contains code, tests, build/package definitions,
legal notices, and a short landing README. This site is the authoritative home
for user, design, operator, and release documentation. Raw review transcripts,
historical scratch inventories, demo directories, generated binaries, and
release output trees are not source inputs and must not be committed.

Generated output is disposable:

- `bin/` — development builds;
- `dist/` and `.goreleaser-*` — release/snapshot staging;
- root `farrow`, `farrow-hosts-helper`, and `catalogsign` binaries;
- Hugo `public/` and `resources/`.

## Build and source gates

```bash
make check
```

`make check` runs every source gate; the Makefile is the authoritative list of
what that includes. A source gate is not native VM evidence;
macOS HVF, Linux KVM/networking, package consumption, release publication, and
the public render remain separate gates.

## Release and package contract

Release tooling under `packaging/`, `.goreleaser.yaml`, and `.github/workflows`
is source, even though its generated directories are not. Archives and Linux
packages contain the matching binaries plus `LICENSE`, the minimal source
README, build metadata, and exact upstream license bytes reconstructed from the
module versions pinned by `go.mod`. Those generated license texts live under
`licenses/` in archives and are not tracked as source. Detailed documentation
stays on this versioned site rather than being copied into every binary payload.

Never infer publication from a successful build. Commit, tag, archive/package,
signature/attestation, upload, CI, and public consumption are distinct gates.
The stable local release command additionally requires a clean tagged commit
and its `origin` remote because GoReleaser records repository identity.

## Image normalization

`packaging/image-pipeline/` accepts an explicit local qcow2 source; it never
downloads or uploads. It copies and hashes the source, forces qcow2 parsing,
rejects backing/external/encrypted/unknown features, runs `qemu-img check`, and
can perform a no-network offline Guest mutation in an explicit QEMU sandbox.
UID/GID 88 collisions are rejected rather than rewritten ambiguously.

Catalog bytes are exported with:

```bash
go run ./tools/catalogexport /absolute/new/catalog.json
```

The exporter is atomic and no-clobber. Catalog signing and application release
signing use separate keys and trust domains.

## Evidence policy

Historical M0–M4 notes were useful during implementation but are not product
documentation. Their durable conclusions are condensed into [Design](design/)
and [Status](status/). A later source edit inherits no native proof; every
status claim names its date, host, path, and remaining gates.
