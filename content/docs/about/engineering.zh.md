---
title: 工程与发布
description: 源码边界、构建测试门禁、镜像归一化、发布输出与证据纪律。
weight: 30
icon: fa-solid fa-screwdriver-wrench
---

## 仓库边界

Farrow 源码仓库只保留代码、测试、构建/打包定义、法律声明与简短入口 README。本网站是
用户、设计、运维与发布文档的唯一权威位置。原始 Review 记录、历史 Scratch Inventory、
Demo 目录、生成二进制与 Release 输出树都不是源码输入，不应提交。

以下输出随时可重建：

- `bin/`：开发构建；
- `dist/` 与 `.goreleaser-*`：Release/Snapshot Staging；
- 根目录 `farrow`、`farrow-hosts-helper`、`catalogsign` 二进制；
- Hugo 的 `public/` 与 `resources/`。

## 构建与源码门禁

```bash
make build
make module-check
make shell-check
make test
make race
make vet
make staticcheck
make vuln
make cross-check
make image-pipeline-test
make license-check
```

`make check` 汇总这些门禁。源码绿色不等于真机验证；macOS HVF、Linux KVM/网络、软件包
消费、Release 发布与线上渲染仍是不同门禁。

## Release 与软件包契约

`packaging/`、`.goreleaser.yaml` 与 `.github/workflows` 属于源码；它们生成的目录不是。
Archive 与 Linux Package 只携带匹配二进制、`LICENSE`、精简源码 README、
构建元数据，以及根据 `go.mod` 锁定模块版本重建的准确上游许可证字节。生成的许可证文本
在 Archive 中位于 `licenses/`，不作为源码跟踪。详细文档留在版本化网站，不再复制到每份
二进制 Payload。

构建成功绝不代表发布。Commit、Tag、Archive/Package、签名/证明、上传、CI 与公开消费是
彼此独立的门禁。
稳定版 Local Release 命令还要求干净且已打标签的 Commit 与 `origin` Remote，因为
GoReleaser 会记录仓库身份。

## 镜像归一化

`packaging/image-pipeline/` 只接受显式本地 qcow2，不下载也不上传。它复制并哈希源文件，
强制 qcow2 解析，拒绝 Backing/External/Encryption/未知 Feature，运行 `qemu-img check`，
并可在显式 QEMU Sandbox 中做无网络 Offline Guest Mutation。UID/GID 88 冲突会拒绝，
不会含糊改写。

Catalog 逐字节导出命令：

```bash
go run ./tools/catalogexport /absolute/new/catalog.json
```

导出器原子且绝不覆盖。Catalog 签名与应用 Release 签名使用不同密钥与信任域。

## 证据纪律

历史 M0–M4 记录在实现期有价值，但不是产品文档。可长期保留的结论已收敛到[设计](design/)
与[当前状态](status/)。后续源码修改不会自动继承真机证明；每条状态结论都应说明日期、宿主、
路径与剩余门禁。
