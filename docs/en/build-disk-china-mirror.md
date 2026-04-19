[中文](../zh/build-disk-china-mirror.md)

# Disk Image Build Acceleration Patch (China Network)

## Problem

On a China-based direct connection, `./scripts/run_mi300x_fs.sh build-disk`
often hangs because `apt` inside the VM fetches packages from
`us.archive.ubuntu.com` (Packer reports `Timeout waiting for SSH`, or the
provisioner aborts while installing ROCm).

## Apply the patch

```bash
cd gem5-resources
git apply ../scripts/patches/0001-user-data-cn-mirror.patch
```

Revert:

```bash
cd gem5-resources
git apply -R ../scripts/patches/0001-user-data-cn-mirror.patch
```

To use a different mirror, edit the URI in the patch and re-apply.
