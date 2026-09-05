---
title: CLI
description: Farrow commands, important flags, structured output, and exit codes.
weight: 20
icon: fa-solid fa-terminal
aliases: [/docs/reference/json-api/]
---

```text
farrow [--json|--yaml] [-v|--verbose] <command> [flags] [node...]
```

The installed binary is the authoritative reference for its own version. Every
visible command includes its operational boundary and copyable examples:

```bash
farrow --help
farrow setup --help
farrow image pull --help
```

Bare `farrow` prints its help and exits 2; a bare namespace such as
`farrow image` does the same. In JSON/YAML mode a bare namespace is a
structured usage error; explicit `--help` always renders human help.

## Commands

| Area | Commands |
|---|---|
| Prepare | `setup`, `init`, `validate`, `doctor` |
| Lifecycle | `plan`, `up`, `start`, `stop`, `restart`, `reload`, `recreate`, `status`, `destroy`, `purge` |
| Access | `ssh`, `exec`, `logs`, `provision`, `ssh-config`, `hosts install/uninstall` |
| Images | `update`, `image list/info/pull/import/sync/prune/reset`, `repo scan/build/verify` |
| Host network | `network status/install/uninstall` |
| Misc | `version`, `completion` |

No command refreshes the Catalog implicitly. `update` fetches the configured
repository's Catalog, verifies it, and activates it; `image sync` is the
explicit recovery path for an exact URL or file. Ordinary commands use the
active local Catalog. Neither operation updates the Farrow executable.

Frequently used commands have scoped aliases:

| Command | Aliases | Command | Aliases |
|---|---|---|---|
| `setup` | `s` | `validate` | `v` |
| `plan` | `pl` | `recreate` | `rc` |
| `status` | `st` | `destroy` | `de` |
| `ssh-config` | `sc` | `image` | `images`, `im` |
| `doctor` | `dt` | `network` | `n`, `net` |
| `exec` / `logs` | `ex` / `l` | `version` | `ver` |
| `purge` | `rm` |  |  |

`up`, `ssh`, `init`, `start`, `stop`, `restart`, `reload`, `provision`,
`hosts`, and `completion` have no aliases. `purge` uses `rm` as its explicit
whole-deployment disposal alias. Inside a namespace, `hosts` and
`network` use `i`/`u` for install/uninstall; `network status` uses `st`; and
`image` maps `list=ls`, `info=in`, `pull=p`, `prune=pr`, `sync=sy`,
and `import=i`. `image reset` keeps `reset-manifest` as a compatibility alias.

Commands using applied state work from any directory. Configuration selection
is command-scoped; `-f` is deliberately not a global flag:

| Commands | Desired-state source |
|---|---|
| `setup [template]` | explicit `-f`, otherwise discovery, otherwise generate `meta`; template and `-f` are exclusive |
| `init [template]` | generate a new inventory; read no desired state; `--force` explicitly replaces the output |
| `validate` | explicit `-f`, then discovery; never applied state |
| `plan`, `up`, `reload`, `recreate` | explicit `-f`, then discovery, then the applied resolved specification |
| other lifecycle/access commands | no desired-state inventory; they use applied state |

## Important flags

| Flag | Meaning |
|---|---|
| `--json`, `--yaml` | machine-readable stdout; progress remains on stderr; intentionally no shorthand |
| `-v`, `--verbose` | bounded diagnostics on stderr |
| `-c`, `--cidr` | select the RFC1918 `/24` for generated `init`/`setup` templates or host-network inspection/install |
| `-f`, `--file` | select an Inventory for commands that read desired state |
| `-r`, `--repo` | choose an image/artifact repository; overrides `--mirror` and `FARROW_REPO` |
| `--mirror` | use the China official repository for setup, Catalog, and image-resolving lifecycle commands |
| `-m`, `--mode` | select `host` or `shared` where the command exposes the macOS network mode |
| `-d`, `--dry-run` | show a setup/image plan without changing state |
| `-y`, `--yes` | apply a displayed host/setup/image plan |
| `--force` (`init`, `destroy`, `recreate`) | overwrite generated output or skip the typed confirmation; long-only because `-f` selects the Inventory |
| `-n`, `--no-wait` | return once QEMU is running, without waiting for the guest to boot |
| `--rollback` (`up`, `reload`) | remove the prepare artifacts of nodes that failed to prepare in this run |
| `--delete-persistent` | during whole destroy, also delete retained data disks; invalid with node selectors |
| `--purge` | whole-deployment disposal: delete disks, keys, and deployment state; keep images |

Rare, selection, or safety-widening controls such as `--mirror`, `--force`,
`--rollback`, `--remove`, `--allow-downgrade`, `--sudo`,
`--delete-persistent`, and `--purge` are
long-only. On commands that read an Inventory, `-f` always selects a file;
`logs -f` retains the conventional `--follow`. `-n` always means `--no-wait`,
and `-d` always means a dry run.

`farrow purge` (alias `farrow rm`) is the no-confirmation shortcut for
`farrow destroy --force --purge`. It accepts no nodes or Inventory, removes the
complete deployment plus persistent disks, keys, state, and the default SSH
fragment, and keeps images and the host network. With no deployment it succeeds
without changing the image cache. Missing state never authorizes deletion of
residual node artifacts whose identity cannot be proven.

If a failing command has not already emitted a richer typed result, structured
mode writes one object containing `error` and `message` before returning the
documented non-zero exit code. Existing typed failure results are never followed
by a second JSON/YAML document.

`plan` is read-only and returns success even when its action is `recreate` or
`blocked-removal`; automation must inspect the action and `create`, `recreate`,
`start`, `missing`, and `blocked` fields. Plans only read local configuration
and catalog data, so QEMU and host networking need not be installed. They show
exact images, total resources, change reasons, and disk effects. `up` checks
host capabilities and address availability before applying changes. `up` creates missing nodes, starts stopped ones,
re-checks readiness of running ones, and rewrites the SSH client configuration
Farrow installed from the complete applied deployment. `recreate` performs the
same full refresh; node destroy removes stale entries, and whole destroy
removes that configuration. `start` powers on stopped nodes and re-checks
readiness of running ones; it does not touch the SSH client configuration.
Destructive drift returns a conflict that names the next commands:
`farrow plan`, then `farrow recreate <node>` or `farrow destroy <node>`. On a
terminal those commands ask you to type the confirmation word; `--force` is for
scripts. If VM lifecycle succeeds but the SSH client configuration cannot be
written, structured output reports the VM status as a partial `ssh_config`
failure and exits 5.

A multi-node operation in which some nodes failed exits 5 and reports
`N of M node(s) failed: <node> (<stage>: <error>); ...`. Stages are `prepare`,
`start`, `readiness`, `stop`, `status`, and `guest-metadata`; a `readiness` failure adds
`run \`farrow logs <node>\` for the guest console`. Structured output carries
`failures[]` with `node`, `stage`, and `error`, plus `rolled_back` when
`--rollback` removed the prepare artifacts of nodes that never committed. See
[A node did not become ready](../../start/troubleshooting/#a-node-did-not-become-ready).

`status` shows node, state, IP, exact image, and CPU/memory. Use `--verbose` for
SSH ports, architecture, accelerator, and PID. TCG is marked in ordinary text
as well. One degraded node does not hide its peers; status exits 5 and retains
per-node errors and `failures[]`. Running means the VM process is running;
status does not claim to have checked guest readiness.

Starting commands also refresh Farrow hosts and control-node SSH entries in
running guests. Stopped guests catch up when started. `--no-wait` skips guest
readiness and that refresh; a later `up` completes both. Selected recreate
refuses remaining peer drift before stopping or deleting disks; select the
required nodes together as directed.

The control guest's Farrow-managed SSH entries accept replacement host keys
without recording them in known_hosts, so recreated lab nodes remain reachable.
User-added SSH entries are preserved.

## SSH passthrough and completion

`farrow ssh [node] [--] [command ...]` opens a session or runs an optional
command. `farrow exec [node] [--] <command ...>` requires a command and passes
through its exit status. Presentation flags before `--` belong to Farrow;
arguments after `--` are joined with spaces and interpreted by the remote
shell, like plain SSH. Before `--`, only zero or one known node is accepted.
For convenience, omitting `--` uses a known first argument as the node, or
runs all arguments as a command on the default node with a warning. Use an
explicit `--` in scripts.

Load `farrow completion bash|zsh|fish|powershell` for command and scoped-flag
completion. It also provides command aliases, templates, image aliases, closed
flag choices, and best-effort node names from the desired or applied
specification.

## Exit codes

| Code | Meaning |
|---:|---|
| 0 | success |
| 1 | runtime failure |
| 2 | usage or invalid configuration |
| 3 | missing host capability |
| 4 | state conflict or explicit convergence required |
| 5 | partial completion across nodes or post-lifecycle integration |
| 6 | resource conflict |
| 7 | integrity or ownership failure |
| 130 | interrupted (SIGINT/SIGTERM) |

`ssh` and `exec` pass through the SSH child exit code unchanged, including
255. That value may indicate an SSH connection failure or a remote command
returning 255; text, JSON, and process exit status agree.
