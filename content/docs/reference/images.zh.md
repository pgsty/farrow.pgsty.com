---
title: 镜像
description: 签名 Catalog、内置别名、仓库选择、本地缓存校验、导入与清理。
weight: 30
icon: fa-solid fa-hard-drive
---

Farrow 使用物化的静态 Catalog 与不可变 qcow2 工件。官方与 HTTP Catalog 必须签名；
用户显式选择的本地或 HTTPS 仓库可以不签名。更新 Catalog 不需要发布新的 Farrow
二进制，但二进制决定信任哪些签名公钥与镜像安全规则。

> [!WARNING]
> EL7、EL9 9.3/9.6 与 EL10 10.0 是 `deprecated` 兼容镜像；其余内置版本均为
> `supported`。

## 别名与拉取顺序

内置 Catalog 包含 9 个 Family、27 个工件：`el7` 只有 amd64，其余 Family 均有
amd64 与 arm64。EL9 包含 9.3、9.6、9.7、9.8；EL10 包含 10.0、10.1、10.2。
默认请求为本机架构的 `u24:stable`（Ubuntu 24.04）。

| 别名 | 发行版 | 架构 | 启动 | 状态 |
|---|---|---|---|---|
| `el7` | CentOS Linux 7.9 / 2211 | amd64 | BIOS | deprecated |
| `el8` | Rocky Linux 8.10 | amd64、arm64 | UEFI | supported |
| `el9` | Rocky Linux 9.7 / 9.8 | amd64、arm64 | UEFI | supported |
| `el9` | Rocky Linux 9.3 / 9.6 | amd64、arm64 | UEFI | deprecated |
| `el10` | Rocky Linux 10.1 / 10.2 | amd64、arm64 | UEFI | supported |
| `el10` | Rocky Linux 10.0 | amd64、arm64 | UEFI | deprecated |
| `d12`、`d13` | Debian | amd64、arm64 | UEFI | supported |
| `u22`、`u24`、`u26` | Ubuntu | amd64、arm64 | UEFI | supported |

```bash
farrow image list
farrow image info d13
farrow image info d13:stable
farrow image info el9@9.7
farrow image pull d13@20260810.2566.0
farrow image pull d13 --arch arm64
farrow update
```

Catalog 状态只表达支持策略，不是启动开关：`supported` 表示已通过声明的支持门禁；
`testing` 可在显式测试/风险接受下使用，但不受支持；`deprecated` 只为 EOL 兼容保留；
`unknown` 尚无支持分类。非 `supported` 条目仍可运行，但会打印警告。

拉取时 Farrow 会：

1. 为整条命令读取一次当前本地 Catalog：即本次构建内置的 Catalog，或最近一次
   `farrow update`/`image sync` 激活的 Catalog；
2. 解析 `image[:channel]` 或 `image@version-prefix`，缺省为 `u24:stable`；独立
   `image pull` 使用本机架构，生命周期解析遵循 `vm_arch`；
3. 只有尺寸、SHA-256、qcow2 结构全部匹配时才复用本地文件；
4. 否则只从选定仓库下载 Catalog 指定的准确工件；不可变 Upstream URL 是溯源，不是回退源。

Release 构建默认使用 `https://repo.pigsty.io/farrow`；仅有长参数的 `--mirror` 选择
`https://repo.pigsty.cc/farrow`。优先级依次为 `--repo`、`--mirror`、`FARROW_REPO`、
全球默认仓库，两个官方根都保持规范的签名 Catalog 信任。Farrow 不会自动刷新 Catalog，
因此只有必须下载镜像时命令才需要仓库可达。运行 `farrow update` 可获取、校验并激活选定
仓库当前的 Catalog；更新失败或工件缺失都会直接报错。

## 运行时策略

匹配架构正常使用原生 HVF/KVM，只有一个 Catalog 已知例外：Stock EL8 arm64 的 64K
Granule Kernel 无法通过 Apple HVF 运行，因此 Apple Silicon 会自动选择可见的同架构
TCG。显式外来 `vm_arch` 也会使用 TCG；arm64 宿主上的 amd64 Guest 使用单翻译线程，
以保留 x86 内存序。TCG 结果不能作为性能证据。

EL7 刻意仅支持 Linux/amd64 原生运行。Linux setup 只安装宿主原生 QEMU；外来架构必须
先安装对应 System Emulator 与 UEFI 固件，`plan`、`up`、`recreate` 才会继续。

```bash
farrow image pull d13 --mirror
farrow image pull d13 --repo https://mirror.example/farrow
FARROW_REPO=/absolute/local/repository farrow up
```

未签名仓库必须是本地路径或 HTTPS。HTTP 仓库必须提供可信密钥签名；不可变 Upstream
工件 URL 必须使用 HTTPS。

## 信任与校验

当前普通构建已经内置两把生产校验公钥；私有签名密钥不在源码仓库中。Catalog 激活会拒绝
未知密钥、畸形内容、同 Revision 异内容，以及低于该仓库独立 High-water Mark 的
Revision；只有操作者显式允许时才可降级。

镜像必须是尺寸与 SHA-256 匹配的纯 qcow2，不得有 backing file、外部数据文件、加密或
未知不兼容 Feature。通过校验的 Base Image 变成只读；节点根盘使用 Overlay，永不修改
Base。

```bash
farrow update
farrow image sync https://repo.example/farrow/catalog.json
farrow image sync --allow-downgrade /absolute/repo/catalog.json
farrow image reset
```

`image reset` 恢复二进制内置的 Catalog，但不会清除防回滚 High-water Mark；
`reset-manifest` 作为兼容别名保留。

`farrow update` 立即检查仓库并激活更新的 Catalog。Farrow 不会自动刷新 Catalog；每个版本
内嵌的 Catalog 会一直使用到你运行 update。`image sync` 是指定精确 URL 或文件（含降级）的
恢复路径。

按仓库恢复时要显式传入同一个根目录：

```bash
farrow image sync --repo /srv/farrow --allow-downgrade /srv/farrow/catalog.json
farrow image reset --repo /srv/farrow
```

`--repo` 决定要操作的独立 High-water 槽；省略时依次考虑 `--mirror`、`FARROW_REPO`
与全球编译期默认值。

## 静态仓库格式

发布根刻意保持很小：

```text
farrow/
├── repo.yaml
├── catalog.json
├── catalog.json.minisig       # HTTP 仓库必需，其余可选
└── images/
    └── <image>-<version>-<arch>.qcow2
```

`repo.yaml` 保存人工意图：默认值、别名、Channel、精确版本、架构、启动模式、状态和可选、
只用于溯源的 Upstream URL。`source_user` 表示发行版原始镜像内置的登录账号（例如
`rocky`）；离线归一化
会用它清理并锁定该上游账号，随后把它保留为导入溯源。它不会替换 deployment SSH 用户
（默认 `dba`）。该文件不保存任何生成的摘要或大小；`catalog.json` 保持同一逻辑树，
但为每个 Variant 物化文件名、SHA-256、工件大小和虚拟大小。`repo.yaml` 是 `schema: 1`；
生成的 `catalog.json` 则是 Farrow 内嵌并签名的 Schema-3 Catalog。

```yaml
schema: 1
revision: 1
defaults: { image: u24, channel: stable, arch: native, boot: uefi }
images:
  u24:
    aliases: [ubuntu24, noble, ubuntu]
    channels: { stable: "1" }
    versions:
      "1":
        status: unknown
        variants:
          amd64: {}
          arm64: {}
```

不显式设置 `file` 时，两个预期工件分别是 `images/d13-1-amd64.qcow2` 与
`images/d13-1-arm64.qcow2`；已有自定义文件可在 Variant 中覆盖为安全 basename。

Channel 与数值前缀都是可移动 Selector。存在精确 Key 时优先精确匹配；否则只在点分量
边界匹配并选择数值语义最新版本（`el9@9.7` 选择最新 9.7 Build，`el9@9` 选择最新
9.x Release）。不可变工件身份仍是 `(image, exact version, arch)`：

```text
d13:stable + native
  -> d13@20260810.2566.0 + arm64
  -> images/d13-20260810.2566.0-arm64.qcow2
```

`farrow repo scan` 只读；`build` 执行严格 YAML 校验、完整 qcow2 inspect/check，
并原子替换 Catalog，永不修改 `repo.yaml` 或 QCOW 字节；`verify` 要求新鲜物化结果与
现有 Catalog 逐字节一致。`build`、`verify` 需要本机 `qemu-img`，`scan` 不需要。
应在安装 QEMU 的机器上构建，再按“工件在前、Catalog 在后”的顺序 rsync 发布。

## 本地布局与导入

镜像位于 `FARROW_HOME/images`（默认 `~/.farrow/images`）：各 Family 目录保存下载工件，
`manifests/` 保存当前激活的 Catalog 与每个仓库独立的 High-water 状态，`local/` 与
`local-images.json` 保存导入镜像。

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
不是候选；`destroy`、`destroy --purge` 与 `purge` 后镜像仍保留缓存。

使用 `go run ./tools/catalogexport /absolute/new/catalog.json` 可逐字节导出编译期
Schema-3 Catalog。
公开 Catalog 若使用相同版本，就必须使用完全相同的字节；同版本不同内容会按 equivocation
拒绝。Release 签名与镜像 Catalog 签名仍属于不同信任域。
