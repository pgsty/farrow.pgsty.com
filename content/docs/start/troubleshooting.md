---
title: Troubleshooting
description: Short, safe runbooks for setup, networking, images, drift, interrupted state, and SSH.
weight: 30
icon: fa-solid fa-life-ring
---

Start read-only:

```bash
farrow doctor --json
farrow network status --json
farrow status --json
```

## No inventory found

For the first deployment, run `plan`, `up`, or `validate` beside
`farrow.yml`/`pigsty.yml`, pass `-f /path/to/file`, or run `farrow init` to
write one. Once state exists, `plan`, `up`, `reload`, and `recreate` can fall
back to its applied spec. Status, start, stop, SSH, and destroy always use
applied state. If `status` reports `no deployment state found`, the selected
`FARROW_HOME` has never run `up`.

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

Apply it with `--yes` only when it names Farrow-owned paths. A failed Linux
bridge smoke test rolls the install back automatically; an explicit
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

`recreate` means the node's definition changed: review it with `farrow plan`,
then run `farrow recreate <node>`. On a terminal the command asks you to type
`recreate`; `--force` is for scripts. `missing` is only a report: restore the
host entry or run `farrow destroy <node>`.

## A node did not become ready

`up`, `start`, and `recreate` wait for each guest to finish its bootstrap. When
some nodes fail, the command exits 5 and reports
`N of M node(s) failed: <node> (<stage>: <error>)`. Stages are `prepare`,
`start`, `readiness`, and `stop`. A `readiness` failure names the guest's own
bootstrap stage:

| Bootstrap stage | Usually means |
|---|---|
| `identity`, `hosts` | the image or its cloud-init did not run as expected |
| `management-network` | the host has no egress, or a proxy is required |
| `data-disks` | a disk definition is wrong, or the image lacks the requested `mkfs` |
| `shares` | a shared directory could not be mounted |
| `control-ssh` | the control-node SSH key could not be installed |
| `private-network` | the fixed-IP interface never came up; check `farrow network status` |
| `ready` | the ready marker could not be written |

`guest bootstrap failed during data-disks (exit status 1)` names the failing
stage; `guest did not become ready within 3m0s: <last ssh error>` means the
guest never answered. Read the guest console and QEMU output, then check the
recorded state:

```bash
farrow logs <node>                  # serial console
farrow logs <node> --source qemu    # QEMU diagnostics
farrow status
```

Nodes that failed during `prepare` never joined the deployment. Pass
`--rollback` to `up` or `reload` to remove their leftover artifacts in the
same run; structured output lists them under `rolled_back`. `--no-wait` skips
the readiness wait and returns once QEMU is running.

## SSH fails

Check `farrow status`, `farrow ssh-config`, and the serial log. Farrow's own
SSH uses a loopback management port; direct Ansible traffic uses the fixed IP.
`doctor` excludes fixed IPs reserved by the applied deployment from its generic
eligibility scan; `up` and `start` still reject a new or stopped node address
that already accepts SSH.

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
