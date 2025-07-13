---
title: "学习Linux-01启动"
date: 2025-07-13T21:46:45+08:00
draft: false
toc: false
tags: 
- "linux"
---

# 学习Linux-01启动

> ref: https://jyywiki.cn/OS/2024/lect18.md
>
> src: wget -r -np -nH --cut-dirs=2 -R "index.html*" "https://jyywiki.cn/os-demos/virtualization/linux/"

## 1. 折腾的原因总结

1. linux 启动过程中报错不是那么准确，例如：执行的二进制（init）缺少了 .so，会报错 exec init failed: no such file.
2. 例如 virtio_blk 之类的驱动，需要自己在 initramfs 中自己加载。

## 2. 使用的环境

1. kali 2024.4
   1. kernel 6.11.2-amd64
   2. initrd.img 6.11.2-amd64 (被我替换了静态编译的 busybox)
    ~~~
        不能使用 cpio 等命令解压，看起来是多个文件直接拼接起来的一个文件（使用 binwalk 能够看到），直接使用 cpio 会只能解出来里面的第一个文件；
        如果需要提取完整的内容，需要使用 unmkinitramfs
   ~~~
   
2. qemu-system-x86_64 10.0.2
3. [busybox 1.37.0](https://busybox.net/downloads/busybox-1.37.0.tar.bz2)

## 3. Makefile 和 init 脚本

* Makefile (unmkinitramfs 的部分没有写在里面，手动执行的)
~~~makefile
# Requires statically linked busybox in $PATH

INIT := /init
IMG := build/disk.img
MOUNT := $(shell mktemp -d)
K := $(shell uname -r)

all: initramfs fsroot

initramfs:
# Copy kernel and busybox from the host system
	@mkdir -p build/initramfs/bin
	sudo bash -c "cp /boot/vmlinuz-$K build/vmlinuz && chmod 666 build/vmlinuz"
	cp init build/initramfs/
	chmod 777 build/initramfs/init

# Pack build/initramfs as gzipped cpio archives
	cd build/initramfs && find . | cpio -H newc -o | gzip > ../initramfs.cpio.gz

fsroot:
	mkdir -p fsroot/lib/modules
	chmod 777 fsroot/etc/init.d/rcS
	cp $$(find /lib/modules/$K/ -name e1000.ko.xz) fsroot/lib/modules/

	dd if=/dev/zero of=$(IMG) bs=1M count=64
	mkfs.ext4 -F $(IMG)
	sudo mount $(IMG) $(MOUNT)
	cd fsroot && sudo cp -r * $(MOUNT)
	sudo umount $(MOUNT)
	chmod 777 $(IMG)

run:
# Run QEMU with the installed kernel and generated initramfs 
	qemu-system-x86_64 \
	  -nographic \
	  -drive file=$(IMG),format=raw,index=0,media=disk,if=virtio \
	  -netdev user,id=net0,hostfwd=tcp:127.0.0.1:8080-:8080 -device e1000,netdev=net0 \
	  -kernel build/vmlinuz \
	  -initrd build/initramfs.cpio.gz \
	  -machine accel=kvm:tcg \
	  -m 2048 \
	  -append "console=ttyS0 quiet root=/dev/vda rdinit=$(INIT) debug" \

clean:
	rm -rf build fsroot/lib/modules

.PHONY: initramfs run clean fsroot
~~~

* init
~~~shell
#!/bin/busybox sh

# At this point, we only have:
#   /bin/busybox - the binary
#   /dev/console - the console device

/bin/busybox ls
/bin/busybox echo start_init

BB=/bin/busybox

# Delete this file
# (Yes, we can do this on UNIX! The file is "removed"
# after the last reference, the fd, is gone.)
$BB rm /init

# $BB find / -type f
$BB echo "Unlimited power!!"
# $BB poweroff -f
# ----------------------------------------------------
 
# "Create" command-line tools by making symbolic links
#   try: busybox --list
# for cmd in $($BB --list); do
#     $BB ln -s $BB /bin/$cmd
# done

# Mount procfs and sysfs
mkdir -p /proc && mount -t proc  none /proc
mkdir -p /sys  && mount -t sysfs none /sys

modprobe virtio_blk

# Create devices
mknod /dev/random  c 1 8
mknod /dev/urandom c 1 9
mknod /dev/null    c 1 3
mknod /dev/tty     c 4 1
mknod /dev/vda     b 254 0

echo -e "\033[31mInit OK; launch a shell (initramfs).\033[0m"

#busybox sh

# Display a countdown

echo -e "\n\n"
# echo -e "\033[31mSwitch root in...\033[0m"
# for sec in $(seq 3 -1 1); do
#     echo $sec; sleep 1
# done

# Switch root to /newroot (a real file system)
N=/newroot

mkdir -p $N
mount -t ext4 /dev/vda $N
chmod 777 $N

mkdir -p $N/bin
chmod 777 $N/bin

cp $BB $N/bin/
cp $BB /init
cp $BB /$N/init

ls / -l
ls $N -l
ls $N/bin -l

busybox sh

exec switch_root /newroot/ /init
~~~

* fsroot/init
~~~shell
#!/bin/busybox sh

# Now, "/" is the virtual disk--for real
# systems, we expect installed binaries.
BB=/bin/busybox

$BB echo "init"

for cmd in $($BB --list); do
    $BB ln -s $BB /bin/$cmd
done

mkdir -p /proc && mount -t proc  none /proc
mkdir -p /sys  && mount -t sysfs none /sys
mkdir -p /tmp  && mount -t tmpfs none /tmp

mkdir -p /dev
mknod /dev/random  c 1 8
mknod /dev/urandom c 1 9
mknod /dev/null    c 1 3
mknod /dev/tty     c 4 1
mknod /dev/vda     b 254 0

# Configure network
insmod /lib/modules/e1000.ko.xz
ip link set lo up
ip link set eth0 up
ip addr add 10.0.2.15 dev eth0
ip route add 10.0.2.0/24 dev eth0

# Prepare the terminal
echo -e "\033[31mGoodbye, QEMU Console!\033[0m"
echo -e "\033[H\033[2J" >/dev/tty
echo -e "JYY's minimal Linux" >/dev/tty

# Switch to terminal
setsid /bin/sh </dev/tty >/dev/tty 2>&1

sync
poweroff -f
~~~