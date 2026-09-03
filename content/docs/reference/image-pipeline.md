---
title: Image Pipeline
description: Validate or offline-normalize an explicit qcow2 candidate without downloading, uploading, or signing it.
weight: 40
icon: fa-solid fa-shield-halved
---

The low-level `packaging/image-pipeline/build.sh` accepts one already-downloaded
immutable qcow2 and an independently obtained SHA-256. It never downloads,
uploads, touches Farrow runtime/network state, reads signing keys, or marks an
image `supported`.

## Modes

- `validate`: copy/re-hash, force qcow2 inspection, validate the single backing
  chain, run `qemu-img check`, and emit an explicitly unpublishable evidence
  bundle. Guest credentials are not changed.
- `offline`: additionally use libguestfs `virt-customize --no-network` and
  `virt-cat` on the staged copy. It rejects unrelated UID/GID 88 occupants,
  normalizes the locked `dba`/`admin` identity, disables password/root SSH,
  removes keys/history/host identity/cloud-init cache, restores targeted SELinux
  labels, and reads back a deterministic marker.

## Official candidate matrix

`build-official.py` wraps the same offline boundary for a fixed eight-target
matrix: Debian 12/13 and Rocky Linux 8/9, each on amd64 and arm64. Every
upstream qcow2, RPM/DEB input, release name, digest, and source epoch is pinned
in `official-v1.json`.

```bash
./packaging/image-pipeline/build-official.py --list

./packaging/image-pipeline/build-official.py \
  --source-cache /absolute/source-cache \
  --package-cache /absolute/package-cache \
  --output /absolute/existing-output-root \
  --target d13/arm64 --fetch
```

Without `--fetch`, every locked input must already exist in the two canonical
cache directories. With it, the wrapper downloads only the pinned HTTPS URLs
and rejects any digest mismatch before invoking offline normalization. Debian
12/13 install the locked XFS userspace closure; Rocky Linux 8 installs the
locked Python/SELinux closure, and Rocky Linux 9 needs no extra package input.

Each result remains an unsigned `testing` candidate. Supplying all eight bundle
roots to `--assemble-from` creates a new candidate static repository and runs
`farrow repo build` plus `verify`; this still does not perform native smoke,
signing, upload, or Catalog publication.

```bash
SOURCE_DATE_EPOCH=1787486400

./packaging/image-pipeline/build.sh \
  --mode validate \
  --source /absolute/source.qcow2 \
  --expected-sha256 <digest> \
  --output /absolute/new/evidence-directory \
  --name u24 --release 20260801.0.0 --arch amd64 \
  --source-user ubuntu --boot uefi \
  --source-uri https://immutable.example/source.qcow2 \
  --artifact-url 'https://images.example/u24/{sha256}.qcow2' \
  --license NOASSERTION \
  --source-date-epoch "$SOURCE_DATE_EPOCH" \
  --manifest-version 2026082903
```

Source/output paths must be absolute; source is canonical, regular,
non-symlinked, stable while copied, and at most 16 GiB. Output must not exist.
The builder uses an exclusive adjacent lock, mode-0700 staging, and one final
rename. Failure removes only its guarded staging directory.

Every successful bundle contains the read-only qcow2, recipe, SLSA provenance,
SPDX boundary SBOM, `manifest-candidate.json` (`testing`), validation evidence,
and checksums. Signing is deliberately outside this pipeline. Validate mode is
byte-reproducible for fixed inputs/tools; offline mutation must be built twice
and compared before release evidence is accepted.
