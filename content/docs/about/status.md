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

## EL7/EL8 compatibility — 2026-08-28

Commit `7c666c7` restored EL7/EL8 after two independent Claude Code Opus 5 max
adversarial reviews. The first review blocked on destructive runtime preflight
ordering and signed-Catalog baseline migration; both were fixed, regression
tested, and the second review returned PASS with no required fixes.

Catalog `2026082801` is signed and active on the development repository: 9
families, 17 image artifacts, and 19 SHA-verified repository payloads including
the two socket_vmnet archives. A clean client accepted the public signature and
exact embedded digest.

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

Current images remain `testing`, except EOL EL7 which is `deprecated`. Active
and standby Catalog public keys are embedded, but the repository must move off
the development host and private-key custody/rotation plus release custody must
be formalized before 1.0.
