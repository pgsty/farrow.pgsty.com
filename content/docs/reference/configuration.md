---
title: Configuration
description: The Pigsty-compatible Inventory fields Farrow reads, their defaults, and node-level drift behavior.
weight: 10
icon: fa-solid fa-file-code
aliases: [/docs/concepts/project-model/]
---

## Discovery

Configuration lookup order is explicit `-f`, then `farrow.yml`,
`farrow.yaml`, `pigsty.yml`, and `pigsty.yaml` in the current directory. Every
name uses the same Pigsty-compatible YAML Inventory format.

The retired top-level `version:`/`nodes:` format is rejected with migration
guidance.

For `plan`, `up`, `reload`, and `recreate`, absence of a file falls back to
the applied spec when a deployment exists. `validate` has no fallback. A
configuration must be a regular non-symlink file no larger than 4 MiB.

## What Farrow reads

Farrow reads host IPs, `nodename`, `admin_ip`, `pg_cluster`, `pg_seq`,
`node_admin_username`, `node_admin_uid`, and the documented `vm_*` variables.
`admin_ip` is read from `all.vars`; it selects the control node, or the first
managed host is used. All nodes must resolve the same login username. For the
default `dba` user, an explicit `node_admin_uid` must be 88.

Everything else is opaque and cannot create drift. This means unconsumed
fields such as `pg_role`, `pg_version`, `repo_*`, and `node_packages`, not all
possible `pg_*` or `node_*` names.

Inside this namespace validation is strict: unknown `vm_*` names, wrong
types, Jinja expressions, invalid addresses, and conflicting sibling-group
values are errors.

## VM variables

| Variable | Default | Meaning |
|---|---|---|
| `vm_skip` | `false` | do not virtualize this real/external host |
| `vm_image` | `d13` | image family, channel reference, or `image@version` selector |
| `vm_version` | unset | newest numeric version matching this prefix, such as `9` or `9.7` |
| `vm_arch` | `native` | deployment-wide Guest architecture: `native`, `amd64`, or `arm64` |
| `vm_cpu` | `2` | vCPU count |
| `vm_mem` | `4096` | MiB integer, or a size such as `8GiB` |
| `vm_disk` | `64` | root disk GiB |
| `vm_disks` | `[{path: /data}]` | extra disks |
| `vm_alias` | `[]` | guest `/etc/hosts`, SSH-config, and optional host aliases |
| `vm_shares` | `[]` | QEMU 9p host-directory shares |

An empty host entry is a complete VM. A deployment contains 1–20 managed
hosts; `vm_cpu` accepts 1–256 and memory must be at least 512 MiB.

`vm_version` keeps short version intent separate from the image family:

```yaml
vm_image: el9
vm_version: 9.7
```

An exact Catalog version wins first. Otherwise Farrow matches only on a dot
component boundary and selects the numerically newest match: `9.7` resolves to
the newest `9.7.*` build, while `9` resolves to the newest 9.x release. Numeric
components are compared as integers, so 9.10 sorts after 9.9. Do not combine
`vm_version` with a `vm_image` that already contains `:channel` or `@version`.

`vm_arch` is stricter than ordinary per-host VM fields: when present it must
resolve to one value on every managed host, so define it once in `all.vars`.
Changing it is a deployment-envelope change and requires whole-deployment
recreation. Linux setup installs only the native emulator; a foreign
architecture also needs its matching `qemu-system-*` binary and firmware.

## Data disks

```yaml
vm_disks:
  - path: /data
    size: 128
    fs: auto
    persistent: false
```

`path` is the disk identity and mount point. `fs` is `auto`, `xfs`, or `ext4`;
omitting it retains the released `xfs` default. An explicitly selected `auto`
disk uses XFS when `mkfs.xfs` is available and otherwise uses ext4. Explicit
`xfs` and `ext4` never fall back, and an existing filesystem is preserved.
`persistent: true` survives normal destroy. Use `vm_disks: []` for no extra
disk.

## Shares

```yaml
vm_shares:
  - host: /absolute/owned/source
    guest: /src
    readonly: true
```

Shares must be real, caller-owned, non-overlapping directories. They are for
trusted development files, not PostgreSQL data.

## Names and addresses

Node name order: `nodename`, then `<pg_cluster>-<pg_seq>`, then
`node-<last-octet>`. Names must be unique.

All managed hosts must be in one RFC1918 `/24`: `.1` is the host, `.2`–`.8`
are reserved, and nodes use `.9`–`.254`.

## Drift

Farrow hashes each resolved node. Added hosts are created by `up`; selected
existing stopped nodes are started; running peers stay untouched. Changed VM
definitions require per-node recreate; removed hosts are reported but never
destroyed. Deployment architecture, user, or subnet changes require whole-deployment
recreation. Changing a field used to derive a node name appears as a missing
old node plus a new node, so prefer stable explicit `nodename` values.
