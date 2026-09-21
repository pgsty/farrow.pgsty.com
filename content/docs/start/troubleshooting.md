---
title: Troubleshooting
description: Short, safe runbooks for setup, networking, images, drift, interrupted state, and SSH.
weight: 30
icon: fa-solid fa-life-ring
---

Start with diagnostics (`status` may reconcile interrupted runtime state):

```bash
farrow doctor --json
farrow network status --json
farrow status --json
```

## Download and PATH problems

The installer uses GitHub Release assets; `--mirror` selects the Farrow image
repository and does not redirect installer downloads. If your network needs a
proxy, set `HTTPS_PROXY` or `ALL_PROXY` in the terminal to your existing proxy's
address. A macOS system proxy setting alone does not configure these environment
variables for command-line tools.

The user-scoped installer defaults to `~/.local/bin`. If `farrow` is missing or
reports an older version after installation, check which executable is selected:

```bash
export PATH="$HOME/.local/bin:$PATH"
command -v farrow
farrow version
```

For Homebrew or native packages, use that channel's executable instead. Keep the
CLI and its packaged `farrow-hosts-helper` from the same release together.

## No inventory found

Interactive `up` can create the first default inventory when no deployment
exists. For an explicit configuration, run `plan`, `up`, or `validate` beside
`farrow.yml`/`pigsty.yml`, pass `-f /path/to/file`, or run `farrow init` to
write one. Once state exists, `plan`, `up`, `reload`, and `recreate` can fall
back to its applied spec. Status, start, stop, SSH, and destroy always use
applied state. If `status` reports `no deployment state found`, the selected
`FARROW_HOME` has no applied deployment; it may be fresh or previously purged.

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

`plan` resolves the intended runtime without QEMU installed. `up` and `recreate`
check the selected emulator and firmware before changing VM resources.
Performance results from TCG are not meaningful.

## Network is partial or invalid

An intact but inactive Farrow network can be restored by interactive `up`.
For partial or invalid installations, do not delete host files by hand; review
the owned cleanup plan:

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

Management SSH and guest instance identity are required for readiness. If a node
cannot be created, started, or reached, the operation reports the node and stage
and exits 5. Read its logs:

```bash
farrow logs <node>                  # serial console
farrow logs <node> --source qemu    # QEMU diagnostics
farrow status
```

Data disks, shares, hostnames, guest hosts, control-node SSH, and private-network
setup run independently. Failure of one does not prevent management SSH or the
other stages. A usable guest returns 0 with specific limitations; JSON/YAML
expose them as `nodes[].warnings`. Internet access is not a readiness requirement.

| Limitation | Next action |
|---|---|
| Data disk unavailable | Correct a missing device, probe, tool, busy mount, or I/O problem, then run `up` |
| Shared directory is read-only | Correct host permissions, then run `up` to retry writes |
| Guest hosts or control-node SSH incomplete | Run `up` to refresh the managed files |
| Private interface unavailable | Check `farrow network status`, then run `up`; management SSH can still work |

Repeat `up` after fixing the underlying issue. It retries unfinished stages,
upgrades old guest helpers in place, and skips healthy work without restarting
running VMs. Unrecognized or confirmed damaged test data filesystems are reset
automatically, **including persistent disks**; the result reports discarded data.
Failed probes, busy mounts, and I/O failures do not trigger formatting. See
[Data disks](../../reference/configuration/#data-disks).

After an interrupted 0.6.0 bootstrap, the staged control-node SSH key may already
be missing. In that case `up` restores management access but cannot reinject
the key in place. If peer SSH is required, review `farrow plan` and recreate
the affected control node; see the [0.7.0 upgrade notes](../../../blog/release/farrow-0.7.0/).

A repeated `up` can also clean recognized leftovers from interrupted preparation.
`--rollback` removes failed prepare artifacts in the same run and lists them in
`rolled_back`. `--no-wait` returns once QEMU is running and skips readiness,
guest recovery, and metadata refresh; a later `up` completes them.

## SSH fails

In Farrow 0.8, startup restores a missing deployment public
key from the intact original private key. It also covers VMs created with 0.7.0.
If the private key is missing, restore that same key from backup; Farrow refuses
to generate a replacement identity for existing VMs. This host-side recovery is
separate from the old control node's missing guest key described above.
`up` now checks that installed guest key as well: a missing copy is a
`control-ssh` limitation, not a management SSH failure. Restoring the original
guest key and running `up` clears the limitation. Automatic private-key
reinjection into old guests remains pending.

Check `farrow status`, `farrow ssh-config`, and the serial log. Farrow's own
SSH uses a loopback management port; direct Ansible traffic uses the fixed IP.
If another process occupies a stopped VM's automatically allocated management
port, the next start selects a free port and refreshes its SSH aliases. Running
VM ports stay unchanged. SSH host-key trust is scoped to the VM instance UUID,
so recreating a VM does not require deleting unrelated known-host entries.
A changed key for the same instance still fails verification.

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
