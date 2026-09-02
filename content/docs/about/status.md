---
title: Status
description: What the current tree has passed natively, what remains unverified, and what blocks 1.0.
weight: 20
icon: fa-solid fa-list-check
aliases: [/docs/project/, /docs/project/status/, /docs/project/roadmap/, /docs/project/release/, /docs/project/design-history/]
---

Farrow is pre-1.0. Source tests, dated native replays, packages, release, CI,
and the public site are separate gates. Install instructions are in the
[Quick Start](../../start/tutorial/#install).

The current public release is
[`v0.4.0`](https://github.com/pgsty/farrow/releases/tag/v0.4.0). Its source and
package gates passed at commit `8ecb8476c7dbc934d8cbbee935ac53884d00fcdb`;
the native-host rows below remain the latest dated VM evidence rather than
being implicitly promoted to 0.4.0.

## Summary

| Host | Path | Last verified | Result |
|---|---|---|---|
| macOS 26.6.2 arm64 | HVF, QEMU 11.1, socket_vmnet | 2026-09-01 (`v0.2.0`) | selected create/SSH/stop/start, incremental cached create, whole status, whole destroy passed |
| macOS 26.6.2 arm64 | HVF, QEMU 11.1, socket_vmnet | 2026-08-27 | one node and additive four nodes passed |
| Ubuntu 26.04 amd64 (`mx`) | KVM, QEMU 10.2.1, NetworkManager | 2026-09-01 (`v0.2.0`) | audited an existing four-node deployment as live and reached its control guest |
| Ubuntu 26.04 amd64 (`mx`) | KVM, QEMU 10.2.1, NetworkManager | 2026-08-27 | setup, one node, additive four nodes, and uninstall passed |
| macOS arm64 | HVF host, TCG compatibility rule, Rocky Linux 8.10 arm64 | 2026-08-28 | boot, stop/start, readiness in 44.2 s passed |
| Published Catalog `2026082903` | 9 families, 27 signed qcow2 artifacts | 2026-08-29 | full SHA-256 sweep and clean-client `d13:stable` pull passed |

## Still open

- EL9 + NetworkManager + firewalld native replay;
- current systemd-networkd replay;
- host reboot persistence;
- macOS amd64 and Linux arm64 native runs;
- EL7 through the current native Linux/amd64 lifecycle;
- current 9p share replay;
- a complete current Pigsty `configure → farrow up → install.yml` run;
- clean-host published Homebrew/DEB/RPM consumption.

Current built-in versions are `supported`, except EOL EL7 and the retained
compatibility versions EL9 9.3/9.6 and EL10 10.0, which are `deprecated`.
Active and standby Catalog public keys are embedded, but the repository must
move off the development host and private-key custody/rotation plus release
custody must be formalized before 1.0.

## Verification history

Each entry belongs to the exact checkpoint exercised that day. Later source or
documentation edits do not inherit native proof without another replay.

### Farrow 0.4.0 release — 2026-09-02

The exact tag commit passed `make check` and the release workflow's archive,
DEB/RPM, SBOM, checksum, installer, Homebrew-formula, and package-parity gates.
`up` now runs `setup` itself on an unprepared terminal host, `vm_disks[].fs`
defaults to `auto`, readiness failures carry the guest's last error line, and
every command shares one output style. The first-run path was not replayed on a
fresh host; no new native VM replay is claimed here.

### Farrow 0.3.0 release — 2026-09-02

The exact tag commit passed `make check` and the release workflow's archive,
DEB/RPM, SBOM, checksum, installer, Homebrew-formula, and package-parity gates.
Catalog refresh is now explicit (`farrow update` for the configured repository,
`image sync` for an exact source), and guest readiness failures carry per-node
stages and next-step log commands. No new native VM replay is claimed here.

### Farrow 0.2.0 release — 2026-09-01

Source commit `59d1b62aebb3d044a317e4006cc8a0bf56f4feaf` is tagged `v0.2.0`.
Its source CI and independently dispatched packaging workflow passed the exact
commit. The stable local release path also built and verified all four platform
archives, amd64/arm64 DEB and RPM packages, eight SPDX documents, paired helper
digests, archive/package parity, Homebrew formula, installer, release metadata,
and 19 checksummed final assets.

The macOS arm64/HVF replay ran with MonoProxy's covering `10.0.0.0/8` exclusion
present. Selected `u24-1` create/SSH/stop/start, incremental cached `el9-1`
create, whole status with five absent desired peers, both SSH connections, and
whole destroy/SSH-fragment cleanup passed. On Ubuntu 26.04 amd64/KVM, the Linux
binary audited an existing four-node Farrow deployment as live and reached its
control guest without mutating that host.

The compiled default image repository remains the signed COS-backed
`https://repo.pigsty.cc/farrow`. The independently checked
`https://repo.pgsty.com/farrow` source endpoint serves identical Catalog,
authoring metadata, checksums, and image bytes with a read-only Nginx worker.

### Schema-3 Catalog closure — 2026-08-29

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
client-path checks, not a replacement for the native lifecycle matrix; no
existing VM was recreated for this Catalog check.

### 0.1.0 candidates — 2026-08-28 and 2026-08-29

Two isolated `v0.1.0` candidates passed the stable local release path before
0.2.0 superseded them. Two facts from those runs still stand on their own: the
Darwin/arm64 binary repaired two live nodes whose QMP sockets had been removed
externally by stopping and starting only those nodes, both reaching readiness in
13.7 seconds while the two untouched peers kept their boot IDs; and a full macOS
factory reset exposed a source-test dependency on an installed `qemu-img`, so
catalog-only `image list` could not run on a blank host. The store is resolved
lazily only when local qcow2 bytes need validation, regression tests explicitly
remove QEMU from `PATH`, and `make check` passes with QEMU, Farrow, and network
state absent.

### EL7/EL8 compatibility — 2026-08-28

Commit `7c666c7` restored EL7/EL8 after two independent adversarial reviews.
The first review blocked on destructive runtime preflight ordering and
signed-Catalog baseline migration; both were fixed, regression tested, and the
second review returned PASS with no required fixes.

At that checkpoint, Catalog `2026082801` was signed and active on the
development repository: 9 families, 17 image artifacts, and 19 SHA-verified
repository payloads including the two socket_vmnet archives. A clean client
accepted the public signature and exact embedded digest.

An isolated macOS arm64 lifecycle replay booted Rocky Linux 8.10 arm64 with the
built-in TCG compatibility rule, passed stop/start and readiness in 44.2 seconds,
and verified NetworkManager, fixed IP/no-route/no-DNS, `dba` UID/GID 88, and the
generation/spec marker. EL7 bytes, qcow2 structure, BIOS layout, and 4K XFS root
are verified; the native Linux/amd64 Farrow lifecycle replay remains open.

### Native replay — 2026-08-27

Both hosts in the summary table passed fixed IP, SSH readiness, default
CPU/memory/root/data disk, cloud-init, stop/start, cross-directory commands,
unchanged control-node boot ID during scale-out, control-to-peer SSH, ignored
unconsumed Pigsty changes, absence-never-destroys, and explicit destroy.

Linux additionally proved valid NOPASSWD automation, caller-accessible Debian
helper policy, unprivileged bridge smoke, refusal to uninstall with four tap
members, and exact restoration after destroy.

Interactive host-network and hosts commands invoke sudo themselves; an external
`sudo -v` is optional. Darwin cleanup can reconstruct an uninstall-only
ownership plan from byte-identical interface evidence, the exact launchd plist,
and installed binary digests when `network.json` is missing.

On 2026-08-28 the post-calibration tree passed unit, race, vet, staticcheck,
govulncheck, all four cross-builds, the simulated image-pipeline boundary,
license verification, and GoReleaser configuration validation. An isolated
local GoReleaser snapshot also built and verified all four archives, both
DEB/RPM architectures, SPDX documents, checksums, dependencies, modes, and
archive/package parity. Nothing was published, and those results do not extend
the native matrix.
