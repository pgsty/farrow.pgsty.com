---
title: 设计
description: Farrow 的单 deployment 架构、网络、状态与安全边界。
weight: 10
icon: fa-solid fa-compass-drafting
aliases: [/docs/concepts/, /docs/concepts/networking/, /docs/concepts/storage/, /docs/concepts/safety/, /docs/architecture/, /docs/architecture/overview/, /docs/architecture/networking/, /docs/architecture/security/]
---

## 一个有用的抽象

Farrow 把一份 Pigsty Inventory 启动成一套本地 QEMU deployment。它刻意不再拥有
project marker、项目注册表、租约模型、Provider Layer 或第二种配置格式。

状态位于当前 Unix 用户的 `~/.farrow`。产品假设每台电脑只有一套运行中的 Pigsty；
这并不是 root 强制的跨用户单例。

## 节点级收敛

Farrow 只提取已记录的 VM 与 Pigsty 原生字段，计算逐节点哈希，并保存应用状态与完整
进程身份。新增节点增量创建；`up` 也会启动选中的已停止节点，运行中同伴不受影响；
变更需要显式节点重建；
配置缺席永远不授权删除。

## 运行时选择

Guest 架构是部署级期望状态。省略或 `native` 跟随宿主；显式 `amd64`/`arm64` 会准确
选择对应 Catalog 工件。HVF/KVM 原生加速仍是默认路径；外来架构或 Catalog 已知的
镜像/宿主不兼容规则才会选择固定 TCG Profile。没有用户可传的 Accelerator 参数，也不会
因任意原生失败静默回退。

实际架构与加速器保存在每个 QEMU Invocation 中，并通过 `status` 展示。执行破坏性
recreate 前，Farrow 会证明所选 QEMU 二进制与版本、网络后端、镜像字节、启动模式与固件。
以后若新二进制改变运行时策略，也不能把新旧节点混跑：Runtime Drift 必须整体重建。

## 双网卡与一个固定子网

管理网卡负责 DHCP、DNS、出网与回环 SSH；固定 IP 网卡负责宿主、节点间与 Ansible
流量。macOS 使用 socket_vmnet；Linux 优先跟随当前 NetworkManager，否则使用
systemd-networkd，并通过发行版 bridge helper 接入。若 networkd 尚未启动，只有在
Activation-safety 扫描证明现有 Unit 不会接管真实宿主链路后才启动。

Debian helper 会临时、可逆地限制给调用者真实加入的组。setup 必须通过一次非特权
QEMU bridge smoke；失败后自动回滚安装。

## 安全边界

QEMU 与所有 Guest 工件都以调用者身份运行。root 仅用于宿主网络与可选 hosts publisher。
销毁必须同时匹配属主、路径包含、节点身份、QMP/进程身份与工件白名单；任何歧义都会停止。
