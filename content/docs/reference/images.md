---
title: Images
description: Signed catalogs, built-in aliases, repository selection, local cache verification, imports, and pruning.
weight: 30
icon: fa-solid fa-hard-drive
---

Farrow uses a materialized static-file Catalog plus immutable qcow2 artifacts.
Official and HTTP Catalogs are signed; explicitly selected local and HTTPS
repositories may be unsigned. A Catalog update does not require a new Farrow
binary, but the binary decides which signing keys and image safety rules are
trusted.

> [!WARNING]
> EL7 is `deprecated`; every other built-in image is currently `testing`, not
> `supported`. The warning printed by `up` is intentional: a successful pull
> is an integrity result, not a production-support promise.

## Aliases and pull order

The embedded Catalog contains 9 families and 17 artifacts: `el7` is
amd64-only; `el8`, `el9`, `el10`, `d12`, `d13`, `u22`, `u24`, and `u26`
have amd64 and arm64 artifacts. `d13:stable` on the native architecture is the
default request.

| Alias | Distribution | Architectures | Boot | Status |
|---|---|---|---|---|
| `el7` | CentOS Linux 7.9 / 2211 | amd64 | BIOS | deprecated |
| `el8` | Rocky Linux 8.10 | amd64, arm64 | UEFI | testing |
| `el9`, `el10` | Rocky Linux | amd64, arm64 | UEFI | testing |
| `d12`, `d13` | Debian | amd64, arm64 | UEFI | testing |
| `u22`, `u24`, `u26` | Ubuntu | amd64, arm64 | UEFI | testing |

```bash
farrow image list
farrow image info d13
farrow image info d13:stable
farrow image pull d13@20260810.2566.0
farrow image pull d13 --arch arm64
```

For a pull, Farrow:

1. refreshes `catalog.json` and its adjacent `.minisig` from `--repo`, then
   `FARROW_REPO`, then an optional compiled public default;
2. verifies a required signature, or records the explicitly selected local or
   HTTPS repository as unsigned;
3. resolves `image[:channel]` or `image@version`, defaulting to
   `d13:stable`; standalone `image pull` uses the native architecture, while
   lifecycle resolution honors `vm_arch`;
4. reuses a local file only after size, SHA-256, and qcow2 checks pass;
5. otherwise downloads the repository artifact, with the Catalog's immutable
   HTTPS upstream URL as fallback.

Ordinary public builds have no compiled repository default: they use the
embedded signed Catalog and its immutable upstream URLs. An explicit repository
failure is fatal. A future release may compile in a public mirror only after it
is live; an unavailable non-explicit compiled default falls back to the embedded
Catalog.

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
farrow image pull d13 --repo https://mirror.example/farrow
FARROW_REPO=/absolute/local/repository farrow up
```

Unsigned repositories must be local paths or HTTPS. HTTP repositories require a
Catalog signed by a trusted key. Immutable upstream artifact URLs must be HTTPS.

## Trust and verification

Current ordinary builds embed both production public verification keys. The
private signing keys are external to the source repository. Catalog activation
rejects unknown keys, malformed content, equivocation, and revisions below the
repository-scoped high-water mark unless the operator explicitly allows a
downgrade.

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

For repository-scoped recovery, pass the same root explicitly:

```bash
farrow image sync --repo /srv/farrow --allow-downgrade /srv/farrow/catalog.json
farrow image reset-manifest --repo /srv/farrow
```

`--repo` selects the independent high-water slot. If omitted, `FARROW_REPO`,
then the compiled default, is used.

## Static repository format

The published root is deliberately small:

```text
farrow/
├── repo.yaml
├── catalog.json
├── catalog.json.minisig       # optional except for HTTP
└── images/
    └── <image>-<version>-<arch>.qcow2
```

`repo.yaml` stores author intent: defaults, aliases, channels, exact versions,
architectures, boot mode, status, and optional upstream URLs. It contains no
generated checksum or size fields. `catalog.json` uses the same logical tree but
materializes each variant's file, SHA-256, artifact size, and virtual size.

```yaml
schema: 1
revision: 1
defaults: { image: d13, channel: stable, arch: native, boot: uefi }
images:
  d13:
    aliases: [debian13, trixie, debian]
    channels: { stable: "1" }
    versions:
      "1":
        status: unknown
        variants:
          amd64: {}
          arm64: {}
```

With no explicit `file`, the two expected artifacts are
`images/d13-1-amd64.qcow2` and `images/d13-1-arm64.qcow2`. A variant may use a
safe basename override for an existing custom file.

`stable` and `testing` are movable channels and never appear in artifact file
names. The immutable artifact identity is `(image, exact version, arch)`:

```text
d13:stable + native
  -> d13@20260810.2566.0 + arm64
  -> images/d13-20260810.2566.0-arm64.qcow2
```

`farrow repo scan` is read-only. `build` performs strict YAML validation,
full qcow2 inspection/checking, and atomic Catalog replacement without changing
`repo.yaml` or QCOW bytes. `verify` requires the generated Catalog bytes to
match a fresh materialization exactly. `build` and `verify` require local
`qemu-img`; `scan` does not. Build on a QEMU host, then rsync artifacts first
and the Catalog last.

## Local layout and imports

Images live under `FARROW_HOME/images` (default `~/.farrow/images`): family
directories contain downloaded artifacts, `manifests/` stores Catalog state
with an independent high-water entry per repository,
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

The compiled schema-3 Catalog can be exported byte-for-byte with
`go run ./tools/catalogexport /absolute/new/catalog.json`. A public Catalog at
the embedded version must use those exact bytes; same-version different bytes
are rejected as equivocation. Release signing and image Catalog signing remain
separate trust domains.
