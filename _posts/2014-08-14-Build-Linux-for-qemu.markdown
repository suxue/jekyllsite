---
layout: post
title:  "How to build a compact and bootable linux image for qemu/kvm"
date:   2014-08-14 15:02:31
categories: kernel
---

Prepare a qemu image
---------------------

	qemu-img create -f qed test.img 200M
	qemu-nbd --connect=/dev/nbd9 test.img

Then we make a bootable partition:

	fdisk /dev/nbd9

format this new partition:

	kpartx -a /dev/nbd9
	mkfs.ext2 /dev/mapper/nbd9p1

Mount this partition:

	mount -t ext2 -o loop /dev/mapper/nbd9p1 fs_root/


Prepare kernel image
---------------------

You may take file `qemu_x86_64_defconfig` from buildroot source as a reference
for your own kernel configuration.

Nowadays qemu/kvm supports Paravirtualized drivers -- name virtio, related options
are:

+ `CONFIG_VIRTIO_BLK`
+ `CONFIG_VIRTIO_CONSOLE`
+ `CONFIG_VIRTIO_PCI`

As usual, we compile the kernel

	make mrproper
	make menuconfig
	make

Resulting kernel image is at `arch/x86/boot/bzImage`


Build target filesystem using buildroot
--------------------------------------------

Contemporary Linux distributions are bloat, especially for virtual machines
with only few megabytes memory.

The [buildroot](http://buildroot.uclibc.org/) project ease this burden.

+ download the package
+ Similar to the kernel, `make menuconfig` and activate options you need
+ `make`, the script will download all source tarballs and compile them.

There are some configuration options in `.config` you may be interested in:

+ `BR2_TARGET_GENERIC_ROOT_PASSWD`, the root password
+ `BR2_TARGET_GENERIC_GETTY_PORT`, the serial port device name, because we
   will use virtconsole, so this option would be set to `hvc0`
+ `BR2_TARGET_ROOTFS_TAR`, set to `y` buildroot will generate a ready-to-boot
  filesystem image named `output/image/rootfs.tar`

After `make` completed successfully, you can untar the `rootfs.tar` to mounted
directory fs_root/

	tar xvf rootfs.tar -C /path/to/fs_root/

There are still some important things to note:

+ Change /etc/fstab, because we will use virtio block driver, the root device
  will be /dev/vda1
+ append one line to `/etc/inittab`, hence you will be able to login through
  the SDL window of qemu/kvm.

	tty1::respawn:/sbin/getty -L tty1 9600 vt100

Try to boot the new image
--------------------------

The command line will looks like

	qemu-system-x86_64 -cpu host --enable-kvm  -drive file=test.img,if=virtio -vga qxl -device virtio-serial -chardev socket,path=/tmp/foo,server,nowait,id=foo  -device virtconsole,chardev=foo,name=org.fedoraproject.console.foo -kernel bzImage --append 'root=/dev/vda1 console=tty1'

You can login through the gui window, or type this command in the host:

	socat /tmp/foo -

Make a self-contain image
--------------------------

So far so good, the image is bootable with a host kernel image, how about
make it bootable by itself?

First, copy the kernel image into bootable image,

	cp bzImage /path/to/fs_root/boot

Then, install grub:

	grub-install --modules=part_msdos --boot-directory=/path/to/fs_root/boot /dev/nbd9

Make sure `root=/dev/vda1` in `/path/to/fs_root/boot/grub/grub.cfg`.

Next step, unload all partitions in host machine to avoid corrupting the
guest filesystem:

	umount /path/to/root_fs
	kpartx -d /dev/nbd9
	nbd-client -d /dev/nbd9

Finally, boot this image

    export MAC_ADDR=00:16:3e:00:00:04
    qemu-system-x86_64   -drive file=test.img,if=virtio -net nic,model=virtio,macaddr=$MAC_ADDR -net tap,vhost=on,helper=/usr/libexec/qemu-bridge-helper

or, using local kernel

    qemu-system-x86_64  -net nic,model=virtio,macaddr=$MAC_ADDR -net tap,vhost=off,helper=/usr/libexec/qemu-bridge-helper -kernel bzImage -append "root=/dev/vda1 tty=tty1"  -drive file=linux.img,if=virtio

Example images
--------------

linux.img(7m), with only base system

    wget 'https://drive.google.com/uc?export=download&id=0B8Dry_1TaeLMMFd5QTNDOWtMSmM' -O - | xzcat  > linux.img

linux_net.img(25m), with tcp/ip stack, wget, openssh ..., configs are in /root

    wget 'https://drive.google.com/uc?export=download&id=0B8Dry_1TaeLMbGNFUjFEYW51Tkk' -O - | xzcat > linux_net.img


Update: User Network
---------------------

Compared to the bridge/taps approach, usermode networking is much easier..

The configuration is two-folds, from the guest's side, you should append 2
lines to the init file to activate virtual ethernet and assign a usable ip address
to it. This can be achieved by editing /etc/init.d/S40network

Beneath `/sbin/ifup -a`, append

    ip link set eth0 up
    ip addr add 10.0.2.1/24 dev eth0
    ip ro add default via 10.0.2.2 dev eth0 # This is the host's ip

From the host's side, you will boot your image by
`-net nic,model=virtio  -net user,hostfwd=::60022-10.0.2.1:22`, then ssh into the guest:

    ssh root@127.0.0.1 -P 60022

Here is [an example image and some utility scripts](https://drive.google.com/file/d/0B8Dry_1TaeLMdFYzdEZFanlRV00/edit?usp=sharing)
