---
title: 日常操作
description: 唯一 deployment 的正常生命周期：检查、访问、扩容、变更、停止与销毁。
weight: 30
icon: fa-solid fa-gears
aliases: [/docs/start/lifecycle/, /docs/start/provisioning/, /docs/start/images/]
---

## 检查与访问

```bash
farrow status
farrow ssh meta
farrow exec node-1 -- hostname
farrow logs meta --source serial
farrow ss                         # 安装 SSH 别名，之后可直接 ssh meta
```

应用状态位于 `~/.farrow`，这些命令可在任意目录执行。
Status 会显示持久化的 Guest 架构与加速器，因此 TCG 永远不是不可见回退。
`plan`、`up`、`reload`、`recreate` 依次优先使用 `-f`、当前目录发现的 Inventory，
两者都没有时才回退到已应用规格；`validate` 始终需要文件。

## 停止与启动

```bash
farrow stop                       # 别名：halt
farrow start
farrow restart node-1
farrow reload -f farrow.yml       # 停止、重新读配置、收敛
```

`restart` 使用已应用状态；`reload` 会重新读取 Inventory。

## 变更 deployment

```bash
farrow plan
farrow up                         # 创建新增节点，并启动选中的已停止节点
farrow recreate node-1 --force    # 应用某个节点的 VM 定义变化
```

| 字段 | 含义 | 操作 |
|---|---|---|
| `create` | 配置有、状态无 | `farrow up` |
| `recreate` | VM 定义改变 | `farrow recreate <node> --force` |
| `missing` | 状态有、配置无 | 恢复配置，或显式 destroy |

删除 YAML 永远不会删除 VM。未消费的 Pigsty 变更得到 `action:none`；命名与
node-admin 字段虽然不以 `vm_` 开头，仍会被消费。

## 销毁

```bash
farrow destroy node-3 --force
farrow destroy --force
farrow destroy --force --delete-persistent
farrow destroy --force --purge
```

`--delete-persistent` 与 `--purge` 只适用于整体销毁，不能和节点选择器一起使用。
`--purge` 删除持久盘、密钥和 deployment 状态，但保留镜像。宿主网络单独卸载，仍有
VM 挂接时会拒绝：

```bash
farrow network uninstall --yes
```

## 镜像

```bash
farrow image list
farrow image info u24
farrow image pull u24
farrow image prune --dry-run
```

镜像必须来自签名 Catalog，并在使用前通过 SHA-256 校验。当前镜像均为 `testing`，
只有 EOL EL7 是 `deprecated`，因此每次启动都会打印相应警告。
