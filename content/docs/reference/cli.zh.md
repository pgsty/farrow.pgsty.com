---
title: 命令行
description: Farrow 命令、关键参数、结构化输出与退出码。
weight: 20
icon: fa-solid fa-terminal
aliases: [/docs/reference/json-api/]
---

```text
farrow [--json|--yaml] [-v|--verbose] <command> [flags] [node...]
```

已安装的二进制是当前版本最准确的参考。每一条可见命令都自带操作边界与可复制样例：

```bash
farrow --help
farrow setup --help
farrow image pull --help
```

文本模式下，直接运行 `farrow` 或 `farrow image` 这样的命名空间会展示上下文帮助并成功
退出；JSON/YAML 模式下，空命名空间返回结构化用法错误。显式 `--help` 始终输出供人阅读的
帮助文本。

## 命令

| 范围 | 命令 |
|---|---|
| 准备 | `setup`、`init`、`validate`、`doctor` |
| 生命周期 | `plan`、`up`、`start`、`stop`、`restart`、`reload`、`recreate`、`status`、`destroy` |
| 访问 | `ssh`、`exec`、`logs`、`provision`、`ssh-config`、`ss`、`hosts` |
| 镜像 | `image list/info/pull/import/sync/prune/reset-manifest` |
| 宿主网络 | `network status/install/uninstall` |
| 其他 | `version`、`completion` |

常用命令提供作用域明确的短别名：

| 命令 | 别名 | 命令 | 别名 |
|---|---|---|---|
| `setup` | `s` | `init` | `i` |
| `validate` | `v` | `plan` | `pl` |
| `recreate` | `rc` | `status` | `st` |
| `destroy` | `de` | `provision` | `p` |
| `ssh-config` | `sc` | `hosts` | `h` |
| `image` | `images`、`im` | `doctor` | `dt` |
| `network` | `n`、`net` | `exec` / `logs` | `ex` / `l` |
| `version` | `ver` | `completion` | `cp` |

`up`、`ssh`、`ss` 本身已经很短；`start`、`stop`、`restart`、`reload`
刻意不公开别名；旧 `halt` 仅作为隐藏的 deprecated 兼容命令保留。命名空间内部，
`hosts` 与 `network` 的 install/uninstall 使用 `i`/`u`，`network status` 使用 `st`；
`image` 使用 `list=ls`、`info=in`、`pull=p`、
`prune=pr`、`sync=sy`、`import=i`。涉及恢复边界的 `reset-manifest` 保持完整命令名。

使用已应用状态的命令可在任意目录运行。配置来源由命令决定，`-f` 刻意不做全局参数：

| 命令 | 期望状态来源 |
|---|---|
| `setup [template]` | 显式 `-f`，否则发现配置，否则生成 `meta`；模板与 `-f` 互斥 |
| `init [template]` | 生成新 Inventory，不读取期望状态；`-f/--force` 显式替换输出文件 |
| `validate` | 显式 `-f`，再发现配置；绝不回退到已应用状态 |
| `plan`、`up`、`reload`、`recreate` | 显式 `-f`，再发现配置，最后回退到已应用规格 |
| 其他生命周期/访问命令 | 不读取期望配置；按需使用已应用状态或带属主标记的状态 |

## 关键参数

| 参数 | 含义 |
|---|---|
| `--json`、`--yaml` | stdout 机器可读；进度仍写 stderr；刻意不设短参数 |
| `-v`、`--verbose` | stderr 有界诊断 |
| `-c`、`--cidr` | 为 `init`/`setup` 生成模板或宿主网络检查/安装选择 RFC1918 `/24` |
| `-f`、`--file` | 为读取期望状态的命令选择 Inventory |
| `-r`、`--repo` | 为需要解析下载的命令选择镜像/制品仓库 |
| `-m`、`--mode` | 在提供该参数的命令中选择 macOS `host`/`shared` 网络模式 |
| `-d`、`--dry-run` | 只展示 setup/image 计划，不改变状态 |
| `-y`、`--yes` | 应用已展示的宿主/setup/image 计划 |
| `-f`、`--force`（`init`、`destroy`） | 覆盖生成文件或跳过 destroy 确认 |
| `--force`（`recreate`） | 跳过 recreate 确认；因为 `-f` 用于选择 Inventory，所以只保留长参数 |
| `-n`、`--no-wait` | QMP/进程身份确认后返回，不等 Guest readiness |
| `--delete-persistent` | 整体销毁时也删持久盘；不能与节点选择器一起使用 |
| `--purge` | 整体处置：删除磁盘、密钥与 deployment 状态，保留镜像 |

`--allow-downgrade`、`--sudo`、`--delete-persistent`、`--purge` 等低频或扩大风险
边界的参数刻意只保留长版本；因此整套 CLI 中 `-d` 始终表示 Dry-run。

如果失败命令尚未输出更丰富的类型化结果，结构化模式会先输出一份包含 `error` 与
`message` 的对象，再返回约定的非零退出码；已经携带失败状态的结果后面绝不会追加第二份
JSON/YAML 文档。

`plan` 是只读操作，即使 action 为 `recreate` 或 `blocked-removal` 也返回成功；自动化必须
检查 action 与 `create`、`recreate`、`missing` 字段。`up` 会创建新增节点、启动选中的
已停止节点，并根据完整的 applied deployment 重建默认 marker-owned SSH 配置；`recreate`
同样执行全量刷新，节点级 destroy 删除旧条目，整体 destroy 移除默认集成。`start` 只负责
启动已创建节点。破坏性 drift 仍返回冲突，不会被静默应用。如果 VM 生命周期成功但 SSH
收敛失败，结构化输出会携带 VM 状态并报告部分 `ssh_config` 失败，退出码为 7。

`status` 会为每个节点报告持久化的 `guest_arch` 与 `accelerator`；文本和结构化输出都会
明确显示 TCG。

## SSH 透传与命令补全

`farrow ssh [node] [--] [command ...]` 打开会话或运行可选命令；
`farrow exec [node] [--] <command ...>` 必须给出命令并透传退出码。`--` 之前的展示参数
属于 Farrow，之后的参数属于 OpenSSH 或远端程序。

加载 `farrow completion bash|zsh|fish|powershell` 可获得命令与作用域准确的参数补全，
同时补全命令别名、模板、镜像别名、枚举参数，以及从期望/已应用规格只读解析出的节点名。

## 退出码

| 代码 | 含义 |
|---:|---|
| 0 | 成功 |
| 1 | 运行时失败 |
| 2 | 用法或配置错误 |
| 3 | 缺少宿主能力 |
| 4 | 状态冲突或需要显式收敛 |
| 5 | 多节点部分完成 |
| 6 | 资源冲突 |
| 7 | 完整性或属主失败 |

`ssh` 与 `exec` 会透传远端程序退出码；但 SSH 保留的传输失败码 255 会被 Farrow 映射为
运行时失败 1。
