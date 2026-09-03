---
title: Build from Source
description: Build Farrow for development and review, then run the complete source gate.
weight: 50
icon: fa-solid fa-code-branch
---

Ordinary users should prefer an archive or native package from a formal
Release. This page is for development and source review. Farrow remains
pre-1.0; see [Status](../../about/status/) for public-package availability.

## Build

The current tree pins Go 1.27.1. You also need Git, Make, and standard build
tools. From a source checkout:

```bash
cd /path/to/farrow
make build
export PATH="$PWD/bin:$PATH"
farrow version
```

`make build` writes the matching `farrow` and `farrow-hosts-helper` binaries
under the Git-ignored `bin/` directory. Do not mix the two binaries across
commits or releases.

Then use a separate lab directory:

```bash
mkdir -p ~/farrow-lab && cd ~/farrow-lab
farrow setup
farrow up
```

## Complete checks

Before submitting a source change, run:

```bash
make check
```

This gate includes unit and race tests, Vet, Staticcheck, vulnerability checks,
cross-builds, and static contracts around releases, catalogs, and the image
pipeline. A passing source gate is not package publication or a native VM
lifecycle replay.

See [Engineering](../../about/engineering/) for release tooling, dependency
licenses, and validation boundaries.
