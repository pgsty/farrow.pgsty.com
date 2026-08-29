---
title: 当前状态
description: 当前简化源码已经通过哪些真机验证、哪些仍未验证、什么阻塞 1.0。
weight: 20
icon: fa-solid fa-list-check
aliases: [/docs/project/, /docs/project/status/, /docs/project/roadmap/, /docs/project/release/, /docs/project/design-history/]
---

Farrow 仍是 pre-1.0。源码测试、带日期的真机重放、软件包、发布、CI 与线上站点是不同门禁。

## 最近一次有记录的真机重放：2026-08-27

该矩阵只属于当天真正执行过的准确 Checkpoint；后续源码或文档修改不会自动继承真机证明。

| 宿主 | 路径 | 结果 |
|---|---|---|
| macOS 26.6.2 arm64 | HVF、QEMU 11.1、socket_vmnet | 单节点与增量四节点通过 |
| Ubuntu 26.04 amd64（`mx`） | KVM、QEMU 10.2.1、NetworkManager | setup、单节点、增量四节点与卸载通过 |

两台宿主均通过固定 IP、SSH readiness、默认 CPU/内存/根盘/数据盘、cloud-init、
stop/start、跨目录操作、扩容时控制节点 boot ID 不变、控制节点横向 SSH、忽略未消费的
Pigsty 变更、配置缺席不删除、显式 destroy。

Linux 还验证了 NOPASSWD 自动化、调用者可用的 Debian helper 权限、非特权 bridge smoke、
四个 tap 挂接时拒绝卸载，以及 destroy 后精确恢复宿主状态。

交互式宿主网络与 hosts 命令现在会自行调用 sudo，外部 `sudo -v` 只是可选优化。
Darwin 的 `network.json` 丢失时，也可用字节一致的接口双份证据、准确 launchd plist 与
已安装二进制摘要重建仅用于卸载的归属计划。

2026-08-28，校准后的工作树通过 unit、race、vet、staticcheck、govulncheck、四平台交叉
构建、模拟镜像流水线边界、许可证校验与 GoReleaser 配置校验。隔离的本地 GoReleaser
Snapshot 还构建并验证了四个平台归档、两个架构的 DEB/RPM、SPDX、Checksum、依赖、权限
以及归档/软件包一致性。没有发布任何产物；这些结果也不会扩展本真机矩阵。

## 0.1.0 就绪重放：2026-08-28

从本次 Review 工作树复制出的隔离干净 Commit 以 `v0.1.0` 打标签后，完整通过稳定版 Local
Release：四平台 Archive、amd64/arm64 DEB 与 RPM、依赖许可证、SPDX、Homebrew Formula、
Installer、Release Metadata、Checksum，以及临时 Cosign Signature/Provenance Round Trip。
生成的用户态 Installer 还在隔离 HOME 中连续执行两次并保持幂等。

对应 Darwin/arm64 二进制处理了两台 QMP Socket 被外部删除的运行中节点：仅 stop/start
这两台，13.7 秒后均恢复 readiness，另外两台同伴的 boot ID 保持不变。随后四台均报告
QMP/进程身份匹配，`plan` 返回 `none`，`doctor` 通过，控制节点到同伴 SSH 正常。在 Ubuntu
26.04 amd64 上，DEB 与 tar Payload 字节一致，包内二进制可运行；干净镜像视图直接使用内置
17 工件 Catalog，不探测私有镜像，KVM/QEMU 10.2.1 doctor smoke 通过。DEB 仅解包，未安装。

这些只是本地 Candidate 证据，并非已发布版本。真实源码 Commit/Tag、远端 CI/OIDC Bundle、
公开仓库与文档 DNS，以及干净宿主的 Homebrew/DEB/RPM 安装仍是独立门禁。

随后一次完整 macOS 出厂清理暴露了源码测试对已安装 `qemu-img` 的依赖：空白宿主无法仅列出
Catalog。现在只有真正需要校验本地 qcow2 字节时才解析 Store；回归测试会显式从 `PATH`
移除 QEMU，完整 `make check` 已在 QEMU/Farrow/网络状态全部不存在时通过。由于该源码修复
晚于上面的 Artifact Candidate，那套产物只保留为历史证据；下方最终源码 Candidate 已从
后续 Commit 重新生成。

## 最终 0.1.0 源码 Candidate：2026-08-29

干净源码 Commit `e8e08315ac7d07e30d1ea1adb98131dde293eacd` 通过 `make check` 与
`goreleaser check`：unit/race、vet、staticcheck、govulncheck、四平台交叉构建、模拟镜像
流水线边界、依赖许可证与 GoReleaser 配置校验全部通过。

一份隔离 Clone 临时在该准确 Commit 上标记 `v0.1.0`，并通过稳定版 Local Release 路径：
四平台 Archive、amd64/arm64 DEB 与 RPM、8 份 SPDX、配套 Helper 摘要、Archive/Package
一致性、Homebrew Formula、Installer、Release Metadata 与 20 项最终 Checksum 全部通过。
解包后的 Darwin/arm64 二进制报告版本 `0.1.0` 与准确源码 Commit。该临时 Tag 没有写回源码
仓库；真实 Tag、Push、远端 CI/OIDC Bundle 与发布仍是彼此独立的 Release 动作。

## Schema-3 Catalog 收口：2026-08-29

Catalog Revision `2026082903` 是当前源码与开发仓库检查点：9 个 Family、27 个分架构工件。
内置 Catalog 与已发布 `catalog.json` 的 SHA-256 同为
`571b1ff9c7d4d42355df3392ea62a339471c2d01d868669a7625fac8b93f245d`；
已发布 `repo.yaml` 也与源码维护文件完全一致。全新 HTTP 与 HTTPS 客户端均接受了生产公钥
`4686B39A40F9B562` 对应的分离签名。

全部 27 个已发布 qcow2（合计 19 GiB）均按 Catalog 完成全量 SHA-256 校验。随后一套空白
临时 Farrow Home 完整下载了 409.3 MiB 的 Darwin/arm64 默认 `d13:stable` 工件，重新计算
摘要，并通过 qcow2 结构与虚拟容量检查。这些只证明发布完整性与客户端路径，不能替代下方
真机生命周期矩阵；本次 Catalog 核验没有重建任何现有 VM。

## EL7/EL8 兼容性：2026-08-28

Commit `7c666c7` 在两轮独立 Claude Code Opus 5 max 对抗审查后恢复 EL7/EL8。第一轮因
破坏前运行时预检顺序与签名 Catalog 基线迁移问题给出 BLOCK；修复并补回归测试后，第二轮
给出 PASS，且没有 Required Fix。

在当时的检查点，Catalog `2026082801` 已在开发仓库签名激活：9 个 Family、17 个镜像
工件；包含两份 socket_vmnet Archive 在内的 19 个 Repository Payload 均重新通过完整
SHA 校验。干净客户端接受了公开签名与准确嵌入摘要。

隔离的 macOS arm64 生命周期重放用内置 TCG 兼容规则启动 Rocky Linux 8.10 arm64，
stop/start 后 44.2 秒达到 readiness，并验证 NetworkManager、固定 IP/无路由/无 DNS、
`dba` UID/GID 88 与 generation/spec marker。EL7 字节、qcow2、BIOS 布局与 4K XFS
Root 已验证；当前 Linux/amd64 原生 Farrow 生命周期仍待重放。

## 仍未完成

- EL9 + NetworkManager + firewalld 真机重放；
- 当前 systemd-networkd 重放；
- 宿主重启持久性；
- macOS amd64 与 Linux arm64 真机；
- 当前 Linux/amd64 原生 EL7 生命周期；
- 当前 9p share 重放；
- 完整的 Pigsty `configure → farrow up → install.yml`；
- 公开 Homebrew/DEB/RPM 安装与发布 CI。

当前内置版本均为 `supported`，只有 EOL EL7 与保留兼容版本 EL9 9.3/9.6、EL10 10.0
为 `deprecated`。active/standby Catalog 公钥已经内置，但镜像仓库仍需迁离开发宿主，
私钥托管/轮换与 Release 职责也必须在 1.0 前正式落实。
