---
title: 故障排查
description: 面向 setup、网络、镜像、漂移、中断状态与 SSH 的简短安全手册。
weight: 30
icon: fa-solid fa-life-ring
---

先收集诊断信息（`status` 可能收敛中断的运行时状态）：

```bash
farrow doctor --json
farrow network status --json
farrow status --json
```

## 下载与 PATH 问题

安装器从 GitHub Release 下载程序；`--mirror` 选择的是 Farrow 镜像仓库，不会重定向安装器
下载。如果访问 GitHub 需要代理，在终端将 `HTTPS_PROXY` 或 `ALL_PROXY` 设置为已有代理的
地址。macOS 系统代理设置本身不会替命令行工具配置这些环境变量。

用户态安装器默认写入 `~/.local/bin`。安装后找不到 `farrow`，或版本仍旧时，检查当前使用的
程序路径：

```bash
export PATH="$HOME/.local/bin:$PATH"
command -v farrow
farrow version
```

Homebrew 或系统软件包安装应使用对应渠道的程序。CLI 与配套 `farrow-hosts-helper` 应来自
同一 Release，并保留软件包规定的相对位置。

## 找不到 Inventory

尚无部署时，交互式 `up` 可以生成首份默认配置。显式使用配置时，在
`farrow.yml`/`pigsty.yml` 所在目录运行 `plan`、`up`、`validate`，
传入 `-f /path/to/file`，或运行 `farrow init` 生成一份。状态存在后，`plan`、`up`、
`reload`、`recreate` 可回退到已应用规格；status、start、stop、SSH 与 destroy 始终使用
已应用状态。如果 `status` 报告 `no deployment state found`，说明所选 `FARROW_HOME`
没有已应用部署，可能是首次使用，也可能已执行过 purge。

## setup 需要 sudo

提示前一行会说明具体宿主变更；特权步骤开始时 Farrow 会直接把交互终端交给 sudo。
自动化环境需要已有凭据或合适的 NOPASSWD，再使用 `--yes`。

## 原生加速或兼容运行时不可用

原生路径需要 macOS HVF 或 Linux KVM。只有显式外来 `vm_arch` 或内置镜像/宿主兼容规则
才会选择 TCG；任意原生失败绝不会静默回退。Homebrew QEMU 包含两个 System Emulator；
Linux setup 只安装宿主原生家族，因此外来 Guest 还需要对应 `qemu-system-*` 与固件。

`plan` 无需安装 QEMU 就能解析目标运行时；`up`、`recreate` 在变更 VM 资源前检查
所选模拟器与固件。TCG 性能结果没有参考意义。

## 网络是 partial 或 invalid

完整但未激活的 Farrow 网络可由交互式 `up` 恢复；对于 partial 或 invalid 安装，
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

就绪要求管理 SSH 可用且客机实例身份一致。节点无法创建、启动或连接时，命令会指出
节点和失败阶段，并以 5 退出。先查看日志：

```bash
farrow logs <node>                  # 串口控制台
farrow logs <node> --source qemu    # QEMU 诊断
farrow status
```

数据盘、共享目录、主机名、guest hosts、控制节点 SSH 与私网分别初始化，一项失败不会
阻止管理 SSH 或其他步骤。客机可用时返回 0，并列出具体限制；JSON/YAML 通过
`nodes[].warnings` 暴露这些问题。访问互联网不是就绪的前提。

| 功能限制 | 下一步 |
|---|---|
| 数据盘不可用 | 处理设备暂缺、探测、工具、挂载占用或 I/O 问题，再执行 `up` |
| 共享目录只读 | 修正宿主权限，再执行 `up` 重试可写挂载 |
| Guest hosts 或控制节点 SSH 未完成 | 执行 `up` 刷新托管文件 |
| 私网网卡不可用 | 检查 `farrow network status` 后再次 `up`；管理 SSH 仍可能可用 |

修正原因后重复 `up`：它会重试未完成步骤、原位更新旧客机脚本、跳过健康步骤，不重启
运行中的 VM。无法识别或确认损坏的测试数据文件系统会自动清空重建，**包括持久盘**，
结果会报告旧数据已丢弃。探测失败、忙碌挂载和 I/O 故障不会触发格式化。详见
[数据盘说明](../../reference/configuration/#数据盘)。

0.6.0 初始化中断时，暂存的控制节点 SSH 私钥可能已经丢失。此时 `up` 可以恢复管理访问，
但无法原位重新注入密钥；若需要节点间 SSH，应查看 `farrow plan` 后重建受影响的控制节点。
详见 [0.7.0 升级说明](../../../blog/release/farrow-0.7.0/)。

重复 `up` 也可清理能够确认归属的中断准备残留。`--rollback` 在同一次运行中清除 prepare
失败的产物，并在 `rolled_back` 中列出。`--no-wait` 在 QEMU 运行后即返回，跳过就绪检查、
客机恢复与元数据刷新；后续执行 `up` 补齐。

## SSH 失败

**0.8 开发源码**在启动时会从完好的原私钥恢复缺失的部署公钥，也适用于 0.7.0 创建的
VM。若私钥丢失，需要从备份恢复同一把私钥；Farrow 不会为已有 VM 生成替代身份。
这是宿主侧派生公钥的恢复，与前述旧控制节点中缺失的 Guest 私钥是两种情况。
`up` 也会检查当前安装的 Guest key：文件缺失时报告 `control-ssh` 限制，管理 SSH
仍可使用。恢复原 Guest key 后执行 `up` 会清除限制；向旧 Guest 自动重新注入私钥
仍是待完成的功能。

检查 `farrow status`、`farrow ssh-config` 与串口日志。Farrow 自身 SSH 使用回环管理端口；
Ansible 直连固定 IP。
若已停止 VM 的自动分配管理端口被其他进程占用，下次启动会选择空闲端口并刷新 SSH 别名；
运行中的 VM 保留原端口。SSH 主机密钥信任按 VM 实例 UUID 区分，重建 VM 无需删除无关的
known-host 条目；同一实例的密钥变化仍会校验失败。

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
