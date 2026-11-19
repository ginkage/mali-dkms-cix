# mali-dkms-cix: Arm Mali kbase for CIX Sky1, ported to kernel 7.x

This is the Arm Mali CSF kernel driver (`mali_kbase.ko`, DDK **r54p1-11eac0**, platform `sky1`) for the CIX P1 / CD8180 Mali-G720-Immortalis. It is packaged for DKMS and ported to build and run on mainline ARM64 kernels; it is developed on Armbian `7.3.5-edge-cix-p1` with an ACPI boot.

The history starts from CIX's `cixtech/cix_opensource__gpu_kernel`, branch `cix_p1_k6.6_master`, commit `a9c0a47` ("CIX P1 2026Q2 RC4 RELEASE"). New CIX releases can be merged from the `cix` remote. The DDK version must match the userspace `libmali.so.0.54.1` and its `mali_csffw.bin`.

## Changes on top of CIX

- **7.x port:** compiler flags are passed through `KCFLAGS`, because Kbuild ignores `EXTRA_CFLAGS` since 6.15. Kernel API shims in `include/linux/version_compat_defs.h` are version-guarded, so 6.6 still builds. The shim set is the same as `mali-dkms-rockchip`.
- **Page migration:** off on 6.17+, unless the kernel carries the `PGTY_mali_gpu` patch from `mali-dkms-rockchip/kernel-patches`.
- **Cache-clean wait:** bounded at 2 s, with a GPU reset to recover.
- **ACPI coherency:** the Sky1 DSDT declares the GPU `_CCA=1`, so kbase now selects ACE-Lite. Stock kbase used COHERENCY_NONE under ACPI, which reads stale data because the DMA API skips cache maintenance for `_CCA=1` devices.
- **DKMS:** builds against the target kernel (`KDIR=${kernel_source_dir}`), and only on kernels with `CONFIG_ARCH_CIX`.

## Install (manual, until packaged)

The module binds ACPI `CIXH5000`, the same device Panthor binds, and udev autoloads it, so put the blacklist in place **before** installing:

```sh
# /etc/modprobe.d/blacklist-cix-gpu.conf: Panthor mode, mali_kbase only by hand
#blacklist panthor
blacklist mali_kbase
#options linlon_dp enable_render=1

sudo dkms add .            # copies to /usr/src/cix-gpu-kmd-1.0.0
sudo dkms install cix-gpu-kmd/1.0.0
sudo install -m644 /path/to/cix-gpu-umd/usr/lib/firmware/mali_csffw.bin /lib/firmware/
```

kbase loads `/lib/firmware/mali_csffw.bin`, and Panthor uses `arm/mali/arch12.8/mali_csffw.bin`, so the two firmware files don't clash. To switch to the vendor stack, swap the two `blacklist` lines and reboot.
