---
title: 配置
description: Farrow 读取的 Pigsty Inventory 字段、默认值与节点级漂移行为。
weight: 10
icon: fa-solid fa-file-code
aliases: [/docs/concepts/project-model/]
---

## 发现顺序

依次查找：显式 `-f`、当前目录的 `farrow.yml`、`farrow.yaml`、`pigsty.yml`、
`pigsty.yaml`。所有文件名都使用同一种 Pigsty 兼容 YAML Inventory。

旧的顶层 `version:`/`nodes:` 格式会直接报迁移提示。

`plan`、`up`、`reload`、`recreate` 找不到文件时，如果 deployment 已存在，会回退到
已应用规格；`validate` 不会回退。配置必须是最大 4 MiB 的普通非符号链接文件。

## Farrow 读取什么

Farrow 读取主机 IP、`nodename`、`admin_ip`、`pg_cluster`、`pg_seq`、
`node_admin_username`、`node_admin_uid` 与已记录的 `vm_*` 变量。`admin_ip` 只从
`all.vars` 读取，用于选择控制节点；没有匹配时使用第一台托管主机。所有节点必须解析为
同一个登录用户名；默认用户 `dba` 的显式 `node_admin_uid` 必须为 88。

其余内容完全不读，也不会产生 drift。这里指 `pg_role`、`pg_version`、`repo_*`、
`node_packages` 等未消费字段，不能泛化为所有 `pg_*` 或 `node_*`。

命名空间内严格校验：未知 `vm_*`、错类型、Jinja 表达式、非法地址、同级分组冲突都会报错。

## VM 变量

| 变量 | 默认值 | 含义 |
|---|---|---|
| `vm_skip` | `false` | 不虚拟化这台真实/外部主机 |
| `vm_image` | `u24` | 镜像别名 |
| `vm_arch` | `native` | 部署级 Guest 架构：`native`、`amd64` 或 `arm64` |
| `vm_cpu` | `2` | vCPU 数量 |
| `vm_mem` | `4096` | MiB 整数，或 `8GiB` 等尺寸 |
| `vm_disk` | `64` | 根盘 GiB |
| `vm_disks` | `[{path: /data}]` | 额外数据盘 |
| `vm_alias` | `[]` | Guest `/etc/hosts`、SSH config 与可选宿主别名 |
| `vm_shares` | `[]` | QEMU 9p 宿主目录共享 |

空主机条目就是一台完整 VM。每套 deployment 支持 1–20 台托管主机；`vm_cpu` 范围
1–256，内存至少 512 MiB。

`vm_arch` 比普通逐主机字段更严格：出现时必须在所有托管主机上解析为同一个值，因此
应只在 `all.vars` 定义一次。修改它属于 deployment envelope 变化，必须整体重建。
Linux setup 只安装宿主原生模拟器；外来架构还需要对应 `qemu-system-*` 与固件。

## 数据盘

```yaml
vm_disks:
  - path: /data
    size: 128
    fs: xfs
    persistent: false
```

`path` 同时是磁盘身份与挂载点；`fs` 为 `xfs` 或 `ext4`。`persistent: true` 在普通
destroy 后保留；`vm_disks: []` 表示不要额外盘。

## 目录共享

```yaml
vm_shares:
  - host: /absolute/owned/source
    guest: /src
    readonly: true
```

源目录必须真实、属于调用者且互不重叠。9p 只适合可信开发文件，不能放 PostgreSQL 数据。

## 名称与地址

节点名依次取 `nodename`、`<pg_cluster>-<pg_seq>`、`node-<IP末段>`，且必须唯一。

所有托管主机必须位于同一个 RFC1918 `/24`：`.1` 属于宿主，`.2`–`.8` 保留，节点使用
`.9`–`.254`。

## 漂移

Farrow 对每个解析后节点计算哈希。新增主机由 `up` 创建；选中的已停止节点会启动，
运行中同伴不受影响。VM 定义变化需要节点级 recreate；删除主机条目只报告、绝不销毁。
deployment 架构、用户或子网变化需要整体重建。修改用于派生节点名的字段会表现为旧节点
`missing` 加新节点，建议使用稳定、显式的 `nodename`。
