---
title: Image Pipeline
description: Validate or offline-normalize an explicit qcow2 candidate without downloading, uploading, or signing it.
weight: 40
icon: fa-solid fa-shield-halved
---

`packaging/image-pipeline/` accepts one already-downloaded immutable qcow2 and
an independently obtained SHA-256. It never downloads, uploads, touches Farrow
runtime/network state, reads signing keys, or marks an image `supported`.

## Modes

- `validate`: copy/re-hash, force qcow2 inspection, validate the single backing
  chain, run `qemu-img check`, and emit an explicitly unpublishable evidence
  bundle. Guest credentials are not changed.
- `offline`: additionally use libguestfs `virt-customize --no-network` and
  `virt-cat` on the staged copy. It rejects unrelated UID/GID 88 occupants,
  normalizes the locked `dba`/`admin` identity, disables password/root SSH,
  removes keys/history/host identity/cloud-init cache, restores targeted SELinux
  labels, and reads back a deterministic marker.

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
  --manifest-version 2026082801
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
