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

直接运行 `farrow` 会打印帮助并以 2 退出；`farrow image` 这样的裸命名空间同理。
JSON/YAML 模式下，空命名空间返回结构化用法错误。显式 `--help` 始终输出供人阅读的
帮助文本。

## 命令

| 范围 | 命令 |
|---|---|
| 准备 | `setup`、`init`、`validate`、`doctor` |
| 生命周期 | `plan`、`up`、`start`、`stop`、`restart`、`reload`、`recreate`、`status`、`destroy` |
| 访问 | `ssh`、`exec`、`logs`、`provision`、`ssh-config`、`hosts install/uninstall` |
| 镜像 | `update`、`image list/info/pull/import/sync/prune/reset`、`repo scan/build/verify` |
| 宿主网络 | `network status/install/uninstall` |
| 其他 | `version`、`completion` |

没有命令会隐式刷新 Catalog。`update` 获取配置仓库的 Catalog，校验并激活；
`image sync` 是为精确 URL 或文件准备的显式恢复路径。普通命令只使用当前本地 Catalog。
两者都不更新 Farrow 可执行文件。

常用命令提供作用域明确的短别名：

| 命令 | 别名 | 命令 | 别名 |
|---|---|---|---|
| `setup` | `s` | `validate` | `v` |
| `plan` | `pl` | `recreate` | `rc` |
| `status` | `st` | `destroy` | `de` |
| `ssh-config` | `sc` | `image` | `images`、`im` |
| `doctor` | `dt` | `network` | `n`、`net` |
| `exec` / `logs` | `ex` / `l` | `version` | `ver` |

`up`、`ssh`、`init`、`start`、`stop`、`restart`、`reload`、`provision`、`hosts`、
`completion` 没有别名。命名空间内部，`hosts` 与 `network` 的 install/uninstall 使用
`i`/`u`，`network status` 使用 `st`；`image` 使用 `list=ls`、`info=in`、`pull=p`、
`prune=pr`、`sync=sy`、`import=i`。`image reset` 保留 `reset-manifest` 作为兼容别名。

使用已应用状态的命令可在任意目录运行。配置来源由命令决定，`-f` 刻意不做全局参数：

| 命令 | 期望状态来源 |
|---|---|
| `setup [template]` | 显式 `-f`，否则发现配置，否则生成 `meta`；模板与 `-f` 互斥 |
| `init [template]` | 生成新 Inventory，不读取期望状态；`--force` 显式替换输出文件 |
| `validate` | 显式 `-f`，再发现配置；绝不回退到已应用状态 |
| `plan`、`up`、`reload`、`recreate` | 显式 `-f`，再发现配置，最后回退到已应用规格 |
| 其他生命周期/访问命令 | 不读取期望配置；使用已应用状态 |

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
| `--force`（`init`、`destroy`、`recreate`） | 覆盖生成文件或跳过输入确认词；因为 `-f` 用于选择 Inventory，所以只保留长参数 |
| `-n`、`--no-wait` | QEMU 运行后即返回，不等待 Guest 启动完成 |
| `--rollback`（`up`、`reload`） | 清除本次运行中 prepare 失败节点的残留产物 |
| `--delete-persistent` | 整体销毁时也删持久盘；不能与节点选择器一起使用 |
| `--purge` | 整体处置：删除磁盘、密钥与 deployment 状态，保留镜像 |

`--force`、`--rollback`、`--remove`、`--allow-downgrade`、`--sudo`、
`--delete-persistent`、`--purge` 等低频或扩大风险边界的参数只保留长版本。读取
Inventory 的命令中 `-f` 始终选择文件；`logs -f` 保留惯用的 `--follow`。
`-n` 始终表示 `--no-wait`，`-d` 始终表示 Dry-run。

如果失败命令尚未输出更丰富的类型化结果，结构化模式会先输出一份包含 `error` 与
`message` 的对象，再返回约定的非零退出码；已经携带失败状态的结果后面绝不会追加第二份
JSON/YAML 文档。

`plan` 是只读操作，即使 action 为 `recreate` 或 `blocked-removal` 也返回成功；自动化必须
检查 action 与 `create`、`recreate`、`missing` 字段。`up` 会创建缺失节点、启动已停止
节点、复查运行中节点的就绪状态，并根据完整的 applied deployment 重写 Farrow 安装的
SSH 客户端配置；`recreate` 同样执行全量刷新，节点级 destroy 删除旧条目，整体 destroy
移除该配置。`start` 启动已停止节点并复查运行中节点的就绪状态，不触碰 SSH 客户端配置。
破坏性 drift 返回冲突，并给出下一步命令：先 `farrow plan`，再 `farrow recreate <node>`
或 `farrow destroy <node>`；终端上这两条命令会要求输入确认词，`--force` 仅用于脚本。
如果 VM 生命周期成功但 SSH 客户端配置无法写入，结构化输出会携带 VM 状态并报告部分
`ssh_config` 失败，退出码为 5。

多节点操作中若有节点失败，退出码为 5，并报告
`N of M node(s) failed: <node> (<stage>: <error>); ...`。阶段为 `prepare`、`start`、
`readiness`、`stop`；`readiness` 失败会追加 `run \`farrow logs <node>\` for the guest
console`。结构化输出携带 `failures[]`（`node`、`stage`、`error`）；当 `--rollback` 清除了
从未提交节点的 prepare 产物时，还会带上 `rolled_back`。参见
[节点未就绪](../../start/troubleshooting/#节点未就绪)。

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
| 5 | 节点操作或生命周期后集成部分完成 |
| 6 | 资源冲突 |
| 7 | 完整性或属主失败 |
| 130 | 被中断（SIGINT/SIGTERM） |

`ssh` 与 `exec` 会透传远端程序退出码；但 SSH 保留的传输失败码 255 会被 Farrow 映射为
运行时失败 1。
