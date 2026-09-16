---
title: 快速上手
linkTitle: 快速上手
description: 用 up 启动单节点 Farrow，用 ssh 进入，再通过同一份配置增量扩容。
weight: 10
icon: fa-solid fa-play
aliases: [/docs/start/installation/, /docs/start/upgrade/, /docs/start/lab/, /docs/start/pigsty/, /docs/features/]
---

## 安装

Farrow 仍是 pre-1.0。每个 Release 都附带用户态 Installer、Homebrew Formula 与 DEB/RPM
软件包。先从 [Farrow 0.7.0 Release](https://github.com/pgsty/farrow/releases/tag/v0.7.0)
下载资产，再任选一种路径：

```bash
# 从 Release 安装：用户态、无需 sudo、校验 Checksum
curl -fLO https://github.com/pgsty/farrow/releases/download/v0.7.0/install.sh
chmod +x install.sh
FARROW_VERSION=0.7.0 ./install.sh

# Homebrew Formula（作为 Release 资产发布）
brew install --formula ./farrow.rb

# Debian/Ubuntu 与 RHEL 系软件包同样是 Release 资产
sudo apt install ./farrow_0.7.0_linux_amd64.deb
sudo dnf install ./farrow_0.7.0_linux_amd64.rpm
```

GitHub 不会通过 `/releases/latest` 暴露 pre-1.0 预发布版本，因此安装器需要
`FARROW_VERSION=0.7.0`。开发与源码审查可使用[从源码构建](../source-build/)。

## 启动第一个实验环境

在终端的空目录中，启动环境，然后进入虚拟机：

```bash
mkdir -p ~/farrow-lab && cd ~/farrow-lab
farrow up
farrow ssh
```

交互式 `up` 在需要时生成默认单节点 `farrow.yml`，提示准备缺少的宿主依赖，拉取镜像并
启动虚拟机。耗时任务显示进度，完成后给出简短结果和连接命令。中断或部分功能未完成时，
再次执行 `up` 即可继续。

脚本中应先用 `init` 生成配置、用 `setup --yes` 准备宿主。下面说明可选的准备与配置步骤。

## 1. 显式准备宿主（可选）

希望提前查看安装计划或单独准备宿主时，运行：

```bash
farrow setup
```

Farrow 会先展示依赖、固定 IP 网络与权限操作，再一次性请求确认。已经安装的依赖会自动跳过，
所以提前安装 QEMU 可以明显缩短首次 setup。

下载会尊重已配置的 `HTTP_PROXY`、`HTTPS_PROXY`、`ALL_PROXY`、`NO_PROXY` 及其小写形式。
setup 跨 sudo 边界时可能保留已配置的代理变量名，但绝不会打印其中可能带凭据的值。

```bash {tab="自动安装（推荐）" group="host-deps" value="auto"}
farrow setup
```

```bash {tab="macOS：预装 QEMU" value="macos"}
brew install qemu
farrow setup
```

```bash {tab="Debian / Ubuntu：预装 QEMU" value="debian"}
# amd64
sudo apt-get update
sudo apt-get install -y qemu-system-x86 qemu-utils ovmf openssh-client iproute2
farrow setup
```

```bash {tab="RHEL / Fedora：预装 QEMU" value="rpm"}
# amd64
sudo dnf install -y qemu-kvm qemu-img edk2-ovmf openssh-clients iproute
farrow setup
```

ARM64 Linux 将 QEMU 与固件包替换为发行版对应的 arm64 版本即可；准确依赖与宿主支持范围见
[从源码构建](../source-build/)和[当前状态](../../about/status/)。

> [!NOTE]
> 空目录中的 `farrow setup` 会顺便生成默认 `meta` 配置；若当前目录已有 `farrow.yml` 或
> `pigsty.yml`，setup 会直接使用它。

## 2. 配置与启动

### 显式生成配置

如果希望在 setup 之前先查看或选择配置，在空目录中运行：

```bash
farrow init
```

输出示例：

```text
template:  meta
wrote:     /Users/you/farrow-lab/farrow.yml
next:      farrow up
```

不带参数时使用单节点 `meta`。内置 spec 只有四个：

| Spec | 节点 | 地址 |
|---|---:|---|
| `meta` | 1 | `10.10.10.10` |
| `dual` | 2 | `10.10.10.10`–`10.10.10.11` |
| `trio` | 3 | `10.10.10.10`–`10.10.10.12` |
| `full` | 4 | `10.10.10.10`–`10.10.10.13` |

例如 `farrow init full` 生成四节点配置，`farrow init full -c 10.20.30.0/24` 可更换网段。
现有文件不会被覆盖，除非明确传入 `--force`。全部字段见[配置参考](../../reference/configuration/)。

### 启动第一个节点

默认配置的核心只有一个节点：

```yaml
all:
  vars:
    admin_ip: 10.10.10.10
  children:
    nodes:
      hosts:
        10.10.10.10: { nodename: meta }
```

启动它：

```bash
farrow up
```

第一次运行会自动解析、下载并校验默认的 `u24:stable` 镜像，然后等待管理 SSH 就绪。
健康的单节点环境只输出简短结果：

```text
  ✓  1 node ready
connect:   farrow ssh meta
```

现在可以进入虚拟机：

```bash
farrow ssh meta
```

> [!TIP]
> `up` 会自动处理镜像。只有需要切换发行版、使用镜像站或管理缓存时，才需要阅读
> [镜像仓库](../images/)。

可选功能受限时会列出具体问题，仍可进入可用的虚拟机。数据盘按可丢弃的测试存储处理：
`up` 可能清空重建不可用的文件系统，包括持久盘。存放数据前请阅读
[数据盘说明](../../reference/configuration/#数据盘)。

## 3. 扩容与日常操作

在 `farrow.yml` 中继续添加节点：

```yaml
all:
  vars:
    admin_ip: 10.10.10.10
  children:
    nodes:
      hosts:
        10.10.10.10: { nodename: meta }
        10.10.10.11: { nodename: node-1 }
        10.10.10.12: { nodename: node-2 }
        10.10.10.13: { nodename: node-3 }
```

再次执行同一条命令：

```bash
farrow up
```

Farrow 只创建新增的三台节点，不会重启正在运行的 `meta`，并刷新运行中 guest 的
Farrow hosts 与控制节点 SSH 配置：

```text
  ✓  4 nodes ready
connect:   farrow ssh meta
```

随时用 `st`（`status` 的别名）查看状态：

```bash
farrow st
```

```text
NAME       STATE    ADDRESS       IMAGE                CPU  MEMORY
meta       running  10.10.10.10   u24@20260801.0.0     2    4.0 GiB
node-1     running  10.10.10.11   u24@20260801.0.0     2    4.0 GiB
node-2     running  10.10.10.12   u24@20260801.0.0     2    4.0 GiB
node-3     running  10.10.10.13   u24@20260801.0.0     2    4.0 GiB
```

最后销毁整套 deployment：

```bash
farrow destroy
```

终端会要求输入 `destroy` 确认。该命令保留镜像缓存、密钥与声明为 persistent 的数据盘；
彻底清理请看[卸载与清理环境](../uninstall/)。停止、重启、日志、显式变更与缩容见
[日常管理](../operations/)。
