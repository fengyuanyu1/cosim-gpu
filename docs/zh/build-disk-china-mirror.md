[English](../en/build-disk-china-mirror.md)

# 国内构建磁盘镜像加速补丁

## 问题

国内直连环境跑 `./scripts/run_mi300x_fs.sh build-disk` 时，VM 内 `apt`
会从 `us.archive.ubuntu.com` 拉包，常因网络波动挂住（Packer
`Timeout waiting for SSH` 或 provisioner 装 ROCm 时退出）。

## 应用补丁

```bash
cd gem5-resources
git apply ../scripts/patches/0001-user-data-cn-mirror.patch
```

回滚：

```bash
cd gem5-resources
git apply -R ../scripts/patches/0001-user-data-cn-mirror.patch
```

切换其他镜像源：修改 patch 里的 URI 后重新 apply。
