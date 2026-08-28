---
title: Start
linkTitle: Start
description: Start Farrow with two commands; read operations, troubleshooting, images, builds, and cleanup only when needed.
weight: 10
icon: fa-solid fa-rocket
cascade:
  type: docs
---

With Farrow installed, new users only need the [Quick Start](tutorial/):

```bash
farrow setup
farrow up
```

Package availability is recorded on [Status](../about/status/); developers and
source reviewers can use [Build from Source](source-build/).

Everything else is separated by task:

1. [Daily Operations](operations/) — status, access, start/stop, changes,
   scale-in, and destroy.
2. [Troubleshooting](troubleshooting/) — read-only diagnosis and common fixes.
3. [Image Repositories](images/) — choose images, use mirrors, import, and
   prune the cache.
4. [Build from Source](source-build/) — developer builds, checks, and local
   PATH setup.
5. [Uninstall and Clean Up](uninstall/) — remove the deployment, images,
   networking, and state.
