---
title: Image Repositories
description: Choose guest images, use a mirror, import a local qcow2, and prune the cache.
weight: 40
icon: fa-solid fa-box-archive
---

Normal use needs no image command first: `farrow up` automatically pulls and
verifies the default `u24` image.

## Choose an image

Inspect the available aliases:

```bash
farrow image list
farrow image info u24
```

Built-in families are `el7`, `el8`, `el9`, `el10`, `d12`, `d13`, `u22`,
`u24`, and `u26`. Select one with `vm_image` in the inventory:

```yaml
all:
  vars:
    vm_image: el9
```

Run `farrow up` again. New nodes use the selected image; existing nodes are
never rebuilt implicitly. A definition change requires an explicit
`farrow recreate <node>`.

> [!WARNING]
> Built-in images are currently `testing`; EOL `el7` is `deprecated`.
> Successful download and digest verification prove artifact integrity, not
> production support.

## Use a mirror

Select a repository for one command:

```bash
farrow image pull --repo https://mirror.example/farrow u24
farrow up --repo https://mirror.example/farrow
```

Or set the default repository for the current shell:

```bash
export FARROW_REPO=https://mirror.example/farrow
farrow up
```

`FARROW_REPO` may also be an absolute local directory. Farrow still verifies
the signed Catalog, size, SHA-256, and qcow2 structure. An unavailable explicit
repository is an error.

## Import and prune

An independently obtained qcow2 requires an independently obtained digest:

```bash
farrow image import --name local-mybase --boot uefi \
  --source-user ubuntu --sha256 <digest> /path/to/base.qcow2
```

Custom aliases must begin with `local-`. Review the plan before deleting cache
entries not referenced by the applied deployment:

```bash
farrow image prune --dry-run
farrow image prune --yes
```

See [Images](../../reference/images/) for signatures, rollback protection,
cache layout, architecture, and TCG rules. See the
[Image Pipeline](../../reference/image-pipeline/) for building and publishing
image candidates.
