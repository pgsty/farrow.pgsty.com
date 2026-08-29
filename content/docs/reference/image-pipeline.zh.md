---
title: 镜像流水线
description: 不下载、不上传、不签名地校验或离线归一化显式 qcow2 Candidate。
weight: 40
icon: fa-solid fa-shield-halved
---

`packaging/image-pipeline/` 接受一份已下载的不可变 qcow2 与独立获得的 SHA-256。它绝不
下载、上传、修改 Farrow 运行时/网络状态、读取签名密钥，也不会把镜像标成 `supported`。

## 模式

- `validate`：复制并重哈希，强制 qcow2 检查，校验单元素 Backing Chain，运行
  `qemu-img check`，输出明确不可发布的证据 Bundle；不会修改 Guest 凭据。
- `offline`：额外在 Staged Copy 上使用 libguestfs `virt-customize --no-network` 与
  `virt-cat`。它拒绝无关 UID/GID 88 占用，归一化锁定的 `dba`/`admin` 身份，关闭密码与
  Root SSH，清理密钥/历史/Host Identity/cloud-init Cache，恢复定向 SELinux Label，
  并回读确定性 Marker。

```bash
SOURCE_DATE_EPOCH=1787486400

./packaging/image-pipeline/build.sh \
  --mode validate \
  --source /absolute/source.qcow2 \
  --expected-sha256 <digest> \
  --output /absolute/new/evidence-directory \
  --name u24 --release 20260801.0.0 --arch amd64 \
  --source-user ubuntu --boot uefi \
  --source-uri https://immutable.example/source.qcow2 \
  --artifact-url 'https://images.example/u24/{sha256}.qcow2' \
  --license NOASSERTION \
  --source-date-epoch "$SOURCE_DATE_EPOCH" \
  --manifest-version 2026082903
```

Source/Output 必须是绝对路径；Source 必须 Canonical、普通、非符号链接、复制期间稳定，
且不超过 16 GiB；Output 必须不存在。Builder 使用相邻排它锁、0700 Staging 与一次最终
Rename；失败只删除受保护的 Staging。

成功 Bundle 包含只读 qcow2、Recipe、SLSA Provenance、SPDX Boundary SBOM、状态为
`testing` 的 `manifest-candidate.json`、Validation Evidence 与 Checksums。签名刻意位于
流水线之外。固定输入/工具下 validate 模式逐字节可复现；offline Mutation 必须构建两次
并比较，才能成为 Release Evidence。
