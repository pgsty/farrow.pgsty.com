---
title: 从源码构建
description: 为开发与审查构建 Farrow，并运行完整源码检查。
weight: 50
icon: fa-solid fa-code-branch
---

普通用户应优先使用正式 Release 的 Archive 或系统软件包。本页只面向开发者与源码审查；
Farrow 当前仍是 pre-1.0，公开软件包状态见[当前状态](../../about/status/)。

## 构建

当前源码准确固定 Go 1.27.1；此外需要 Git、Make 与标准构建工具。在源码工作区执行：

```bash
cd /path/to/farrow
make build
export PATH="$PWD/bin:$PATH"
farrow version
```

`make build` 在被 Git 忽略的 `bin/` 下生成同一次构建配套的 `farrow` 与
`farrow-hosts-helper`。不要混用来自不同 Commit 或不同 Release 的两个二进制。

然后回到一个独立实验目录：

```bash
mkdir -p ~/farrow-lab && cd ~/farrow-lab
farrow setup
farrow up
```

## 完整检查

提交源码改动前运行：

```bash
make check
```

该门禁包含单元与 Race 测试、Vet、Staticcheck、漏洞检查、跨平台构建以及 Release、Catalog、
镜像流水线等静态契约。通过源码检查不等于已经发布软件包，也不等于完成真机生命周期验证。

发布工程、依赖许可证与边界说明见[工程说明](../../about/engineering/)。
