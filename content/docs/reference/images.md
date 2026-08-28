---
title: Images
description: Signed catalogs, built-in aliases, repository selection, local cache verification, imports, and pruning.
weight: 30
icon: fa-solid fa-hard-drive
---

Farrow uses a signed static-file Catalog plus immutable qcow2 artifacts. A
Catalog update does not require a new Farrow binary, but the binary decides
which signing keys and image safety rules are trusted.

> [!WARNING]
> EL7 is `deprecated`; every other built-in image is currently `testing`, not
> `supported`. The warning printed by `up` is intentional: a successful pull
> is an integrity result, not a production-support promise.

## Aliases and pull order

The embedded Catalog contains 9 families and 17 artifacts: `el7` is
amd64-only; `el8`, `el9`, `el10`, `d12`, `d13`, `u22`, `u24`, and `u26`
have amd64 and arm64 artifacts. `u24` is the VM default.

| Alias | Distribution | Architectures | Boot | Status |
|---|---|---|---|---|
| `el7` | CentOS Linux 7.9 / 2211 | amd64 | BIOS | deprecated |
| `el8` | Rocky Linux 8.10 | amd64, arm64 | UEFI | testing |
| `el9`, `el10` | Rocky Linux | amd64, arm64 | UEFI | testing |
| `d12`, `d13` | Debian | amd64, arm64 | UEFI | testing |
| `u22`, `u24`, `u26` | Ubuntu | amd64, arm64 | UEFI | testing |

```bash
farrow image list
farrow image info u24
farrow image pull u24
```

For a pull, Farrow:

1. refreshes `catalog.json` and its adjacent `.minisig` from `--repo`, then
   `FARROW_REPO`, then the compiled default repository;
2. verifies the signature with an embedded active or standby public key;
3. selects the Catalog's default release; standalone `image pull` uses the
   native architecture, while lifecycle resolution honors `vm_arch`;
4. reuses a local file only after size, SHA-256, and qcow2 checks pass;
5. otherwise downloads the repository artifact, with the Catalog's immutable
   HTTPS upstream URL as fallback.

The current compiled default is the development repository
`https://m0/farrow`; no public image service is claimed yet. An explicit
repository failure is fatal, while an unreachable compiled default falls back
to the embedded Catalog.

## Runtime policy

Matching architectures use native HVF/KVM except one catalogued incompatibility:
the stock EL8 arm64 64K-granule kernel cannot run through Apple HVF, so Apple
Silicon uses visible same-architecture TCG automatically. Explicit foreign
`vm_arch` also uses TCG. amd64-on-arm64 uses a single translation thread to
preserve x86 memory ordering. TCG results are not performance evidence.

EL7 is deliberately limited to native Linux/amd64. Linux setup installs only
the native QEMU family; foreign architectures require the matching system
emulator and UEFI firmware before `plan`, `up`, or `recreate` can proceed.

```bash
farrow image pull --repo https://mirror.example/farrow u24
FARROW_REPO=/absolute/local/repository farrow up
```

Repository URLs may be HTTP or HTTPS because the Catalog signature and image
digest remain authoritative; immutable upstream artifact URLs must be HTTPS.

## Trust and verification

Current ordinary builds embed both production public verification keys. The
private signing keys are external to the source repository. Catalog activation
rejects unknown keys, malformed content, equivocation, and versions below the
recorded high-water mark unless the operator explicitly allows a downgrade.

Every accepted image must be a size- and SHA-256-matched plain qcow2 with no
backing file, external data file, encryption, or unknown incompatible feature.
Verified base images become read-only; node root disks are overlays and never
modify the base.

```bash
farrow image sync https://repo.example/farrow/catalog.json
farrow image sync --allow-downgrade /absolute/repo/catalog.json
farrow image reset-manifest
```

`reset-manifest` restores the embedded bootstrap Catalog but keeps the
anti-rollback high-water mark.

## Local layout and imports

Images live under `FARROW_HOME/images` (default `~/.farrow/images`): family
directories contain downloaded artifacts, `manifests/` stores Catalog state,
and `local/` plus `local-images.json` hold imports. There is no old
`~/.farrow/cache` or content-addressed `sha256/` hierarchy.

```bash
farrow image import --sha256 <digest> /path/to/base.qcow2
farrow image import --name local-mybase --boot uefi \
  --source-user ubuntu --sha256 <digest> /path/to/base.qcow2
```

Named local aliases must begin with `local-`, so a future signed Catalog cannot
shadow them. `--boot` and `--source-user` are required with `--name`; Farrow
does not guess the guest bootstrap contract.

## Pruning

```bash
farrow image prune --dry-run
farrow image prune --yes
```

Prune lists exact unreferenced images and stale staging files before deletion.
An image referenced by applied deployment state is never a candidate. Images
remain cached after `destroy`, including `destroy --purge`.

The compiled Catalog can be exported byte-for-byte with
`go run ./tools/catalogexport /absolute/new/catalog.json`. A public Catalog at
the embedded version must use those exact bytes; same-version different bytes
are rejected as equivocation. Release signing and image Catalog signing remain
separate trust domains.
