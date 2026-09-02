---
title: Daily Operations
description: "The normal lifecycle for the one deployment: inspect, access, scale, change, stop, and destroy."
weight: 20
icon: fa-solid fa-gears
aliases: [/docs/start/lifecycle/, /docs/start/provisioning/]
---

## Inspect and access

```bash
farrow status
farrow ssh meta
farrow exec node-1 -- hostname
farrow logs meta --source serial
```

Applied state is under `~/.farrow`; these commands work from any directory.
Status includes the persisted Guest architecture and accelerator, so TCG is
never an invisible fallback.
`farrow up` rebuilds the default SSH aliases from the complete applied
deployment after the selected VMs are started, so a scoped `up` never drops
unselected peers and plain `ssh meta` just works; `farrow ssh-config --install`
rewrites it by hand if you ever need to.
`plan`, `up`, `reload`, and `recreate` prefer `-f`, then a discovered
Inventory, then the applied spec when no file exists. `validate` always needs
a file.

## Stop and start

```bash
farrow stop
farrow start
farrow restart node-1
farrow reload -f farrow.yml       # stop, re-read config, converge
```

`start` powers on stopped VMs and re-checks readiness of running ones; it does
not refresh the SSH client configuration. `restart` uses applied state. `reload` reads the Inventory again and follows
the complete `up` path after stopping.

## Change the deployment

```bash
farrow plan
farrow up                         # create/start selected nodes and install SSH aliases
farrow recreate node-1            # applies a changed VM definition
```

`recreate` and `destroy` ask you to type the confirmation word on a terminal;
pass `--force` only in scripts.

Inventory changes fall into three visible fields:

| Field | Meaning | Action |
|---|---|---|
| `create` | desired node has no state | `farrow up` |
| `recreate` | VM definition changed | `farrow recreate <node>` |
| `missing` | stateful node left the file | restore it, or destroy it explicitly |

Deleting YAML never deletes a VM. Unconsumed Pigsty changes produce
`action:none`; native naming and node-admin fields are consumed even though
they do not begin with `vm_`. Successful recreate refreshes the complete SSH
fragment as well.

## Destroy

```bash
farrow destroy node-3
farrow destroy
farrow destroy --delete-persistent
farrow destroy --purge
```

`--delete-persistent` and `--purge` are valid only for whole-deployment
destroy, not with node selectors. `--purge` removes persistent disks, keys,
and deployment state; images remain cached. Node destroy refreshes the SSH
fragment for remaining peers, while whole destroy removes the default Farrow
SSH integration. Host network removal is separate
and refuses while a VM is attached:

```bash
farrow network uninstall --yes
```

See [Image Repositories](../images/) for image selection, mirrors, and cache
pruning, or [Uninstall and Clean Up](../uninstall/) to remove host state.
