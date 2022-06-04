+++ 
draft = false
date = 2022-04-14T19:14:39+08:00
title = "Run Linux on RISC-V QEMU with Initramfs"
description = ""
slug = ""
authors = ["Xujie Shen"]
tags = []
categories = []
externalLink = ""
series = []
+++

[Run Linux on RISC-V QEMU](https://fishcrap.github.io/blog/posts/run-linux-on-riscv-qemu/) has described the entile process of booting RISC-V linux on QEMU. However, if you want to run an embedded linux on QEMU in only one binany image, you have to build the bootloader and the root file system into linux. This article will show how you can make it.

# Initramfs

There are several methods to generate initramfs for linux. Two construction will be introduced: one is manual and the other is automatic.

## Manually Build with Busybox

### Build Busybox

```bash
git clone git://busybox.net/busybox.git
git checkout 1_34_stable
CROSS_COMPILE=riscv64-unknown-linux-gnu- make menuconfig
<Setting --> Build Options --> Build static binary (no shared libs) --> save>
CROSS_COMPILE=riscv64-unknown-linux-gnu- make -j $(nproc)
CROSS_COMPILE=riscv64-unknown-linux-gnu- make install
```

### Add Directories

```bash
mkdir initramfs
cp -r _install/* initramfs/
```

### Add `init`

```bash
touch init
```

Alter `init` file as following:
```bash
#!/bin/sh

echo "Loading, please wait..."

export PATH="/bin:/sbin:/usr/bin"

[ -d /dev ] || mkdir -m 0755 /dev
[ -d /root ] || mkdir --mode=0700 /root
[ -d /sys ] || mkdir /sys
[ -d /proc ] || mkdir /proc
[ -d /tmp ] || mkdir /tmp
[ -d /mnt ] || mkdir /mnt

# Mount /proc and /sys:
# mount -n proc /proc -t proc
# mount -n sysfs /sys -t sysfs
mount -t proc none /proc
mount -t sysfs none /sys

[ -e /dev/console ] || mknod /dev/console c 5 1
[ -e /dev/null ] || mknod /dev/null c 1 3

/sbin/mdev -s

setsid cttyhack sh

/bin/sh -i
```

Make `init` executable:

```bash
chmod +x init
```

The directory `initramfs` should be looks like this:

```
├── bin
├── init
├── linuxrc -> bin/busybox
├── sbin
└── usr
```

### Pack and Compress

```bash
find . | cpio -o -H newc | gzip -9c > ../initrd.cpio.gz
```

## Automatically Build with Buildroot

### Buildroot
```bash
wget https://buildroot.org/downloads/buildroot-2022.02.1.tar.gz
tar -zxf buildroot-2022.02.1.tar.gz
cd buildroot-2022.02.1
make menuconfig
```

menuconfig:
* Target Options
  * Target Architecture --> riscv
* Toolchain
  * Toolchain Type --> External Toolchain
  * Toolchain Path --> \<example: /opt/riscv\>
  * Toolchain prefix --> \<example: riscv64-unknown-linux-gnu\>
  * External toolchain gcc version --> \<example: 11.x\>
  * External toolchain kernel headers series --> \<example: 5.10.x\>
  * External toolchain C library --> \<example: glibc/egblic\>
* Filesystem images
  * select *cpio the root filesystem*

```bash
make
```

CPIO image is in directory `output/images/`.


# Linux Kernel

Download Linux Kernel, and choose 5.4.188 version.

```bash
wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.4.188.tar.xz
tar -xJf linux-5.4.188.tar.xz
cd linux-5.4.188
make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- defconfig
make ARCH=riscv menuconfig
```

menuconfig:

+ General Setup
  + Initramfs source file --> \<path-to-initramfs\>
+ Platform type
  + Exclude *Emit compressed instructions when building Linux* (remove c-extension)

```bash
make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- -j4
```

# QEMU

```bash
wget https://download.qemu.org/qemu-6.0.0.tar.xz
tar xvJf qemu-6.0.0.tar.xz
cd qemu-6.0.0
./configure --target-list=riscv64-softmmu,riscv32-softmmu
make
sudo make install
```

# OpenSBI

```bash
git clone https://github.com/riscv-software-src/opensbi.git
export CROSS_COMPILE=riscv64-unknown-linux-gnu-
make PLATFORM=generic FW_PAYLOAD_PATH=<linux_build_directory>/arch/riscv/boot/Image \
PLATFORM_RISCV_ISA=rv64imafd_zicsr_zifencei
```

# Test With QEMU

Linux without initramfs embedded:

```bash
qemu-system-riscv64 -M virt -m 256M -nographic -kernel <linux_build_directory>/arch/riscv/boot/Image \
-initrd <path-to-initramfs>
```

Linux with initramfs embedded:

```bash
qemu-system-riscv64 -M virt -m 256M -nographic -kernel <linux_build_directory>/arch/riscv/boot/Image
```

Linux with initramfs and opensbi embedded:

```bash
qemu-system-riscv64 -M virt -m 256M -nographic \
-bios <opensbi_build_directory>build/platform/generic/firmware/fw_payload.elf
```

# Reference
[Rootfs made easy with Buildroot](https://bootlin.com/pub/conferences/2013/kernel-recipes/rootfs-kernel-developer/rootfs-kernel-developer.pdf)

[构建基于Busybox的Initramfs](https://digwtx.com/initramfs.html)

