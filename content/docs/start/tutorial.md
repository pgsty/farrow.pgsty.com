---
title: Tutorial
description: Boot a one-node lab, verify it, expand it to four nodes without restarting the first, and hand the same inventory to Pigsty.
weight: 20
icon: fa-solid fa-play
aliases: [/docs/start/lab/, /docs/start/pigsty/, /docs/features/]
---

## 1. Create the lab

```bash
mkdir -p ~/lab
cd ~/lab
export FARROW_REPO=https://m0/farrow   # replace outside the development network
farrow setup --dry-run
farrow setup --yes
farrow up
```

With no existing file, setup writes `farrow.yml` for one node:

```yaml
all:
  vars:
    admin_ip: 10.10.10.10
  children:
    nodes:
      hosts:
        10.10.10.10: { nodename: meta }
```

`up` downloads and verifies the image, creates the disks and cloud-init seed,
starts QEMU, and waits for the guest readiness record.

## 2. Verify the node

```bash
farrow status
farrow ssh meta
farrow exec meta -- hostname
ping 10.10.10.10
```

The default node has 2 vCPU, 4 GiB memory, a 64 GiB root disk, and a 128 GiB
XFS disk at `/data`.

`farrow status` also reports the effective Guest architecture and accelerator.
The default EL9/EL10 path is native HVF/KVM. On Apple Silicon, `vm_image: el8`
automatically selects same-architecture TCG for the stock 64K-granule kernel.
To run an amd64 Guest explicitly, set deployment-wide `vm_arch: amd64`; this
uses single-threaded TCG and is intentionally much slower. EL7 is available
only as a deprecated, native Linux/amd64 BIOS/KVM Guest.

## 3. Expand to four nodes

Add three host entries below the same `hosts:` mapping:

```yaml
        10.10.10.11: { nodename: node-1 }
        10.10.10.12: { nodename: node-2 }
        10.10.10.13: { nodename: node-3 }
```

Then converge:

```bash
farrow plan       # create: node-1, node-2, node-3
farrow up
farrow status
```

Only the new nodes are created. `meta` keeps its process and uptime. The
control node received the deployment key on its first boot, so it can SSH to
the new nodes immediately:

```bash
farrow exec meta -- ssh dba@10.10.10.11 hostname
```

## 4. Use the same file with Pigsty

If Pigsty already generated `pigsty.yml`, skip the example file and run Farrow
in that checkout:

```bash
./configure -c meta
farrow setup
farrow up
./install.yml
```

Farrow reads the documented VM fields plus native fields for naming,
control-node selection, and the login identity. Unconsumed settings such as
`pg_role`, `pg_version`, `repo_*`, and `node_packages` do not produce VM
drift; `pg_cluster`/`pg_seq` and the node-admin fields are consumed.

## 5. Stop or remove it

```bash
farrow stop
farrow start
farrow destroy --force
```

Normal destroy preserves the verified image cache, deployment keys, and
persistent disks. Use `--delete-persistent` or `--purge` only when intended.
