---
title: Operations
description: "The normal lifecycle for the one deployment: inspect, access, scale, change, stop, and destroy."
weight: 30
icon: fa-solid fa-gears
aliases: [/docs/start/lifecycle/, /docs/start/provisioning/, /docs/start/images/]
---

## Inspect and access

```bash
farrow status
farrow ssh meta
farrow exec node-1 -- hostname
farrow logs meta --source serial
farrow ss                         # install SSH aliases; then: ssh meta
```

Applied state is under `~/.farrow`; these commands work from any directory.
Status includes the persisted Guest architecture and accelerator, so TCG is
never an invisible fallback.
`plan`, `up`, `reload`, and `recreate` prefer `-f`, then a discovered
Inventory, then the applied spec when no file exists. `validate` always needs
a file.

## Stop and start

```bash
farrow stop                       # alias: halt
farrow start
farrow restart node-1
farrow reload -f farrow.yml       # stop, re-read config, converge
```

`restart` uses applied state. `reload` reads the Inventory again.

## Change the deployment

```bash
farrow plan
farrow up                         # create additions and start stopped selected nodes
farrow recreate node-1 --force    # applies a changed VM definition
```

Inventory changes fall into three visible fields:

| Field | Meaning | Action |
|---|---|---|
| `create` | desired node has no state | `farrow up` |
| `recreate` | VM definition changed | `farrow recreate <node> --force` |
| `missing` | stateful node left the file | restore it, or destroy it explicitly |

Deleting YAML never deletes a VM. Unconsumed Pigsty changes produce
`action:none`; native naming and node-admin fields are consumed even though
they do not begin with `vm_`.

## Destroy

```bash
farrow destroy node-3 --force
farrow destroy --force
farrow destroy --force --delete-persistent
farrow destroy --force --purge
```

`--delete-persistent` and `--purge` are valid only for whole-deployment
destroy, not with node selectors. `--purge` removes persistent disks, keys,
and deployment state; images remain cached. Host network removal is separate
and refuses while a VM is attached:

```bash
farrow network uninstall --yes
```

## Images

```bash
farrow image list
farrow image info u24
farrow image pull u24
farrow image prune --dry-run
```

Images are signed-catalog entries and SHA-256 checked before use. Current
images are `testing`, except EOL EL7 which is `deprecated`, so starts print the
corresponding warning.
