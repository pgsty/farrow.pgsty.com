---
title: Installation
description: Build Farrow, verify native acceleration, and understand the one-time host setup.
weight: 10
icon: fa-solid fa-download
aliases: [/docs/start/upgrade/]
---

## Supported path today

Build from the current checkout:

```bash
cd /path/to/farrow
make build
export PATH="$PWD/bin:$PATH"
farrow version
farrow doctor
```

There is no published 1.0 package; every 0.x build remains pre-1.0. Do not
substitute an unofficial binary for the matching `farrow` and
`farrow-hosts-helper` build.

The source build requires Go 1.27.x. macOS also needs an existing Homebrew
installation; Farrow can install QEMU through Homebrew but never bootstraps
Homebrew itself. Public builds start with the embedded signed Catalog and its
immutable HTTPS upstream artifacts. Use `FARROW_REPO` or `--repo` to opt into a
reachable mirror; no private development host is probed by default.

## Host requirements and dated evidence

Native HVF/KVM is the normal path. Farrow never retries an arbitrary native
failure under emulation. TCG is selected only for an explicit foreign
`vm_arch`, or for the catalogued stock EL8 arm64/Apple-Silicon incompatibility.
TCG is compatibility evidence, never performance evidence.

| Host | Last recorded evidence (2026-08-27) |
|---|---|
| macOS arm64 | verified with HVF, QEMU 11.1, and socket_vmnet |
| Ubuntu 26.04 amd64 | verified with KVM, QEMU 10.2.1, and NetworkManager |
| Rocky Linux 8.10 arm64 guest on macOS arm64 | verified with automatic TCG and full readiness |
| other Debian/Fedora/EL9 amd64 | implemented; repeat native validation before relying on it |
| macOS amd64, Linux arm64 | cross-built and unit-tested, not currently native-verified |

macOS needs Homebrew. Linux needs systemd plus either NetworkManager or
systemd-networkd. `farrow setup` installs missing supported packages through
Homebrew, APT, or DNF.

Package-manager downloads honor the configured `HTTP_PROXY`, `HTTPS_PROXY`,
`ALL_PROXY`, and `NO_PROXY` variables, including their lowercase forms.
Homebrew runs as the invoking user and inherits them directly. When APT or DNF
crosses sudo, Farrow asks sudo to preserve only those configured proxy names;
the setup plan lists the names but never their possibly credential-bearing
values.

## What setup changes

Run a dry plan first if desired:

```bash
farrow setup --dry-run
farrow setup --yes
```

Setup may install QEMU dependencies, prepare the host fixed-IP network, and
install the narrow root-owned hosts publisher. It prints each mutation and
the reason for sudo before applying anything.

On a terminal, Farrow-controlled downloads show a byte-accurate progress bar,
percentage, transfer rate, and ETA. Homebrew/APT/DNF do not expose a reliable
total-work API, so their install phase uses an animated indeterminate bar plus
elapsed time instead of inventing a percentage. Structured stdout remains
machine-readable; progress stays on stderr.

In automation use `--yes`. Non-interactive sudo works with either a cached
credential or a suitable NOPASSWD policy; every privileged command still uses
an exact absolute argv.

Setup is idempotent. A healthy repeat reuses the network and installed helper.
Source tests and this dated native matrix are separate: rebuilding a newer
working tree does not silently refresh the rows above.
