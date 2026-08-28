---
title: Troubleshooting
description: Short, safe runbooks for setup, networking, images, drift, interrupted state, and SSH.
weight: 40
icon: fa-solid fa-life-ring
---

Start read-only:

```bash
farrow doctor --json
farrow network status --json
farrow status --json
```

## No configuration found

For the first deployment, run `plan`, `up`, or `validate` beside
`farrow.yml`/`pigsty.yml`, or pass `-f /path/to/file`. Once state exists,
`plan`, `up`, `reload`, and `recreate` can fall back to its applied spec.
Status, start, stop, SSH, and destroy always use applied state. If `status`
prints the same message, the selected `FARROW_HOME` has no applied state.

## Setup needs sudo

The line before the prompt names the exact host mutation. Farrow attaches an
interactive terminal directly to sudo when the privileged step begins.
Automation needs an existing credential or a suitable NOPASSWD policy, then
passes `--yes`.

## Native acceleration or compatibility runtime is unavailable

Native paths require HVF on macOS or KVM on Linux. TCG is selected only for an
explicit foreign `vm_arch` or a built-in image/host compatibility rule; an
arbitrary native failure never falls back. Homebrew QEMU contains both system
emulators. Linux setup installs only the native family, so a foreign Guest also
requires its matching `qemu-system-*` binary and firmware.

`plan`, `up`, and `recreate` validate the selected emulator and firmware before
any destructive mutation. Performance results from TCG are not meaningful.

## Network is partial or invalid

Do not delete host files by hand. Review the owned cleanup plan:

```bash
farrow network status --json --verbose
farrow network uninstall
```

Apply it with `--yes` only when it names Farrow-owned paths. Linux bridge
smoke failures now trigger automatic manifest-bounded rollback; an explicit
`automatic rollback failed` message means manual inspection is required.

## Linux bridge helper fails

```bash
id
stat -c '%U:%G %a %n' /usr/lib/qemu/qemu-bridge-helper
dpkg-statoverride --list /usr/lib/qemu/qemu-bridge-helper
```

Debian/Ubuntu uses `root:<caller-accessible-group> 4750`; the caller does not
have to belong to `kvm` when `/dev/kvm` access comes from a desktop ACL.

## Plan reports recreate or missing

`recreate` needs `farrow recreate <node> --force`. `missing` is only a report:
restore the host entry or run `farrow destroy <node> --force`.

## SSH fails

Check `farrow status`, `farrow ssh-config`, and the serial log. Farrow's own
SSH uses a loopback management port; direct Ansible traffic uses the fixed IP.

## Catalog or image verification fails

The current binary embeds active and standby Catalog public keys. Unknown
signers, version rollback/equivocation, artifact size/SHA mismatch, and unsafe
qcow2 structure are distinct integrity failures. Use a correctly signed
repository or `farrow image import --sha256 ...`; do not copy bytes directly
into `~/.farrow/images`.

## A command was killed

Run `farrow status`. A provably live or dead runtime is converged from its
recorded identity; an ambiguous process remains blocked. Do not kill an
unknown PID based only on a state file.

If a recorded QEMU process still exists but its QMP socket is absent, preserve
the evidence and inspect serial/QEMU logs before using `stop` to converge it.
Do not delete runtime sockets or state files by hand.

For a bug report include the exact command and exit code, `farrow version`,
the three JSON reports above, host OS/architecture, and QEMU version.
