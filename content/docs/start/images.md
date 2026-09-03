---
title: Image Repositories
description: Choose guest images, use a mirror, import a local qcow2, and prune the cache.
weight: 40
icon: fa-solid fa-box-archive
---

Normal use needs no image command first: `farrow up` resolves `d13:stable` for
the native host architecture and pulls the resulting immutable version.
Farrow uses the Catalog embedded in the installed build until you run
`farrow update`, which fetches, verifies, and activates the repository's current
Catalog. Nothing refreshes it automatically; `image sync` explicitly activates
an exact URL or file for recovery.

## Choose an image

Inspect the available aliases:

```bash
farrow image list
farrow image info d13
farrow image info d13:stable
```

Built-in families are `el7`, `el8`, `el9`, `el10`, `d12`, `d13`, `u22`,
`u24`, and `u26`. A bare name selects `stable`; `name:channel` selects a
channel. An exact `name@version` key wins; a shorter numeric selector chooses
the newest matching version on dot-component boundaries:

```yaml
all:
  vars:
    vm_image: el9
    vm_version: 9.7
```

Here `9.7` selects the newest 9.7.x build; `9` selects the newest 9.x release.
Use `vm_image: el9:stable` instead when the repository's movable stable channel
is the intended policy.

Run `farrow up` again. New nodes use the selected image; existing nodes are
never rebuilt implicitly. A definition change requires an explicit
`farrow recreate <node>`.

> [!WARNING]
> Built-in versions are `supported` except deprecated compatibility images:
> EOL `el7`, EL9 9.3/9.6, and EL10 10.0.

## Use a mirror

Released builds use `https://repo.pigsty.io/farrow` by default. Select the
official China repository for one command with long-only `--mirror`, or name a
custom root with `--repo`:

```bash
farrow image pull d13 --mirror
farrow up --mirror
farrow image pull d13 --repo https://mirror.example/farrow
farrow up --repo https://mirror.example/farrow
```

Or set the default repository for the current shell:

```bash
export FARROW_REPO=https://mirror.example/farrow
farrow up
```

Selection precedence is `--repo`, `--mirror`, `FARROW_REPO`, then the global
default. `--mirror` resolves to `https://repo.pigsty.cc/farrow` and both
official roots retain canonical signed-Catalog trust. `FARROW_REPO` may also be
an absolute local directory. Explicit local and HTTPS repositories may use an
unsigned Catalog; HTTP repositories require a Catalog signed by a trusted key.
Artifact size, SHA-256, and qcow2 structure are always verified.

An unavailable repository only matters when an artifact must be downloaded.
Catalog upstream URLs are provenance, not a hidden fallback: a missing artifact
in the selected repository is an error. `farrow update` fetches that
repository's Catalog; `image sync` fetches or reads the exact source supplied
by the user.

## Build a static repository

A repository is an ordinary directory that can be copied with `rsync` or served
by a static HTTP server:

```text
farrow/
├── repo.yaml
├── catalog.json
├── catalog.json.minisig       # optional
└── images/
    └── d13-1-arm64.qcow2
```

`repo.yaml` is the only human-maintained source. `catalog.json` is generated:

```bash
farrow repo scan /srv/farrow
farrow repo build /srv/farrow
farrow repo verify /srv/farrow
```

Scan is read-only. Build never changes `repo.yaml` or image bytes; it performs a
full `qemu-img check` and materializes file names, SHA-256, artifact size, and
virtual size. `build` and `verify` require local `qemu-img`; `scan` does not.
Build on a machine with QEMU, then publish immutable QCOW files first and
`catalog.json` last.

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
