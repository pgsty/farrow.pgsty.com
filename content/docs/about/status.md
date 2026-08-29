---
title: Status
description: What the current simplified tree has passed natively, what remains unverified, and what blocks 1.0.
weight: 20
icon: fa-solid fa-list-check
aliases: [/docs/project/, /docs/project/status/, /docs/project/roadmap/, /docs/project/release/, /docs/project/design-history/]
---

Farrow is pre-1.0. Source tests, dated native replays, packages, release, CI,
and the public site are separate gates.

## Last recorded native replay — 2026-08-27

This matrix belongs to the exact checkpoint exercised that day. Later source
or documentation edits do not inherit native proof without another replay.

| Host | Path | Result |
|---|---|---|
| macOS 26.6.2 arm64 | HVF, QEMU 11.1, socket_vmnet | one node and additive four nodes passed |
| Ubuntu 26.04 amd64 (`mx`) | KVM, QEMU 10.2.1, NetworkManager | setup, one node, additive four nodes, and uninstall passed |

Both hosts passed fixed IP, SSH readiness, default CPU/memory/root/data disk,
cloud-init, stop/start, cross-directory commands, unchanged control-node boot
ID during scale-out, control-to-peer SSH, ignored unconsumed Pigsty changes,
absence-never-destroys, and explicit destroy.

Linux additionally proved valid NOPASSWD automation, caller-accessible Debian
helper policy, unprivileged bridge smoke, refusal to uninstall with four tap
members, and exact restoration after destroy.

Interactive host-network and hosts commands now invoke sudo themselves; an
external `sudo -v` is optional. Darwin cleanup can reconstruct an uninstall-
only ownership plan from byte-identical interface evidence, the exact launchd
plist, and installed binary digests when `network.json` is missing.

On 2026-08-28 the post-calibration tree passed unit, race, vet, staticcheck,
govulncheck, all four cross-builds, the simulated image-pipeline boundary,
license verification, and GoReleaser configuration validation. An isolated
local GoReleaser snapshot also built and verified all four archives, both
DEB/RPM architectures, SPDX documents, checksums, dependencies, modes, and
archive/package parity. Nothing was published, and those results do not extend
this native matrix.

## 0.1.0 readiness replay — 2026-08-28

An isolated clean commit tagged `v0.1.0` from the review working tree completed
the stable local release path: four archives, amd64/arm64 DEB and RPM packages,
dependency licenses, SPDX documents, Homebrew formula, installer, release
metadata, checksums, and an ephemeral Cosign signature/provenance round trip.
The generated user installer was idempotent in an isolated home directory.

The resulting Darwin/arm64 binary repaired two live nodes whose QMP sockets had
been removed externally by stopping and starting only those nodes. Both reached
readiness in 13.7 seconds; the two untouched peers retained their boot IDs.
All four then reported matching QMP/process identity, `plan` returned `none`,
`doctor` passed, and control-to-peer SSH worked. On Ubuntu 26.04 amd64, the DEB
and tar payloads were byte-identical, the packaged binary ran, its clean image
view used the embedded 17-artifact Catalog without probing a private mirror,
and the KVM/QEMU 10.2.1 doctor smoke passed. The DEB was extracted, not installed.

This is local candidate evidence, not a published release: the real source
commit/tag, remote CI/OIDC bundles, public repository and documentation DNS,
and clean-host Homebrew/DEB/RPM installation remain separate gates.

A later full macOS factory reset exposed one source-test dependency on an
already installed `qemu-img`: catalog-only `image list` could not run on a
blank host. The store is now resolved lazily only when local qcow2 bytes need
validation, regression tests explicitly remove QEMU from `PATH`, and the full
`make check` passes with QEMU/Farrow/network state absent. Because that source
fix postdates the artifact candidate above, that artifact set is historical;
the final source candidate below was regenerated from the later commit.

## Final 0.1.0 source candidate — 2026-08-29

Clean source commit `e8e08315ac7d07e30d1ea1adb98131dde293eacd` passed
`make check` and `goreleaser check`: unit/race tests, vet, staticcheck,
govulncheck, all four cross-builds, the simulated image-pipeline boundary,
dependency-license verification, and GoReleaser configuration validation.

An isolated clone temporarily tagged that exact commit as `v0.1.0` and passed
the stable local release path. It built and verified four platform archives,
amd64/arm64 DEB and RPM packages, eight SPDX documents, paired helper digests,
archive/package parity, a Homebrew formula, installer, release metadata, and
20 checksummed final assets. The extracted Darwin/arm64 binary reported version
`0.1.0` and the exact source commit. The temporary tag was not added to the
source repository; real tag, push, remote CI/OIDC bundles, and publication
remain separate release actions.

## Schema-3 Catalog closure — 2026-08-29

Catalog revision `2026082903` is the current source and development-repository
checkpoint: 9 families and 27 architecture-specific artifacts. The embedded
Catalog and published `catalog.json` have the same SHA-256
`571b1ff9c7d4d42355df3392ea62a339471c2d01d868669a7625fac8b93f245d`;
the published `repo.yaml` also matches the source-controlled authoring file.
Fresh HTTP and HTTPS clients accepted the detached signature from production
key `4686B39A40F9B562`.

All 27 published qcow2 files (19 GiB total) passed a full SHA-256 sweep against
the Catalog. A clean temporary Farrow home then downloaded the complete
409.3 MiB Darwin/arm64 default `d13:stable` artifact, rehashed it, and accepted
its qcow2 structure and virtual size. These are publication-integrity and
client-path checks, not a replacement for the native lifecycle matrix below;
no existing VM was recreated for this Catalog check.

## EL7/EL8 compatibility — 2026-08-28

Commit `7c666c7` restored EL7/EL8 after two independent Claude Code Opus 5 max
adversarial reviews. The first review blocked on destructive runtime preflight
ordering and signed-Catalog baseline migration; both were fixed, regression
tested, and the second review returned PASS with no required fixes.

At that checkpoint, Catalog `2026082801` was signed and active on the
development repository: 9 families, 17 image artifacts, and 19 SHA-verified
repository payloads including the two socket_vmnet archives. A clean client
accepted the public signature and exact embedded digest.

An isolated macOS arm64 lifecycle replay booted Rocky Linux 8.10 arm64 with the
built-in TCG compatibility rule, passed stop/start and readiness in 44.2 seconds,
and verified NetworkManager, fixed IP/no-route/no-DNS, `dba` UID/GID 88, and the
generation/spec marker. EL7 bytes, qcow2 structure, BIOS layout, and 4K XFS root
are verified; the current native Linux/amd64 Farrow lifecycle replay remains
open.

## Still open

- EL9 + NetworkManager + firewalld native replay;
- current systemd-networkd replay;
- host reboot persistence;
- macOS amd64 and Linux arm64 native runs;
- EL7 through the current native Linux/amd64 lifecycle;
- current 9p share replay;
- a complete current Pigsty `configure → farrow up → install.yml` run;
- published Homebrew/DEB/RPM consumption and release CI.

Current built-in versions are `supported`, except EOL EL7 and the retained
compatibility versions EL9 9.3/9.6 and EL10 10.0, which are `deprecated`.
Active and standby Catalog public keys are embedded, but the repository must
move off the development host and private-key custody/rotation plus release
custody must be formalized before 1.0.
