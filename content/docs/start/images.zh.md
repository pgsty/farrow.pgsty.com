---
title: 镜像仓库
description: 选择 Guest 镜像、使用镜像站、导入本地 qcow2，并清理缓存。
weight: 40
icon: fa-solid fa-box-archive
---

正常使用不需要先执行镜像命令：`farrow up` 会自动拉取并校验默认的 `u24` 镜像。

## 选择镜像

先查看可用别名：

```bash
farrow image list
farrow image info u24
```

内置 Family 包括 `el7`、`el8`、`el9`、`el10`、`d12`、`d13`、`u22`、`u24`、
`u26`。在 Inventory 中通过 `vm_image` 选择：

```yaml
all:
  vars:
    vm_image: el9
```

再次运行 `farrow up`。新增节点使用新镜像；已经存在的节点不会被隐式重建，定义变更需要
显式执行 `farrow recreate <node>`。

> [!WARNING]
> 当前内置镜像仍为 `testing`，EOL 的 `el7` 为 `deprecated`。下载与摘要校验成功只证明
> 工件完整性，不代表生产支持。

## 使用镜像站

为单条命令指定仓库：

```bash
farrow image pull --repo https://mirror.example/farrow u24
farrow up --repo https://mirror.example/farrow
```

或为当前 Shell 设置默认仓库：

```bash
export FARROW_REPO=https://mirror.example/farrow
farrow up
```

`FARROW_REPO` 也可以是绝对本地目录。Farrow 仍会验证签名 Catalog、文件大小、SHA-256 与
qcow2 结构；显式仓库不可达时会直接报错。

## 导入与清理

导入自己获得的 qcow2 时必须提供独立校验值：

```bash
farrow image import --name local-mybase --boot uefi \
  --source-user ubuntu --sha256 <digest> /path/to/base.qcow2
```

自定义别名必须以 `local-` 开头。清理未被当前 deployment 引用的缓存前，先看计划：

```bash
farrow image prune --dry-run
farrow image prune --yes
```

签名、回滚保护、缓存布局、架构与 TCG 规则见[镜像参考](../../reference/images/)；构建与发布
镜像 Candidate 见[镜像流水线](../../reference/image-pipeline/)。
