---
title: 镜像仓库
description: 选择 Guest 镜像、使用镜像站、导入本地 qcow2，并清理缓存。
weight: 40
icon: fa-solid fa-box-archive
---

正常使用不需要先执行镜像命令：`farrow up` 默认按本机架构解析 `u24:stable`，并拉取
最终对应的不可变版本。
Farrow 一直使用已安装构建内置的 Catalog，直到你运行 `farrow update`：它会获取、校验并
激活仓库当前的 Catalog；没有任何自动刷新。恢复时可用 `image sync` 显式激活精确 URL
或文件。

## 选择镜像

先查看可用别名：

```bash
farrow image list
farrow image info u24
farrow image info u24:stable
```

内置 Family 包括 `el7`、`el8`、`el9`、`el10`、`d12`、`d13`、`u22`、`u24`、
`u26`。裸名称选择 `stable`，`name:channel` 选择频道；`name@version` 优先精确匹配，
较短的数值 Selector 则按点分量边界选择最新匹配版本：

```yaml
all:
  vars:
    vm_image: el9
    vm_version: 9.7
```

这里 `9.7` 选择最新 9.7.x Build，`9` 选择最新 9.x Release。若希望跟随仓库可移动的
Stable Channel，则改用 `vm_image: el9:stable`。

再次运行 `farrow up`。新增节点使用新镜像；已经存在的节点不会被隐式重建，定义变更需要
显式执行 `farrow recreate <node>`。

> [!WARNING]
> 除兼容用途的 EOL `el7`、EL9 9.3/9.6 与 EL10 10.0 为 `deprecated` 外，
> 其余内置版本均为 `supported`。

## 使用镜像站

Release 构建默认使用 `https://repo.pigsty.io/farrow`。单条命令可通过仅有长参数的
`--mirror` 选择中国官方仓库，也可以用 `--repo` 指定自定义根：

```bash
farrow image pull u24 --mirror
farrow up --mirror
farrow image pull u24 --repo https://mirror.example/farrow
farrow up --repo https://mirror.example/farrow
```

或为当前 Shell 设置默认仓库：

```bash
export FARROW_REPO=https://mirror.example/farrow
farrow up
```

选择优先级依次为 `--repo`、`--mirror`、`FARROW_REPO`、全球默认仓库。`--mirror`
解析为 `https://repo.pigsty.cc/farrow`，两个官方根都保持规范的签名 Catalog 信任。
`FARROW_REPO` 也可以是绝对本地目录。显式本地或 HTTPS 仓库可使用未签名 Catalog；HTTP
仓库必须提供可信密钥签名。文件大小、SHA-256 与 qcow2 结构始终校验。

只有需要下载工件时仓库不可达才有影响。Catalog Upstream URL 只用于溯源，不是隐藏回退；
选定仓库缺失工件时会直接报错。`farrow update` 获取该仓库的 Catalog，`image sync` 则获取
或读取用户明确给出的准确源。

## 构建静态仓库

仓库只是一个可以直接 rsync 或静态 HTTP 托管的目录：

```text
farrow/
├── repo.yaml
├── catalog.json
├── catalog.json.minisig       # 可选
└── images/
    └── d13-1-arm64.qcow2
```

`repo.yaml` 是唯一人工维护源，`catalog.json` 由工具生成：

```bash
farrow repo scan /srv/farrow
farrow repo build /srv/farrow
farrow repo verify /srv/farrow
```

`scan` 只读；`build` 永不修改 `repo.yaml` 或镜像字节，它运行完整的 `qemu-img check`，
并物化文件名、SHA-256、工件大小和虚拟大小。`build`、`verify` 需要本机
`qemu-img`，`scan` 不需要。在安装 QEMU 的机器上构建后，发布时先上传不可变 QCOW，
最后发布 `catalog.json`。

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
