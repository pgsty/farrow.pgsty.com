---
title: 开始使用
linkTitle: 开始使用
description: 两条命令启动 Farrow；其余管理、排障、镜像、构建与清理按需阅读。
weight: 10
icon: fa-solid fa-rocket
cascade:
  type: docs
---

安装 Farrow 后，新用户只需阅读[快速上手](tutorial/)：

```bash
farrow setup
farrow up
```

公开软件包状态见[当前状态](../about/status/)；开发者与源码审查者可使用
[从源码构建](source-build/)。

其余内容按任务拆开：

1. [日常管理](operations/)：状态、访问、启停、变更、缩容与销毁。
2. [故障排查](troubleshooting/)：从只读诊断到常见问题修复。
3. [镜像仓库](images/)：选择镜像、使用镜像站、导入与清理缓存。
4. [从源码构建](source-build/)：开发者构建、检查与本地 PATH。
5. [卸载与清理环境](uninstall/)：移除 deployment、镜像、网络与状态。
