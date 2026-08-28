---
title: 教程
description: 从空白 macOS 开始构建与安装 Farrow，验证单节点、增量扩到四节点、交给 Pigsty，并安全恢复出厂状态。
weight: 20
icon: fa-solid fa-play
aliases: [/docs/start/lab/, /docs/start/pigsty/, /docs/features/]
---

本教程以 Apple Silicon macOS、Farrow 源码工作区和默认 `10.10.10.0/24` 为基准。
Linux 使用相同的 Inventory 与生命周期命令，但宿主网络后端和包管理器不同。首次运行会下载
QEMU、socket_vmnet 与一个镜像；在唯一一次管理员认证前，先完整检查计划。

## 1. 确认真正的空白起点

保留源码工作区，但宿主不应再有 Farrow 的安装或运行状态：

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

后四项应找不到 QEMU/Farrow 命令、进程、状态根或 `bridge100`。如果仍有结果，先按本文末尾
的“恢复出厂状态”处理。`git status` 显示的源码修改不是运行残留，绝不能随清理一起删除。

## 2. 构建 CLI

当前源码需要 Go 1.27.x：

```bash
go version
make build
export PATH="$PWD/bin:$PATH"
farrow version
```

`make build` 会在被 Git 忽略的 `bin/` 下生成配套的 `farrow` 与
`farrow-hosts-helper`。如果正在做源码或 Release 审查，另行执行完整门禁：

```bash
make check
```

## 3. 创建并检查单节点 Inventory

```bash
mkdir -p ~/farrow-lab
cd ~/farrow-lab
farrow init meta
farrow validate --json
```

`farrow init meta` 会写出可继续编辑、且兼容 Pigsty 的 `farrow.yml`：

```yaml
all:
  vars:
    admin_ip: 10.10.10.10
  children:
    nodes:
      hosts:
        10.10.10.10: { nodename: meta }
```

先查看完整宿主事务，不做任何修改：

```bash
farrow setup -f farrow.yml --dry-run
```

在空白 Mac 上，计划应明确列出 Homebrew QEMU、带摘要锁定的 socket_vmnet 来源、固定 IP
网络、权限很窄的 hosts helper，以及每一项需要管理员权限的原因。如果 CIDR 与 VPN、其他
虚拟机产品或宿主接口重叠，应在此停止。

## 4. 执行一次性宿主 Setup

```bash
farrow setup -f farrow.yml --yes
```

macOS 询问时只认证一次。Farrow 以普通用户安装缺失依赖，仅在已展示的网络/helper 事务中
使用 root。计算能力与网络要分开验证：

```bash
farrow doctor --json
farrow network status --json
ifconfig bridge100
```

预期结果是原生 `darwin/arm64`、HVF、受支持的 QEMU 机器/设备、健康的 socket_vmnet，
以及宿主地址 `10.10.10.1/24`。Doctor 通过只证明宿主能力，不代表 VM 已经运行。

## 5. 检查并拉取镜像

```bash
farrow image list
farrow image info u24
farrow image pull u24
```

普通公共构建使用内置签名 Catalog 与其中不可变的 HTTPS Upstream。只有确实存在可达且签名
正确的镜像时，才设置 `FARROW_REPO=https://mirror.example/farrow`。拉取成功证明签名、尺寸、
SHA-256 与 qcow2 结构正确；除非另有声明，内置镜像仍是 `testing`。

## 6. 规划、启动并验证单节点

```bash
farrow plan
farrow up
farrow status
```

`up` 创建磁盘与 cloud-init seed、启动 QEMU，并等待 Guest readiness 记录。不要只停在
“SSH 能连”，还应核对 Guest 契约：

```bash
farrow exec meta -- hostname
farrow exec meta -- id dba
farrow exec meta -- ip -br addr show scope global
farrow exec meta -- lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS
farrow exec meta -- findmnt /data
ping -c 1 10.10.10.10
```

默认节点是 2 vCPU、4 GiB 内存、64 GiB 根盘，以及挂载到 `/data` 的 128 GiB XFS
数据盘。Farrow 自身 SSH 使用 `127.0.0.1:2222`；Pigsty/Ansible 使用固定 IP。

SSH 集成必须显式安装：

```bash
farrow ss
ssh meta
```

`farrow hosts install` 同样先展示计划；只有确实需要 marker-owned `/etc/hosts` 区块时才加
`--yes`。

`farrow status` 会显示实际 Guest 架构与加速器。正常路径使用原生 HVF/KVM。Apple Silicon
遇到 Stock EL8 arm64 的 64K Granule Kernel 时，会自动选择明确可见的同架构 TCG；显式
外来 `vm_arch` 也使用 TCG，且明显更慢。EL7 已 deprecated，仅限原生 Linux/amd64
BIOS/KVM。

## 7. 不重启 meta，增量扩到四节点

先记录控制节点 boot ID，再用规格相同的四节点模板替换单节点模板并检查增量：

```bash
before=$(farrow exec meta -- cat /proc/sys/kernel/random/boot_id)

farrow init full -f
farrow validate
farrow plan
```

Plan 应只包含 `create: node-1, node-2, node-3`。然后收敛：

```bash
farrow up
farrow status

after=$(farrow exec meta -- cat /proc/sys/kernel/random/boot_id)
test "$before" = "$after" && echo 'meta was not restarted'
```

检查四个固定 IP 与控制节点横向 SSH：

```bash
for ip in 10.10.10.10 10.10.10.11 10.10.10.12 10.10.10.13; do
  ping -c 1 "$ip"
done

farrow exec meta -- ssh -o BatchMode=yes dba@10.10.10.11 hostname
```

Farrow 只自动创建新增节点。VM 定义变更必须显式 `recreate`；Inventory 中删除主机只会报告，
绝不会隐式销毁。

## 8. 使用真正的 Pigsty Inventory

Farrow 每个用户只管理一套 deployment。先删除教程环境，再直接使用真实 `pigsty.yml`，无需
转换：

```bash
farrow destroy --force --purge

cd /path/to/pigsty
./configure -c meta
farrow setup -f pigsty.yml --dry-run
farrow setup -f pigsty.yml --yes
farrow up -f pigsty.yml
./install.yml
```

Farrow 只消费已记录的 VM 参数，以及命名、控制节点、登录身份字段。`pg_role`、
`pg_version`、`repo_*`、`node_packages` 等仍是 Pigsty 输入，不会产生 VM drift。

setup、VM readiness、SSH、Pigsty 安装与应用健康度是不同门禁，必须分别记录结果。

## 9. 日常操作

```bash
farrow plan
farrow status
farrow stop meta
farrow start meta
farrow restart meta
farrow logs meta --source serial
farrow destroy meta --force
```

只有确定要应用 VM 定义变更时才使用 `farrow recreate <node> --force`。普通整体 destroy
保留镜像缓存、密钥与声明为 persistent 的磁盘；`--purge` 扩大删除边界，但仍保留镜像与
宿主网络。

## 10. 将 Mac 恢复到出厂状态

本节具有破坏性。先检查每份计划，并确保没有其他 Farrow VM 需要保留。

```bash
# 如果安装过可选集成，先移除。
farrow ssh-config --remove --name farrow
farrow hosts uninstall
farrow hosts uninstall --yes       # 仅当计划显示 changed=true

# 删除 deployment、密钥、状态和所有自有磁盘；暂时保留镜像。
farrow destroy --force --purge

# 删除未引用镜像，再卸载宿主网络/socket_vmnet 服务。
farrow image prune --dry-run
farrow image prune --yes
farrow network uninstall
farrow network uninstall --yes
```

网络卸载器会刻意保留可独立使用的 hosts helper。源码安装若要完全归零，先核对以下准确路径，
再只删除该 helper 与已经为空的目录：

```bash
sudo rm -f -- /opt/farrow/libexec/farrow-hosts-helper
sudo rmdir /opt/farrow/libexec /opt/farrow
```

如果还要重新体验依赖安装，也卸载 QEMU：

```bash
brew uninstall qemu
```

最后检查选中的状态根。仅在使用默认值时，删除不再需要的 manifest/lock；不要使用宽泛递归
目标：

```bash
farrow_home_root="$(cd "$HOME" && pwd -P)/.farrow"
printf 'removing exact state root: %s\n' "$farrow_home_root"
test "$(basename "$farrow_home_root")" = '.farrow'
find "$farrow_home_root" -depth -delete
```

源码 `bin/` 只有在替换并核对其准确绝对路径后才可删除。绝不能把这些命令用于 `$HOME`、
`/` 或工作区根目录。

最后证明归零：

```bash
for process in qemu-system-aarch64 qemu-system-x86_64 socket_vmnet; do
  pgrep -x "$process" || true
done
brew list --versions qemu socket_vmnet 2>/dev/null || true
ifconfig bridge100 2>/dev/null || echo 'no bridge100'
netstat -rn -f inet | grep '10.10.10' || echo 'no 10.10.10 route'
test ! -e "$HOME/.farrow" && echo 'no Farrow state'
```

此时回到第 2 步，就是一次真正干净的 First Run 实验。
