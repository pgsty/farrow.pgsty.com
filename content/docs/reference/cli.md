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

In text mode, bare `farrow` and namespaces such as `farrow image` display
contextual help and exit successfully. In JSON/YAML mode a bare namespace is a
structured usage error; explicit `--help` always renders human help.

## Commands

| Area | Commands |
|---|---|
| Prepare | `setup`, `init`, `validate`, `doctor` |
| Lifecycle | `plan`, `up`, `start`, `stop`, `restart`, `reload`, `recreate`, `status`, `destroy` |
| Access | `ssh`, `exec`, `logs`, `provision`, `ssh-config`, `ss`, `hosts` |
| Images | `image list/info/pull/import/sync/prune/reset-manifest` |
| Host network | `network status/install/uninstall` |
| Misc | `version`, `completion` |

Frequently used commands have scoped aliases:

| Command | Aliases | Command | Aliases |
|---|---|---|---|
| `setup` | `s` | `init` | `i` |
| `validate` | `v` | `plan` | `pl` |
| `recreate` | `rc` | `status` | `st` |
| `destroy` | `de` | `provision` | `p` |
| `ssh-config` | `sc` | `hosts` | `h` |
| `image` | `images`, `im` | `doctor` | `dt` |
| `network` | `n`, `net` | `exec` / `logs` | `ex` / `l` |
| `version` | `ver` | `completion` | `cp` |

`up`, `ssh`, and `ss` are already short. `start`, `stop`, `restart`, and
`reload` deliberately have no advertised aliases; the old `halt` entry remains
as a hidden deprecated compatibility command. Inside a namespace, `hosts` and
`network` use `i`/`u` for install/uninstall; `network status` uses `st`; and
`image` maps `list=ls`, `info=in`, `pull=p`, `prune=pr`, `sync=sy`,
and `import=i`. Safety-sensitive `reset-manifest` stays unabridged.

Commands using applied state work from any directory. Configuration selection
is command-scoped; `-f` is deliberately not a global flag:

| Commands | Desired-state source |
|---|---|
| `setup [template]` | explicit `-f`, otherwise discovery, otherwise generate `meta`; template and `-f` are exclusive |
| `init [template]` | generate a new inventory; read no desired state; `-f/--force` explicitly replaces the output |
| `validate` | explicit `-f`, then discovery; never applied state |
| `plan`, `up`, `reload`, `recreate` | explicit `-f`, then discovery, then the applied resolved specification |
| other lifecycle/access commands | no desired-state inventory; use applied or marker-owned state as applicable |

## Important flags

| Flag | Meaning |
|---|---|
| `--json`, `--yaml` | machine-readable stdout; progress remains on stderr; intentionally no shorthand |
| `-v`, `--verbose` | bounded diagnostics on stderr |
| `-c`, `--cidr` | select the RFC1918 `/24` for generated `init`/`setup` templates or host-network inspection/install |
| `-f`, `--file` | select an Inventory for commands that read desired state |
| `-r`, `--repo` | choose an image/artifact repository for commands that resolve downloads |
| `-m`, `--mode` | select `host` or `shared` where the command exposes the macOS network mode |
| `-d`, `--dry-run` | show a setup/image plan without changing state |
| `-y`, `--yes` | apply a displayed host/setup/image plan |
| `-f`, `--force` (`init`, `destroy`) | overwrite generated output or skip destroy confirmation |
| `--force` (`recreate`) | skip recreate confirmation; long-only because `-f` selects the Inventory |
| `-n`, `--no-wait` | return after QMP/process identity without guest readiness |
| `--delete-persistent` | during whole destroy, also delete retained data disks; invalid with node selectors |
| `--purge` | whole-deployment disposal: delete disks, keys, and deployment state; keep images |

Rare or safety-widening controls such as `--allow-downgrade`, `--sudo`,
`--delete-persistent`, and `--purge` intentionally remain long-only. Across the
CLI, `-d` therefore consistently means a dry run.

If a failing command has not already emitted a richer typed result, structured
mode writes one object containing `error` and `message` before returning the
documented non-zero exit code. Existing typed failure results are never followed
by a second JSON/YAML document.

`plan` is read-only and returns success even when its action is `recreate` or
`blocked-removal`; automation must inspect the action and `create`, `recreate`,
and `missing` fields. `up` creates additions, starts selected stopped nodes, and
regenerates the default marker-owned SSH configuration from the complete
applied deployment. `recreate` performs the same full refresh; node destroy
removes stale entries, and whole destroy removes the default integration.
`start` only powers on already-created nodes. Destructive drift still returns a
conflict instead of being applied implicitly. If VM lifecycle succeeds but SSH
reconciliation fails, structured output reports the VM status as a partial
`ssh_config` failure and exits 7.

`status` reports the persisted `guest_arch` and `accelerator` for each node.
TCG selection is therefore explicit in both text and structured output.

## SSH passthrough and completion

`farrow ssh [node] [--] [command ...]` opens a session or runs an optional
command. `farrow exec [node] [--] <command ...>` requires a command and passes
through its exit status. Presentation flags before `--` belong to Farrow;
arguments after `--` belong to OpenSSH or the remote program.

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
| 5 | partial multi-node completion |
| 6 | resource conflict |
| 7 | integrity or ownership failure |

`ssh` and `exec` pass through the remote program's exit code, except SSH's
reserved transport-failure code 255, which Farrow maps to runtime failure 1.
