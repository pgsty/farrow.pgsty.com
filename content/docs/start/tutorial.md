---
title: Quick Start
linkTitle: Quick Start
description: Boot a one-node Farrow lab with setup and up, then scale it additively from the same inventory.
weight: 10
icon: fa-solid fa-play
aliases: [/docs/start/installation/, /docs/start/upgrade/, /docs/start/lab/, /docs/start/pigsty/, /docs/features/]
---

This guide assumes a matching Farrow installation is already on `PATH`. Check
[Status](../../about/status/) for formal-package availability, or use
[Build from Source](../source-build/) for development and source review.

In an empty directory, the normal runtime path is only two commands:

```bash
mkdir -p ~/farrow-lab && cd ~/farrow-lab
farrow setup
farrow up
```

`setup` prepares the host and creates a default one-node `farrow.yml` when no
inventory exists. `up` pulls the image, starts the VM, and synchronizes the
default SSH aliases. The rest of this page expands those two commands.

## 1. Prepare the host

Run:

```bash
farrow setup
```

Farrow shows one plan for dependencies, fixed-IP networking, and privileged
changes before asking for confirmation. Existing dependencies are skipped, so
installing QEMU first makes the initial setup much faster.

Downloads honor configured `HTTP_PROXY`, `HTTPS_PROXY`, `ALL_PROXY`, and
`NO_PROXY` variables, including lowercase forms. Setup may preserve the names
of configured proxy variables across sudo, but never prints their potentially
credential-bearing values.

```bash {tab="Automatic (recommended)" group="host-deps" value="auto"}
farrow setup
```

```bash {tab="macOS: preinstall QEMU" value="macos"}
brew install qemu
farrow setup
```

```bash {tab="Debian / Ubuntu: preinstall QEMU" value="debian"}
# amd64
sudo apt-get update
sudo apt-get install -y qemu-system-x86 qemu-utils ovmf openssh-client iproute2
farrow setup
```

```bash {tab="RHEL / Fedora: preinstall QEMU" value="rpm"}
# amd64
sudo dnf install -y qemu-kvm qemu-img edk2-ovmf openssh-clients iproute
farrow setup
```

On ARM64 Linux, use the distribution's corresponding arm64 QEMU and firmware
packages. See [Build from Source](../source-build/) and [Status](../../about/status/)
for exact requirements and the current host-support boundary.

> [!NOTE]
> In an empty directory, `farrow setup` also creates the default `meta`
> inventory. If `farrow.yml` or `pigsty.yml` already exists, setup reuses it.

## 2. Configure and start

### Generate an inventory explicitly

To inspect or choose an inventory before setup, run this in an empty directory:

```bash
farrow init
```

Example output:

```text
template:  meta
wrote:     /Users/you/farrow-lab/farrow.yml
next:      farrow setup && farrow up
```

With no argument, `init` uses the one-node `meta` spec. There are four built-in
specs:

| Spec | Nodes | Addresses |
|---|---:|---|
| `meta` | 1 | `10.10.10.10` |
| `dual` | 2 | `10.10.10.10`–`10.10.10.11` |
| `trio` | 3 | `10.10.10.10`–`10.10.10.12` |
| `full` | 4 | `10.10.10.10`–`10.10.10.13` |

For example, `farrow init full` writes a four-node inventory, while
`farrow init full -c 10.20.30.0/24` uses another subnet. Existing files are
preserved unless `--force` is explicit. See [Configuration](../../reference/configuration/)
for every field.

### Start the first node

The essential default inventory is one node:

```yaml
all:
  vars:
    admin_ip: 10.10.10.10
  children:
    nodes:
      hosts:
        10.10.10.10: { nodename: meta }
```

Start it:

```bash
farrow up
```

The first run resolves, downloads, and verifies the default `d13:stable` image, then waits for
guest readiness. This is example output from macOS arm64; the spec hash, PID,
and accelerator vary by inventory and host:

```text
spec hash:   edda3b54e04e00c76de036ea9ea5b6ea90c53abd245dee8aa7a0335999ba0a65
meta             running  runtime=running  arch=arm64  accel=hvf  address=10.10.10.10  ssh=127.0.0.1:2222 pid=16254
created and started 1 node(s)
ssh config:  /Users/you/.ssh/farrow_config (action=install changed=true)
```

Open the VM:

```bash
farrow ssh meta
```

> [!TIP]
> `up` manages the image automatically. Read [Image Repositories](../images/)
> only when you need another distribution, a mirror, or cache management.

## 3. Scale and operate

Add more nodes to `farrow.yml`:

```yaml
all:
  vars:
    admin_ip: 10.10.10.10
  children:
    nodes:
      hosts:
        10.10.10.10: { nodename: meta }
        10.10.10.11: { nodename: node-1 }
        10.10.10.12: { nodename: node-2 }
        10.10.10.13: { nodename: node-3 }
```

Run the same command again:

```bash
farrow up
```

Farrow creates only the three additions and leaves the running `meta` node
untouched:

```text
spec hash:   d61dfc499f32206e56936d83b38246f8c9ca4740b4579ac9aed2e29907441b38
meta             running  runtime=running  arch=arm64  accel=hvf  address=10.10.10.10  ssh=127.0.0.1:2222 pid=16254
node-1           running  runtime=running  arch=arm64  accel=hvf  address=10.10.10.11  ssh=127.0.0.1:2223 pid=16774
node-2           running  runtime=running  arch=arm64  accel=hvf  address=10.10.10.12  ssh=127.0.0.1:2224 pid=16772
node-3           running  runtime=running  arch=arm64  accel=hvf  address=10.10.10.13  ssh=127.0.0.1:2225 pid=16773
created and started 3 node(s)
ssh config:  /Users/you/.ssh/farrow_config (action=install changed=true)
```

Use `st`, the alias of `status`, at any time:

```bash
farrow st
```

```text
spec hash:   d61dfc499f32206e56936d83b38246f8c9ca4740b4579ac9aed2e29907441b38
meta             running  runtime=running  arch=arm64  accel=hvf  address=10.10.10.10  ssh=127.0.0.1:2222 pid=16254
node-1           running  runtime=running  arch=arm64  accel=hvf  address=10.10.10.11  ssh=127.0.0.1:2223 pid=16774
node-2           running  runtime=running  arch=arm64  accel=hvf  address=10.10.10.12  ssh=127.0.0.1:2224 pid=16772
node-3           running  runtime=running  arch=arm64  accel=hvf  address=10.10.10.13  ssh=127.0.0.1:2225 pid=16773
deployment status
```

Destroy the whole deployment when finished:

```bash
farrow destroy
```

The terminal asks you to type `destroy`. Images, keys, and declared persistent
data disks remain. See [Uninstall and Clean Up](../uninstall/) for a complete
reset, or [Daily Operations](../operations/) for stop, restart, logs, explicit
changes, and scale-in.
