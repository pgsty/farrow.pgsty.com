---
title: 日常管理
description: 唯一 deployment 的正常生命周期：检查、访问、扩容、变更、停止与销毁。
weight: 20
icon: fa-solid fa-gears
aliases: [/docs/start/lifecycle/, /docs/start/provisioning/]
---

## 检查与访问

```bash
farrow status
farrow ssh meta
farrow exec node-1 -- hostname
farrow logs meta --source serial
farrow ss                         # 必要时手工刷新 SSH 别名
```

应用状态位于 `~/.farrow`，这些命令可在任意目录执行。
Status 会显示持久化的 Guest 架构与加速器，因此 TCG 永远不是不可见回退。
`farrow up` 会在选中 VM 启动后，根据完整 applied deployment 重建默认 SSH 别名；因此
局部 `up` 不会删除未选中节点，无需再执行 `farrow ss`，即可直接运行 `ssh meta`。
`plan`、`up`、`reload`、`recreate` 依次优先使用 `-f`、当前目录发现的 Inventory，
两者都没有时才回退到已应用规格；`validate` 始终需要文件。

## 停止与启动

```bash
farrow stop
farrow start
farrow restart node-1
farrow reload -f farrow.yml       # 停止、重新读配置、收敛
```

`start` 只启动已创建 VM，不刷新 SSH 别名；`restart` 使用已应用状态；`reload` 停止后
重新读取 Inventory，并执行完整的 `up` 路径。

## 变更 deployment

```bash
farrow plan
farrow up                         # 创建/启动选中节点，并安装 SSH 别名
farrow recreate node-1 --force    # 应用某个节点的 VM 定义变化
```

| 字段 | 含义 | 操作 |
|---|---|---|
| `create` | 配置有、状态无 | `farrow up` |
| `recreate` | VM 定义改变 | `farrow recreate <node> --force` |
| `missing` | 状态有、配置无 | 恢复配置，或显式 destroy |

删除 YAML 永远不会删除 VM。未消费的 Pigsty 变更得到 `action:none`；命名与
node-admin 字段虽然不以 `vm_` 开头，仍会被消费。成功的 recreate 也会刷新完整 SSH
fragment。

## 销毁

```bash
farrow destroy node-3 --force
farrow destroy --force
farrow destroy --force --delete-persistent
farrow destroy --force --purge
```

`--delete-persistent` 与 `--purge` 只适用于整体销毁，不能和节点选择器一起使用。
`--purge` 删除持久盘、密钥和 deployment 状态，但保留镜像。节点级 destroy 会刷新
剩余节点的 SSH fragment，整体 destroy 会移除默认 Farrow SSH 集成。宿主网络单独卸载，仍有
VM 挂接时会拒绝：

```bash
farrow network uninstall --yes
```

镜像选择、镜像站与缓存清理见[镜像仓库](../images/)；彻底移除宿主状态见
[卸载与清理环境](../uninstall/)。
