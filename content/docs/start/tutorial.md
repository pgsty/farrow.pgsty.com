---
title: Tutorial
description: Start from a blank macOS host, build and set up Farrow, verify one node, expand to four, hand off to Pigsty, and reset safely.
weight: 20
icon: fa-solid fa-play
aliases: [/docs/start/lab/, /docs/start/pigsty/, /docs/features/]
---

This path assumes Apple Silicon macOS, a Farrow source checkout, and the default
`10.10.10.0/24` lab. Linux uses the same Inventory/lifecycle commands, but its
host-network backend and package manager differ. The first run downloads QEMU,
socket_vmnet, and one image; review the plan before the single administrator
prompt.

## 1. Confirm the blank starting point

Keep the source checkout, but start without installed/runtime state:

```bash
cd /path/to/farrow
git status --short

command -v qemu-system-aarch64 || true
command -v farrow || true
for process in qemu-system-aarch64 qemu-system-x86_64 socket_vmnet; do
  pgrep -x "$process" || true
done
test ! -e "$HOME/.farrow" && echo 'no Farrow state'
ifconfig bridge100 2>/dev/null || echo 'no bridge100'
```

The last four probes should find no QEMU/Farrow command, process, state root, or
`bridge100`. If they do, use the factory-reset section at the end before
continuing. Source changes shown by `git status` are not runtime residue and
must not be deleted.

## 2. Build the CLI

Farrow source currently requires Go 1.27.x:

```bash
go version
make build
export PATH="$PWD/bin:$PATH"
farrow version
```

`make build` creates a paired `farrow` and `farrow-hosts-helper` under ignored
`bin/`. For a source/release review, run the complete gate separately:

```bash
make check
```

## 3. Create and inspect a one-node Inventory

```bash
mkdir -p ~/farrow-lab
cd ~/farrow-lab
farrow init meta
farrow validate --json
```

`farrow init meta` writes this editable Pigsty-compatible `farrow.yml`:

```yaml
all:
  vars:
    admin_ip: 10.10.10.10
  children:
    nodes:
      hosts:
        10.10.10.10: { nodename: meta }
```

Now inspect the complete host transaction without changing anything:

```bash
farrow setup -f farrow.yml --dry-run
```

On a blank Mac the plan should name Homebrew QEMU, the digest-pinned
socket_vmnet source, the fixed-IP network, the narrow hosts helper, and every
step that needs administrator access. Stop here if the requested CIDR overlaps
a VPN, VM product, or another interface.

## 4. Apply one-time host setup

```bash
farrow setup -f farrow.yml --yes
```

Authenticate once when macOS asks. Farrow installs missing dependencies as the
normal user and uses root only for the reviewed network/helper transaction.
Verify compute and network independently:

```bash
farrow doctor --json
farrow network status --json
ifconfig bridge100
```

Expected: native `darwin/arm64`, HVF, the supported QEMU machine/devices,
healthy socket_vmnet, and host address `10.10.10.1/24`. A successful doctor is
host capability evidence; it is not yet a running VM.

## 5. Inspect and pull the image

```bash
farrow image list
farrow image info u24
farrow image pull u24
```

Ordinary public builds use the embedded signed Catalog and its immutable HTTPS
upstream. Set `FARROW_REPO=https://mirror.example/farrow` only for a reachable,
correctly signed mirror. A successful pull proves signature, size, SHA-256, and
qcow2 structure; built-in images are still `testing` unless stated otherwise.

## 6. Plan, boot, and verify one node

```bash
farrow plan
farrow up
farrow status
```

`up` creates the disks and cloud-init seed, starts QEMU, and waits for the
guest readiness record. Verify the guest contract rather than stopping at SSH:

```bash
farrow exec meta -- hostname
farrow exec meta -- id dba
farrow exec meta -- ip -br addr show scope global
farrow exec meta -- lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS
farrow exec meta -- findmnt /data
ping -c 1 10.10.10.10
```

The default node has 2 vCPU, 4 GiB memory, a 64 GiB root disk, and a 128 GiB
XFS disk at `/data`. Farrow's own SSH uses `127.0.0.1:2222`; Pigsty/Ansible
uses the fixed address.

Optional SSH integration is explicit:

```bash
farrow ss
ssh meta
```

`farrow hosts install` is also plan-first; add `--yes` only if you want the
marker-owned `/etc/hosts` block.

`farrow status` reports the effective Guest architecture and accelerator. The
normal path is native HVF/KVM. Apple Silicon automatically uses visible
same-architecture TCG for the stock EL8 arm64 64K-granule kernel; an explicit
foreign `vm_arch` also uses TCG and is much slower. EL7 is deprecated and
limited to native Linux/amd64 BIOS/KVM.

## 7. Expand to four nodes without restarting meta

Record the control node boot identity, replace the one-node template with the
matching four-node template, and inspect the delta:

```bash
before=$(farrow exec meta -- cat /proc/sys/kernel/random/boot_id)

farrow init full -f
farrow validate
farrow plan
```

The plan should contain only `create: node-1, node-2, node-3`. Apply it:

```bash
farrow up
farrow status

after=$(farrow exec meta -- cat /proc/sys/kernel/random/boot_id)
test "$before" = "$after" && echo 'meta was not restarted'
```

Check fixed-IP reachability and control-to-peer SSH:

```bash
for ip in 10.10.10.10 10.10.10.11 10.10.10.12 10.10.10.13; do
  ping -c 1 "$ip"
done

farrow exec meta -- ssh -o BatchMode=yes dba@10.10.10.11 hostname
```

Only additions are automatic. Changed VM definitions require explicit
`recreate`; removed Inventory hosts are reported but never destroyed.

## 8. Use a real Pigsty Inventory

Farrow manages one deployment per user. Destroy the tutorial deployment first,
then use the real `pigsty.yml` without translating it:

```bash
farrow destroy --force --purge

cd /path/to/pigsty
./configure -c meta
farrow setup -f pigsty.yml --dry-run
farrow setup -f pigsty.yml --yes
farrow up -f pigsty.yml
./install.yml
```

Farrow consumes only the documented VM variables plus naming, control-node,
and login-identity fields. Settings such as `pg_role`, `pg_version`, `repo_*`,
and `node_packages` remain Pigsty input and do not create VM drift.

Treat each result separately: setup, VM readiness, SSH, Pigsty install, and
application health are different gates.

## 9. Daily operations

```bash
farrow plan
farrow status
farrow stop meta
farrow start meta
farrow restart meta
farrow logs meta --source serial
farrow destroy meta --force
```

Use `farrow recreate <node> --force` only for an intended VM-definition change.
Normal whole-deployment destroy preserves image cache, keys, and declared
persistent disks; `--purge` widens that deletion boundary but still keeps
images and the host network.

## 10. Factory-reset the Mac

This section is destructive. Inspect every plan first and run it only when no
other Farrow VM should survive.

```bash
# Remove optional user/host integrations if they were installed.
farrow ssh-config --remove --name farrow
farrow hosts uninstall
farrow hosts uninstall --yes       # only when the plan says changed=true

# Delete the deployment, keys, state, and all owned disks; keep images for now.
farrow destroy --force --purge

# Remove unreferenced images, then the host network/socket_vmnet service.
farrow image prune --dry-run
farrow image prune --yes
farrow network uninstall
farrow network uninstall --yes
```

The network uninstaller intentionally preserves the independently useful
hosts helper. For a complete source-install reset, verify these exact paths and
remove only that helper and now-empty directories:

```bash
sudo rm -f -- /opt/farrow/libexec/farrow-hosts-helper
sudo rmdir /opt/farrow/libexec /opt/farrow
```

To repeat dependency installation from zero, uninstall QEMU too:

```bash
brew uninstall qemu
```

Finally inspect the chosen state root. With the default only, delete the now
unneeded manifests/locks without using a broad recursive target:

```bash
farrow_home_root="$(cd "$HOME" && pwd -P)/.farrow"
printf 'removing exact state root: %s\n' "$farrow_home_root"
test "$(basename "$farrow_home_root")" = '.farrow'
find "$farrow_home_root" -depth -delete
```

Delete a generated source `bin/` only after substituting and checking its exact
absolute checkout path. Never apply these commands to `$HOME`, `/`, or a
workspace root.

Prove the reset:

```bash
for process in qemu-system-aarch64 qemu-system-x86_64 socket_vmnet; do
  pgrep -x "$process" || true
done
brew list --versions qemu socket_vmnet 2>/dev/null || true
ifconfig bridge100 2>/dev/null || echo 'no bridge100'
netstat -rn -f inet | grep '10.10.10' || echo 'no 10.10.10 route'
test ! -e "$HOME/.farrow" && echo 'no Farrow state'
```

At that point, return to step 2 for a genuinely clean first-run experiment.
