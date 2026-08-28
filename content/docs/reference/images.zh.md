---
title: 镜像
description: 签名 Catalog、内置别名、仓库选择、本地缓存校验、导入与清理。
weight: 30
icon: fa-solid fa-hard-drive
---

Farrow 使用一份签名静态 Catalog 与不可变 qcow2 工件。更新 Catalog 不需要发布新的
Farrow 二进制，但二进制决定信任哪些签名公钥与镜像安全规则。

> [!WARNING]
> EL7 标记为 `deprecated`；其余内置镜像均为 `testing`，不是 `supported`。`up`
> 打印的警告是刻意保留的；拉取成功只证明完整性，不代表生产支持承诺。

## 别名与拉取顺序

内置 Catalog 包含 9 个 Family、17 个工件：`el7` 只有 amd64；`el8`、`el9`、
`el10`、`d12`、`d13`、`u22`、`u24`、`u26` 均有 amd64 与 arm64。默认使用 `u24`。

| 别名 | 发行版 | 架构 | 启动 | 状态 |
|---|---|---|---|---|
| `el7` | CentOS Linux 7.9 / 2211 | amd64 | BIOS | deprecated |
| `el8` | Rocky Linux 8.10 | amd64、arm64 | UEFI | testing |
| `el9`、`el10` | Rocky Linux | amd64、arm64 | UEFI | testing |
| `d12`、`d13` | Debian | amd64、arm64 | UEFI | testing |
| `u22`、`u24`、`u26` | Ubuntu | amd64、arm64 | UEFI | testing |

```bash
farrow image list
farrow image info u24
farrow image pull u24
```

拉取时 Farrow 会：

1. 按 `--repo`、`FARROW_REPO`、编译期默认仓库的顺序刷新 `catalog.json` 与相邻
   `.minisig`；
2. 使用内置 active 或 standby 公钥验证签名；
3. 选择 Catalog 默认 Release；独立 `image pull` 使用本机架构，生命周期解析遵循
   `vm_arch`；
4. 只有尺寸、SHA-256、qcow2 结构全部匹配时才复用本地文件；
5. 否则下载仓库工件，仓库缺失时回退到 Catalog 中不可变的 HTTPS Upstream URL。

当前编译默认值仍是开发仓库 `https://m0/farrow`，尚未声称存在公开镜像服务。显式仓库
失败会报错；无法访问编译默认仓库时则回退到内置 Catalog。

## 运行时策略

匹配架构正常使用原生 HVF/KVM，只有一个 Catalog 已知例外：Stock EL8 arm64 的 64K
Granule Kernel 无法通过 Apple HVF 运行，因此 Apple Silicon 会自动选择可见的同架构
TCG。显式外来 `vm_arch` 也会使用 TCG；arm64 宿主上的 amd64 Guest 使用单翻译线程，
以保留 x86 内存序。TCG 结果不能作为性能证据。

EL7 刻意仅支持 Linux/amd64 原生运行。Linux setup 只安装宿主原生 QEMU；外来架构必须
先安装对应 System Emulator 与 UEFI 固件，`plan`、`up`、`recreate` 才会继续。

```bash
farrow image pull --repo https://mirror.example/farrow u24
FARROW_REPO=/absolute/local/repository farrow up
```

仓库 URL 可以是 HTTP 或 HTTPS，因为 Catalog 签名与镜像摘要才是完整性权威；Catalog
中的不可变 Upstream 工件 URL 必须使用 HTTPS。

## 信任与校验

当前普通构建已经内置两把生产校验公钥；私有签名密钥不在源码仓库中。Catalog 激活会拒绝
未知密钥、畸形内容、同版本异内容，以及低于已记录 High-water Mark 的版本；只有操作者
显式允许时才可降级。

镜像必须是尺寸与 SHA-256 匹配的纯 qcow2，不得有 backing file、外部数据文件、加密或
未知不兼容 Feature。通过校验的 Base Image 变成只读；节点根盘使用 Overlay，永不修改
Base。

```bash
farrow image sync https://repo.example/farrow/catalog.json
farrow image sync --allow-downgrade /absolute/repo/catalog.json
farrow image reset-manifest
```

`reset-manifest` 恢复二进制内置的 Bootstrap Catalog，但不会清除防回滚 High-water Mark。

## 本地布局与导入

镜像位于 `FARROW_HOME/images`（默认 `~/.farrow/images`）：各 Family 目录保存下载工件，
`manifests/` 保存 Catalog 状态，`local/` 与 `local-images.json` 保存导入镜像。已经没有旧的
`~/.farrow/cache` 或按摘要组织的 `sha256/` 层级。

```bash
farrow image import --sha256 <digest> /path/to/base.qcow2
farrow image import --name local-mybase --boot uefi \
  --source-user ubuntu --sha256 <digest> /path/to/base.qcow2
```

命名的本地别名必须以 `local-` 开头，因此未来的签名 Catalog 无法覆盖它。使用 `--name`
时必须同时给出 `--boot` 与 `--source-user`；Farrow 不猜 Guest Bootstrap 契约。

## 清理

```bash
farrow image prune --dry-run
farrow image prune --yes
```

Prune 会先列出准确的未引用镜像与遗留 Staging 文件。已应用 deployment 引用的镜像永远
不是候选；`destroy`（包括 `destroy --purge`）后镜像仍保留缓存。

使用 `go run ./tools/catalogexport /absolute/new/catalog.json` 可逐字节导出编译期 Catalog。
公开 Catalog 若使用相同版本，就必须使用完全相同的字节；同版本不同内容会按 equivocation
拒绝。Release 签名与镜像 Catalog 签名仍属于不同信任域。
