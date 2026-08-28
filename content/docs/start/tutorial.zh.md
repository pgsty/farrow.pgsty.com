---
title: 教程
description: 启动单节点、验证它、无重启扩到四节点，并把同一份 Inventory 交给 Pigsty。
weight: 20
icon: fa-solid fa-play
aliases: [/docs/start/lab/, /docs/start/pigsty/, /docs/features/]
---

## 1. 创建实验环境

```bash
mkdir -p ~/lab
cd ~/lab
export FARROW_REPO=https://m0/farrow   # 离开开发网络后替换为可达镜像
farrow setup --dry-run
farrow setup --yes
farrow up
```

目录中没有配置时，setup 会写入单节点 `farrow.yml`：

```yaml
all:
  vars:
    admin_ip: 10.10.10.10
  children:
    nodes:
      hosts:
        10.10.10.10: { nodename: meta }
```

`up` 下载并校验镜像，创建磁盘与 cloud-init seed，启动 QEMU，并等待 Guest 的
readiness 记录。

## 2. 验证节点

```bash
farrow status
farrow ssh meta
farrow exec meta -- hostname
ping 10.10.10.10
```

默认节点有 2 vCPU、4 GiB 内存、64 GiB 根盘，以及挂载到 `/data` 的 128 GiB
XFS 数据盘。

`farrow status` 还会显示实际 Guest 架构与加速器。默认 EL9/EL10 使用原生 HVF/KVM。
Apple Silicon 上设置 `vm_image: el8` 时，Stock 64K Granule Kernel 会自动使用同架构
TCG。显式设置部署级 `vm_arch: amd64` 可运行 amd64 Guest，但会使用单线程 TCG，速度
明显更慢。EL7 仅作为 deprecated 的 Linux/amd64 原生 BIOS/KVM Guest 提供。

## 3. 扩到四节点

在同一个 `hosts:` 映射下增加三行：

```yaml
        10.10.10.11: { nodename: node-1 }
        10.10.10.12: { nodename: node-2 }
        10.10.10.13: { nodename: node-3 }
```

然后收敛：

```bash
farrow plan       # create: node-1, node-2, node-3
farrow up
farrow status
```

只会创建新增节点，`meta` 的进程与 uptime 不变。控制节点第一次启动时就获得了
deployment 密钥，因此可以立即 SSH 到新节点：

```bash
farrow exec meta -- ssh dba@10.10.10.11 hostname
```

## 4. 与 Pigsty 共用同一份文件

如果 Pigsty 已生成 `pigsty.yml`，无需转换：

```bash
./configure -c meta
farrow setup
farrow up
./install.yml
```

Farrow 读取已记录的 VM 字段，以及用于命名、控制节点和登录身份的原生字段。
`pg_role`、`pg_version`、`repo_*`、`node_packages` 等未消费参数不会产生 VM drift；
`pg_cluster`/`pg_seq` 与 node-admin 字段会被消费。

## 5. 停止或移除

```bash
farrow stop
farrow start
farrow destroy --force
```

普通 destroy 保留已校验镜像缓存、deployment 密钥与持久盘。只有明确需要时才使用
`--delete-persistent` 或 `--purge`。
