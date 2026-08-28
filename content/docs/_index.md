---
title: Farrow documentation
linkTitle: Docs
description: Install Farrow, boot a Pigsty inventory, operate the one deployment, and look up the exact configuration and CLI contracts.
weight: 10
icon: fa-solid fa-book
cascade:
  type: docs
---

Farrow turns one Pigsty-compatible Inventory into fixed-IP QEMU virtual
machines. It manages one deployment per Unix user; state lives under
`~/.farrow`, so lifecycle and SSH commands work from any directory.

Choose the shortest path for your task:

- **[Start](start/)** — install, first lab, daily changes, and troubleshooting.
- **[Reference](reference/)** — exact Inventory fields, commands, flags,
  output, and exit codes.
- **[About](about/)** — simplified design, native validation, limits, and
  release gates.

New users should follow [the tutorial](start/tutorial/). The complete normal
path is `farrow setup`, `farrow up`, then `farrow status`.

> [!IMPORTANT]
> Farrow is pre-1.0. Current source behavior, native validation, packaging,
> release, and publication are separate gates. Read [Status](about/status/)
> before depending on a development build.
