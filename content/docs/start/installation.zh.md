---
title: 安装
description: 构建 Farrow、确认原生虚拟化，并理解一次性的宿主准备。
weight: 10
icon: fa-solid fa-download
aliases: [/docs/start/upgrade/]
---

## 当前可用的安装方式

从当前工作区构建：

```bash
cd /path/to/farrow
make build
export PATH="$PWD/bin:$PATH"
farrow version
farrow doctor
```

目前没有公开的 v1 软件包。不要用来源不明的二进制替换同一次构建产生的 `farrow` 与
`farrow-hosts-helper`。

源码构建需要 Go 1.27.x。macOS 还需要已经安装好的 Homebrew；Farrow 可以通过 Homebrew
安装 QEMU，但不会自行引导安装 Homebrew。当前编译默认镜像仓库是开发宿主
`https://m0/farrow`；需要其他可达镜像时使用 `FARROW_REPO` 或 `--repo`。

## 宿主要求与带日期的证据

HVF/KVM 原生加速是正常路径。任意原生失败绝不会被静默改成模拟重试；只有显式外来
`vm_arch`，或 Catalog 已知的 EL8 arm64/Apple Silicon 不兼容规则，才会选择 TCG。
TCG 结果只可作为兼容性证据，不能作为性能证据。

| 宿主 | 最近一次记录的证据（2026-08-27） |
|---|---|
| macOS arm64 | 已用 HVF、QEMU 11.1、socket_vmnet 验证 |
| Ubuntu 26.04 amd64 | 已用 KVM、QEMU 10.2.1、NetworkManager 验证 |
| macOS arm64 上的 Rocky Linux 8.10 arm64 Guest | 已用自动 TCG 与完整 readiness 验证 |
| 其他 Debian/Fedora/EL9 amd64 | 已实现；依赖前应重新做原生验证 |
| macOS amd64、Linux arm64 | 已交叉构建和单测，尚无当前真机证据 |

macOS 需要 Homebrew。Linux 需要 systemd，以及 NetworkManager 或
systemd-networkd 之一。`farrow setup` 会通过 Homebrew、APT 或 DNF 安装缺失依赖。

## setup 会改什么

可以先看 dry-run：

```bash
farrow setup --dry-run
farrow setup --yes
```

setup 可能安装 QEMU 依赖、准备固定 IP 网络、安装权限很窄的 hosts publisher。
所有宿主变更与 sudo 原因都会先打印。

自动化环境使用 `--yes`。非交互 sudo 可以来自已缓存凭据或合适的 NOPASSWD 策略；
每一条真实特权命令仍使用明确的绝对路径 argv。

setup 是幂等的：健康的重复执行只会复用网络与 helper。
源码测试与这份带日期的真机矩阵是不同门禁：重新构建更新的工作树不会自动刷新上表。
