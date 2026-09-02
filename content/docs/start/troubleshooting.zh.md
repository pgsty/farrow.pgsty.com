---
title: 故障排查
description: 面向 setup、网络、镜像、漂移、中断状态与 SSH 的简短安全手册。
weight: 30
icon: fa-solid fa-life-ring
---

先做只读检查：

```bash
farrow doctor --json
farrow network status --json
farrow status --json
```

## 找不到 Inventory

第一次部署时，在 `farrow.yml`/`pigsty.yml` 所在目录运行 `plan`、`up`、`validate`，
传入 `-f /path/to/file`，或运行 `farrow init` 生成一份。状态存在后，`plan`、`up`、
`reload`、`recreate` 可回退到已应用规格；status、start、stop、SSH 与 destroy 始终使用
已应用状态。如果 `status` 报告 `no deployment state found`，说明所选 `FARROW_HOME`
从未运行过 `up`。

## setup 需要 sudo

提示前一行会说明具体宿主变更；特权步骤开始时 Farrow 会直接把交互终端交给 sudo。
自动化环境需要已有凭据或合适的 NOPASSWD，再使用 `--yes`。

## 原生加速或兼容运行时不可用

原生路径需要 macOS HVF 或 Linux KVM。只有显式外来 `vm_arch` 或内置镜像/宿主兼容规则
才会选择 TCG；任意原生失败绝不会静默回退。Homebrew QEMU 包含两个 System Emulator；
Linux setup 只安装宿主原生家族，因此外来 Guest 还需要对应 `qemu-system-*` 与固件。

`plan`、`up`、`recreate` 会在任何破坏性修改前验证所选模拟器与固件。TCG 性能结果没有
参考意义。

## 网络是 partial 或 invalid

不要手工删宿主文件，先查看受控清理计划：

```bash
farrow network status --json --verbose
farrow network uninstall
```

确认只包含 Farrow 自有路径后再加 `--yes`。Linux bridge smoke 失败会自动回滚安装；
只有出现 `automatic rollback failed` 才表示必须人工检查。

## Linux bridge helper 失败

```bash
id
stat -c '%U:%G %a %n' /usr/lib/qemu/qemu-bridge-helper
dpkg-statoverride --list /usr/lib/qemu/qemu-bridge-helper
```

Debian/Ubuntu 使用 `root:<调用者可用组> 4750`。桌面系统通过 ACL 获得 `/dev/kvm`
权限时，调用者不必静态加入 `kvm` 组。

## plan 报 recreate 或 missing

`recreate` 表示节点定义已改变：先用 `farrow plan` 查看，再运行 `farrow recreate <node>`。
终端上该命令会要求输入 `recreate` 确认，`--force` 仅用于脚本。`missing` 只是报告：
恢复主机条目，或运行 `farrow destroy <node>`。

## 节点未就绪

`up`、`start`、`recreate` 会等待每个 Guest 完成 Bootstrap。若有节点失败，命令以 5 退出，
并报告 `N of M node(s) failed: <node> (<stage>: <error>)`。阶段为 `prepare`、`start`、
`readiness`、`stop`；`readiness` 失败会指出 Guest 自身的 Bootstrap 阶段：

| Bootstrap 阶段 | 通常意味着 |
|---|---|
| `identity`、`hosts` | 镜像或其 cloud-init 未按预期运行 |
| `management-network` | 宿主没有出网能力，或需要代理 |
| `data-disks` | 磁盘定义有误，或镜像缺少所请求的 `mkfs` |
| `shares` | 共享目录无法挂载 |
| `control-ssh` | 控制节点 SSH 密钥无法安装 |
| `private-network` | 固定 IP 网卡未能启用；检查 `farrow network status` |
| `ready` | 无法写入 ready 标记 |

`guest bootstrap failed during data-disks (exit status 1)` 指出失败阶段；
`guest did not become ready within 3m0s: <last ssh error>` 表示 Guest 始终没有应答。
先看 Guest 串口与 QEMU 输出，再检查记录的状态：

```bash
farrow logs <node>                  # 串口控制台
farrow logs <node> --source qemu    # QEMU 诊断
farrow status
```

在 `prepare` 阶段失败的节点从未加入 deployment。给 `up` 或 `reload` 加上
`--rollback` 可在同一次运行中清除它们的残留产物，结构化输出会在 `rolled_back` 中列出。
`--no-wait` 跳过就绪等待，QEMU 运行后即返回。

## SSH 失败

检查 `farrow status`、`farrow ssh-config` 与串口日志。Farrow 自身 SSH 使用回环管理端口；
Ansible 直连固定 IP。
`doctor` 的通用可用性扫描会排除已应用部署保留的固定 IP；`up` 与 `start` 仍会拒绝
已经接受 SSH 的新增节点或已停止节点地址。

## Catalog 或镜像校验失败

当前二进制已内置 active 与 standby Catalog 公钥。未知签名者、版本回滚/同版本异内容、
工件尺寸/SHA 不符、qcow2 结构不安全属于不同完整性错误。使用正确签名仓库，或通过
`farrow image import --sha256 ...` 导入；不要直接向 `~/.farrow/images` 复制字节。

## 命令被中断

运行 `farrow status`。可证明存活或死亡的运行时会按完整进程身份收敛；歧义进程继续阻塞。
不要只凭状态文件里的 PID 就杀进程。

如果记录的 QEMU 进程仍存在，但 QMP Socket 缺失，应先保留证据并查看串口/QEMU 日志，
再决定是否用 `stop` 收敛。不要手工删除运行时 Socket 或状态文件。

提交问题时请包含准确命令与退出码、`farrow version`、上面三份 JSON、宿主系统/架构与
QEMU 版本。
