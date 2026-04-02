<!-- edit by codex -->

# Orange Pi 6 Plus Cix Build Guide

This note explains how to use `build.sh` in this repo for `Orange Pi 6 Plus` (`BOARD=orangepi6plus`), how to rebuild only the kernel, and what build artifact is produced at the end.

## 1. What `build.sh` does

`build.sh` is the top-level entry of the Orange Pi build system.

It loads configuration in this order:

1. `userpatches/config-default.conf` or another `userpatches/config-*.conf`
2. command-line variables such as `BOARD=... BRANCH=...`
3. `scripts/main.sh`

If a variable is not provided, the script opens an interactive menu with `whiptail`.

For `Orange Pi 6 Plus`:

- board config: `external/config/boards/orangepi6plus.conf`
- family config: `external/config/sources/families/cix.conf`
- supported branches: `current`, `next`

## 2. Quick start

Interactive mode:

```bash
sudo ./build.sh
```

Recommended selections for `Orange Pi 6 Plus`:

- Board: `orangepi6plus`
- Build target: `kernel` or `image`
- Branch: `current`
- Release: `bookworm`

Non-interactive mode:

```bash
sudo ./build.sh BOARD=orangepi6plus BUILD_OPT=kernel BRANCH=current KERNEL_CONFIGURE=no
```

If you want to edit kernel options with menuconfig:

```bash
sudo ./build.sh BOARD=orangepi6plus BUILD_OPT=kernel BRANCH=current KERNEL_CONFIGURE=yes
```

Build a full flashable image:

```bash
sudo ./build.sh BOARD=orangepi6plus BUILD_OPT=image BRANCH=current RELEASE=bookworm BUILD_DESKTOP=no BUILD_MINIMAL=no KERNEL_CONFIGURE=no

sudo ./build.sh BOARD=orangepi6plus BUILD_OPT=image BRANCH=next RELEASE=noble BUILD_DESKTOP=yes DESKTOP_ENVIRONMENT=gnome DESKTOP_ENVIRONMENT_CONFIG_NAME=config_base DESKTOP_APPGROUPS_SELECTED="3dsupport browsers desktop_tools remote_desktop" KERNEL_CONFIGURE=no GITEE_SERVER=yes DOWNLOAD_MIRROR=china

sudo ./build.sh BOARD=orangepi6plus BUILD_OPT=image BRANCH=next RELEASE=noble BUILD_DESKTOP=yes GITEE_SERVER=yes DOWNLOAD_MIRROR=china


```

If you are in mainland China:

```bash
sudo ./build.sh BOARD=orangepi6plus BUILD_OPT=image BRANCH=current RELEASE=bookworm BUILD_DESKTOP=no GITEE_SERVER=yes DOWNLOAD_MIRROR=china
```

## 3. Common build targets

`BUILD_OPT=kernel`

- rebuilds only the kernel package
- output mainly goes to `output/debs/`
- for Cix boards it also exports `Image` and `dtb` to `output/cix/`

`BUILD_OPT=rootfs`

- builds the root filesystem and packages
- does not finish as a complete flash image

`BUILD_OPT=image`

- builds kernel + rootfs + final image
- this is the target you want for a flashable image

## 4. Important parameters

`BOARD=orangepi6plus`

- target board, required for Orange Pi 6 Plus

`BRANCH=current`

- Cix 6.1 kernel
- better first choice for stability

`BRANCH=next`

- Cix 6.6 kernel
- newer kernel branch

`RELEASE=bookworm`

- Debian Bookworm rootfs
- default supported Cix release

`BUILD_DESKTOP=yes|no`

- `no` for server/CLI image
- `yes` for desktop image

`BUILD_MINIMAL=yes|no`

- only used for CLI images

`KERNEL_CONFIGURE=yes|no`

- `yes` opens kernel `menuconfig`
- `no` uses the existing config directly

## 5. Android container support

This repo already had most Linux container basics enabled for Cix:

- namespaces
- cgroups
- seccomp
- overlayfs
- squashfs
- veth
- bridge
- NAT

The missing part for Android containers was mainly Binder support.

The repo now keeps the official Cix kernel configs unchanged by default.

If you want Android container support, apply this patch before building:

```bash
git apply fork_patch/0001-cix-enable-kernel-android-support.patch
```

The patch enables the following for Cix kernel configs:

- `CONFIG_ANDROID=y`
- `CONFIG_ANDROID_BINDER_IPC=y`
- `CONFIG_ANDROID_BINDERFS=y`
- `CONFIG_ANDROID_BINDER_DEVICES="binder,hwbinder,vndbinder,anbox-binder,anbox-hwbinder,anbox-vndbinder"`

Additionally for `6.6 next` the patch enables:

- `CONFIG_ASHMEM=y`

Notes:

- This patch is the kernel-side prerequisite for Android containers such as Waydroid.
- It does not install Waydroid or Android userspace automatically.
- After flashing the image, you still need to install container userspace packages in the running system.

## 6. Final build artifact

For `BUILD_OPT=image`, the final result for Cix is a flashable system image:

- path: `output/images/<version>/<version>.img`
- optional compressed file: `output/images/<version>/<version>.img.xz`
- checksum: `output/images/<version>/<version>.img(.xz).sha`

So yes: when you choose `BUILD_OPT=image`, the final artifact is a Cix system flash image for Orange Pi 6 Plus, not only a kernel package.

For `BUILD_OPT=kernel`, the result is not a full image. It is mainly:

- kernel `.deb` packages in `output/debs/`
- `Image` and `dtb` files in `output/cix/`

## 7. Suggested workflow

First verify the kernel can compile:

```bash
sudo ./build.sh BOARD=orangepi6plus BUILD_OPT=kernel BRANCH=current KERNEL_CONFIGURE=no
```

Then build the full image:

```bash
sudo ./build.sh BOARD=orangepi6plus BUILD_OPT=image BRANCH=current RELEASE=bookworm BUILD_DESKTOP=no KERNEL_CONFIGURE=no
```

If Android container userspace is your goal, start from `current` first. After the kernel and image build succeed, we can continue with the Waydroid-side package and runtime setup.
