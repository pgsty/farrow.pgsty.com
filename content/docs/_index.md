---
title: Farrow documentation
linkTitle: Docs
description: Start Farrow with two commands, then look up operations, configuration, and CLI contracts as needed.
weight: 10
icon: fa-solid fa-book
cascade:
  type: docs
---

Farrow turns one Pigsty-compatible Inventory into fixed-IP QEMU virtual
machines. It manages one deployment per Unix user; state lives under
`~/.farrow`, so lifecycle and SSH commands work from any directory.

Choose the shortest path for your task:

- **[Start](start/)** — boot the first lab with two commands, then operate and troubleshoot it.
- **[Reference](reference/)** — exact Inventory fields, commands, flags,
  output, and exit codes.
- **[About](about/)** — design, native validation, limits, and
  release gates.

New users should follow the [Quick Start](start/tutorial/). The complete normal
path is `farrow setup` and `farrow up`.

> [!IMPORTANT]
> Farrow is pre-1.0. Current source behavior, native validation, packaging,
> release, and publication are separate gates. Read [Status](about/status/)
> before depending on a development build.
