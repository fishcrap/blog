+++ 
draft = false
date = 2022-04-04T18:49:34+08:00
title = "Run Linux on RISC-V QEMU"
description = ""
slug = ""
authors = ["Xujie Shen"]
tags = []
categories = []
externalLink = ""
series = []
+++

# Build RISC-V toolchain
Since "qemu" submodule is not related to the toolchain's compilation, you can just delete it.  


```bash
git clone https://github.com/riscv-collab/riscv-gnu-toolchain.git
cd riscv-gnu-toolchain
git rm qemu
git submodule update --init --recursive
./configure --prefix=/opt/riscv
make linux
```


If the submodules of the repository is hard to download, you could clone the submodules separately:


```bash
git clone https://github.com/riscv-collab/riscv-binutils-gdb.git
git clone https://github.com/riscv-collab/riscv-dejagnu.git
git clone https://github.com/riscv-collab/riscv-gcc.git
git clone git://sourceware.org/git/glibc.git
git clone git://sourceware.org/git/newlib-cygwin.git
```


And then move the submodules into the toolchain:


```bash
cd riscv-gnu-toolchain
rm riscv-binutils
rm rm riscv-binutils
rm riscv-dejagnu
rm riscv-gcc
rm riscv-gdb
rm riscv-glibc
rm riscv-newlib
mv <submodules> riscv-gnu-toolchain 
```

# Prepare Linux Kernel
Download Linux Kernel, and choose 5.4.188 version.


```bash
wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.4.188.tar.xz
tar -xJf linux-5.4.188.tar.xz
cd linux-5.4.188
make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- defconfig
make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- -j4
```


The default isa is IMAC. If you do not need C-extension, a simple method is to alter .config before compiling the kernel.



```bash
make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- defconfig
<edit ".config": CONFIG_RISCV_ISA_C=y --> CONFIG_RISCV_ISA_C=n>
make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- -j4
```


If you want to disable C-extension more elegantly, use `make ARCH=riscv menuconfig` instead of brutely editing the file `.config`.

# QEMU

```bash
wget https://download.qemu.org/qemu-6.0.0.tar.xz
tar xvJf qemu-6.0.0.tar.xz
cd qemu-6.0.0
./configure --target-list=riscv64-softmmu,riscv32-softmmu
make
sudo make install
```


# Build Root File System

You can use `busybox` to make a rootfs, which is necessary to start the kernel.


## Build Busybox


```bash
git clone git://busybox.net/busybox.git
git checkout 1_34_stable
CROSS_COMPILE=riscv64-unknown-linux-gnu- make menuconfig
<Setting --> Build Options --> Build static binary (no shared libs) --> save>
CROSS_COMPILE=riscv64-unknown-linux-gnu- make -j $(nproc)
CROSS_COMPILE=riscv64-unknown-linux-gnu- make install
```


You could find a `_install` directory which looks like:


```
_install
├── bin
├── linuxrc -> bin/busybox
├── sbin
└── usr
```

## Make Rootfs

### Add Directories

```bash
dd if=/dev/zero of=rootfs.img bs=1M count=64
mkfs.ext4 rootfs.img

mkdir mnt
sudo mount -o loop rootfs.img mnt
cd mnt
sudo cp -r ../busybox/_install/* .
sudo mkdir proc sys dev etc etc/init.d
```

### Init rcS
```bash
cd etc/init.d/
sudo touch rcS
sudo vi rcS
```

Edit `rcS` as following:

```bash
#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
/sbin/mdev -s
```

Add execute permission for `rsS`.

```bash
sudo chmod +x rcS
```

Unmount.
```bash
sudo umount mnt
```


# OpenSBI

```bash
git clone https://github.com/riscv-software-src/opensbi.git
export CROSS_COMPILE=riscv64-unknown-linux-gnu-
make PLATFORM=generic FW_PAYLOAD_PATH=<linux_build_directory>/arch/riscv/boot/Image
```


# Run Linux on QEMU

```bash
qemu-system-riscv64 -M virt -m 256M -nographic \
	-bios <opensbi_build_directory>/build/platform/generic/firmware/fw_jump.bin \
	-kernel <linux_build_directory>/arch/riscv/boot/Image \
	-drive file=<linux_rootfs_directory>/rootfs.img,format=raw,id=hd0 \
	-device virtio-blk-device,drive=hd0 \
	-append "root=/dev/vda rw console=ttyS0"
```



# Reference
[在 QEMU 上运行 RISC-V 64 位版本的 Linux](https://zhuanlan.zhihu.com/p/258394849)

[riscv64 qemu上进行Linux环境搭建与开发记录](http://mp.weixin.qq.com/s?__biz=MzI4MDQ3MzU1MQ==&mid=2247484931&idx=1&sn=c3e2a344d30d34869093ce3dc8d15ed9&chksm=ebb6bca3dcc135b5c9a32c0b7eee242844646411eff5d824a128d11790122fa9f66b78001594&scene=21#wechat_redirect)

<https://github.com/UCanLinux/riscv64-sample>

<https://github.com/riscv-software-src/opensbi/blob/master/docs/platform/qemu_virt.md>