---
title: Farrow 文档
linkTitle: 文档
description: 两条命令启动 Farrow，再按需查阅管理、配置与命令契约。
weight: 10
icon: fa-solid fa-book
cascade:
  type: docs
---

Farrow 把一份 Pigsty 兼容的 Inventory 启动成固定 IP 的 QEMU 虚拟机。每个 Unix
用户只有一套 deployment，状态位于 `~/.farrow`，因此生命周期与 SSH 命令可在任意
目录运行。

按任务选择最短路径：

- **[开始使用](start/)**：两条命令启动第一个实验环境，再按需管理与排障。
- **[参考](reference/)**：Inventory 字段、命令、参数、输出与退出码。
- **[关于](about/)**：简化设计、真机验证、已知限制与发布门禁。

新手直接跟随[快速上手](start/tutorial/)即可。正常路径只有两条命令：`farrow setup`
和 `farrow up`。

> [!IMPORTANT]
> Farrow 仍是 pre-1.0。当前源码、真机验证、打包、发布与线上站点是不同门禁。
> 依赖开发构件前请阅读[当前状态](about/status/)。
