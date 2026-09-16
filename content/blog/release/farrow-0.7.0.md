---
title: "Farrow 0.7.0: simpler test labs and automatic recovery"
linkTitle: Farrow 0.7.0
description: A shorter first run, compact progress, independent guest setup, and recovery through up.
date: 2026-09-16
weight: 1
categories: [Release]
tags: [Farrow 0.7.0, QEMU, Release]
icon: fa-solid fa-rocket
---

Farrow 0.7.0 makes local test labs easier to start and recover. Repeat `farrow up`
to finish interrupted work, retry incomplete guest setup, and refresh older
guest helpers without restarting running VMs.

## What changed

- **A shorter first run.** Interactive `up` can create the default inventory,
  prepare missing host tools, and restore an intact inactive Farrow network.
  Short checks stay quiet; long work shows progress and a compact final result.
- **Usable guests stay available.** Data disks, shares, guest hostnames,
  node-to-node SSH, and private networking initialize independently. Optional
  failures report specific limitations while working management SSH remains
  available. Healthy stages are skipped on subsequent `up` calls.
- **Test disks recover automatically.** Working filesystems are reused;
  unrecognized or confirmed damaged filesystems are reset and mounted again.
  The result explicitly reports discarded data. Probe errors, missing devices,
  busy mounts, and backend I/O errors do not trigger formatting.
- **Fewer manual fixes.** Unwritable shares fall back to read-only and retry after
  permissions are corrected. Occupied automatic SSH ports are reassigned for
  stopped guests. Interrupted image transfers resume, and official repositories
  can fail over without bypassing digest verification.
- **Clearer output and state.** Successful commands show a short summary;
  limitations are grouped, and JSON/YAML keep structured results. Guest warnings
  use a disposable cache outside the schema-2 VM documents and node artifact
  directories, so older releases can still read and manage the deployment.

## Upgrade notes

**Data disks are disposable test storage. `up` may clear a damaged filesystem,
including one marked `persistent`.** Persistence retains a disk across VM
destroy/recreate; it does not preserve corrupt filesystem contents during
recovery. Keep valuable data outside these test disks. Root disks and host
shared directories are not reset by this recovery.

There is no separate `repair` command. Repeating `up` performs recovery;
`start` continues to power on existing nodes without applying an inventory.
`--no-wait` skips readiness and these guest recovery checks.

A usable guest with optional limitations returns exit 0. Automation that needs
all configured features must inspect `nodes[].warnings` in JSON/YAML. Recovery
actions, including data resets, appear in `nodes[].repairs`. Downgrading the CLI
does not restore discarded data or revert installed guest helpers.

A pre-existing 0.6.0 bootstrap failure can already have deleted the staged
control-node SSH key before installing it. `up` restores management access and
independent setup, but reports `control-ssh` if that key is missing. It does not
reinject private keys during an in-place retry. After reviewing disk effects with
`farrow plan`, explicitly recreate the affected control node if peer SSH is needed.

## Install or upgrade

```bash
curl -fLO https://github.com/pgsty/farrow/releases/download/v0.7.0/install.sh
chmod +x install.sh
FARROW_VERSION=0.7.0 ./install.sh
farrow version
farrow up
```

The release includes macOS/Linux amd64/arm64 archives, Linux DEB/RPM packages,
the installer, checksums, SBOMs, and a Homebrew formula. It follows the existing
pre-1.0 GitHub pre-release policy; specify `FARROW_VERSION` when installing.

## Validation

The complete source gate covers unit and race tests, static checks, four-target
compilation, installer and image-pipeline checks, and licenses. Isolated Ubuntu
amd64/KVM tests with Ubuntu 24.04 guests exercised disk corruption and resets,
busy mounts, failed probes, retained-disk recreation, share recovery, and repeated
healthy `up` calls without process replacement. macOS validation includes CLI
smoke checks and cross-compilation; this release does not claim a new HVF guest
replay, host reboot test, or full Pigsty installation.
