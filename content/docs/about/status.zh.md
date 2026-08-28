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

## EL7/EL8 兼容性：2026-08-28

Commit `7c666c7` 在两轮独立 Claude Code Opus 5 max 对抗审查后恢复 EL7/EL8。第一轮因
破坏前运行时预检顺序与签名 Catalog 基线迁移问题给出 BLOCK；修复并补回归测试后，第二轮
给出 PASS，且没有 Required Fix。

Catalog `2026082801` 已在开发仓库签名激活：9 个 Family、17 个镜像工件；包含两份
socket_vmnet Archive 在内的 19 个 Repository Payload 均重新通过完整 SHA 校验。干净客户端
接受了公开签名与准确嵌入摘要。

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

当前镜像仍为 `testing`，只有 EOL EL7 是 `deprecated`。active/standby Catalog 公钥已经
内置，但镜像仓库仍需迁离开发宿主，私钥托管/轮换与 Release 职责也必须在 1.0 前正式落实。
