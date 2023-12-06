# Installation

## Alpine Linux disk image

TBD

## Alpine Linux "from scratch"

This is a quick guide on how to install the Alpine base system on your RK1, "by hand."
I am writing this mainly to help familiarize users with tinkering on Linux distros for the RK1 at a somewhat basic level,
which should help build up intuition about how the boot process works, and hopefully inspires many of you to port your
own favorite distros to the RK1. :)

**Before beginning, back up any data on the RK1 that you care about. This process will reformat the RK1 fully.**

### BMC access

Note that all commands here will be done in the BMC shell itself. SSH into your Turing Pi 2 to begin.
**Make sure you are on BMC firmware 2.0.6+; versions <=2.0.5 have a minor eMMC access bug, which may interfere with partitioning.**

First, put the RK1 into "msd" mode. This makes it accessible from the BMC as an ordinary block device:
```
# tpi advanced -n 1234 msd
```

Now, run `lsmod` to make sure the `rockusb` module is loaded (`modprobe rockusb`, and [report it](https://github.com/turing-machines/BMC-Firmware/issues), if not).

Confirm that `/dev/sda` exists:
```
# ls /dev/sda
/dev/sda
```

### Partitioning

We're going to reformat the RK1. Let's begin with a fresh partition table:
```
# sgdisk -Z /dev/sda
```

We need two partitions; one for the (main) U-Boot code, and one for the rootfs.
You can create additional partitions and/or rename these as you like, but it is very important that
the U-Boot partition starts at LBA `16384` (and is large enough to hold the `u-boot.its` file).
If you create other partitions, it's good practice to ensure that none of them begin before LBA 2048.

```
# sgdisk -n 1:16384:+2M -c 1:uboot /dev/sda
# sgdisk -n 2:: -c 2:alpine -A 2:set:2 /dev/sda
```

Let's format that "alpine" partition as ext4:
```
# mkfs.ext4 /dev/sda2
```

And get it mounted up so we can access it:
```
# mkdir /tmp/rootfs
# mount -t ext4 /dev/sda2 /tmp/rootfs
```

### Installing the base Alpine system

Alpine's userland is available as a "rootfs.tar" download. We are **not** going to use this, because we need a few extra packages.
Instead, we'll use a statically-linked build of `apk` -- Alpine's own package manager -- to pull in the packages we need.
This will run **on the BMC**, but target our new Alpine rootfs.

```
# mkdir /tmp/apk
# curl https://dl-cdn.alpinelinux.org/alpine/v3.18/main/armv7/apk-tools-static-2.14.0-r2.apk | gunzip | tar -xC /tmp/apk
```

Now we can ask it to download the entire Alpine base system for us. You should *probably* replace `latest-stable` with a specific version:
```
# export REPO_URL=https://dl-cdn.alpinelinux.org/alpine/latest-stable
# /tmp/apk/sbin/apk.static -U --allow-untrusted --initdb --no-scripts --arch aarch64 \
    -X ${REPO_URL}/main -p /tmp/rootfs \
    add alpine-base
# cat > /tmp/rootfs/etc/apk/repositories <<EOF
${REPO_URL}/main
${REPO_URL}/community
EOF
```

### Enable boot services

Since we installed the base system with `--no-scripts`, the packages' post-installation scripts did not run,
so some of the packages need additional setup. One of these is OpenRC, the init system. Let's enable several
services needed on boot:
```
# for svc in devfs dmesg hwdrivers mdev; do ln -s /etc/init.d/$svc /tmp/rootfs/etc/runlevels/sysinit/; done
# for svc in bootmisc hostname modules networking seedrng sysctl syslog; do ln -s /etc/init.d/$svc /tmp/rootfs/etc/runlevels/boot/; done
# for svc in crond ntpd; do ln -s /etc/init.d/$svc /tmp/rootfs/etc/runlevels/default/; done
# for svc in killprocs mount-ro savecache; do ln -s /etc/init.d/$svc /tmp/rootfs/etc/runlevels/shutdown/; done
```

And none of this will work in the first place if we don't have
`/sbin/init`, which we currently lack because Busybox didn't get to install its symlinks
(again, due to `--no-scripts`). Here's a workaround:
```
# cat > /tmp/rootfs/sbin/init <<EOF
#!/bin/sh

/bin/busybox mount -t proc proc proc
/bin/busybox mount -o remount,rw /
/bin/busybox rm /sbin/init
/bin/busybox --install -s && /bin/bbsuid --install
exec /sbin/init
EOF

# chmod +x /tmp/rootfs/sbin/init
```

Also, you're definitely going to want to enable serial port logins:
```
# vi /tmp/rootfs/etc/inittab
...
# Put a getty on the serial port
ttyS0::respawn:/sbin/getty -L ttyS0 115200 vt100
# ^ uncomment this line
...
```

### Installing core RK1 system packages

We'll add the `alpine-rk1` repository (the one containing this README), so we can grab packages from that:
```
# echo 'https://alpine-rk1.cfs.works/packages/main' >> /tmp/rootfs/etc/apk/repositories
# wget -P /tmp/rootfs/etc/apk/keys/ http://alpine-rk1.cfs.works/packages/cfsworks@gmail.com-6549341f.rsa.pub
# sha512sum /tmp/rootfs/etc/apk/keys/cfsworks*.pub
c6f826eeb590eb8315ce2d9b382e2b95827ec4a597d3dd0735f93313fe4de0a509f565433fe8dbca90330c26d9c7e5a34583380fa702ff3225b50d7c22aebada  /tmp/rootfs/etc/apk/keys/cfsworks@gmail.com-6549341f.rsa.pub
```

With that added, we'll have our `apk.static` binary fetch the (RK1-supporting) kernel and bootloader:
```
# /tmp/apk/sbin/apk.static -p /tmp/rootfs -U add --no-scripts \
    linux-firmware-none linux-turing u-boot-turing
```

Now we install the two components of the U-Boot bootloader. This is the important part! It makes the RK1 bootable!
**If you changed the partitioning, make sure you write `u-boot.itb` to the proper destination partition!!!**

```
# dd if=/tmp/rootfs/usr/share/u-boot-turing/turing-rk1-rk3588/idbloader.img of=/dev/sda bs=512 seek=64
# dd if=/tmp/rootfs/usr/share/u-boot-turing/turing-rk1-rk3588/u-boot.itb of=/dev/sda1
```

### Preparing the kernel for booting

Our system is *almost* startable; we're just missing the initramfs file and bootloader script.
For those who don't know, the initramfs file is (primarily) intended to address this circular problem:
- The base Linux kernel typically doesn't include built-in support for various filesystems and devices.
- These are loaded dynamically as kernel modules (.ko files) from the filesystem on the system storage...
- ...which the kernel doesn't have built-in support to access!

The initramfs file gets around this: it is a package of essential drivers/modules and init programs to
teach the nascent kernel enough to load the rest of the system, typically provided to it by the bootloader
(which knows how to load the initramfs file the same way it knows how to load the kernel file).
Alpine contains a handy script (`mkinitfs`) to generate this file for us; we just have to run it. Easy enough!
Only, one hiccup: this script wants to run programs for 64-bit ARM. We're on the BMC, which is 32-bit ARM. ...argh!
To get around this, we'll have our trusty `apk.static` fetch the programs needed into the BMC's memory.
From there, we use a few bind-mounts to override some ARM64 binaries with ARM32 equivalents:
```
# /tmp/apk/sbin/apk.static -U --allow-untrusted --initdb \
    -X ${REPO_URL}/main -p /tmp/apk \
    add mkinitfs
# mkdir /tmp/apk/lib/modules
# mount --rbind /tmp/rootfs/lib/modules /tmp/apk/lib/modules
# for path in /lib /usr/lib /bin /sbin /usr/bin; do mount --rbind /tmp/apk${path} /tmp/rootfs${path}; done
```

Finally, we can create our initramfs:
```
# chroot /tmp/rootfs mkinitfs $(ls /tmp/rootfs/lib/modules)
==> initramfs: creating /boot/initramfs-turing
# ls /tmp/rootfs/boot/initramfs-turing
/tmp/rootfs/boot/initramfs-turing
```

Let's quickly unmount all of those bind-mounts before we forget to clean up the mess:
```
# umount /tmp/apk/lib/modules
# for path in /lib/modules /lib /usr/lib /bin /sbin /usr/bin; do umount /tmp/rootfs${path}; done
```

...phew. That was the hard part.

### Writing the bootloader script

In the past section, I mentioned we were missing 2 files. That's now down to one: the bootloader script.
It tells U-Boot how to load the kernel (+ the files it needs) and run it. To create this file, we need a
program called `mkimage`. Let's just grab that onto the BMC real quick:
```
# /tmp/apk/sbin/apk.static --allow-untrusted \
    -X ${REPO_URL}/main -p /tmp/apk \
    add u-boot-tools
```

And now we can write and pack our bootloader script:
```
# cat >/tmp/apk/boot.cmd <<'EOF'
### After editing, repack this script with:
### mkimage -c none -A arm64 -T script -d boot.cmd boot.scr

kernel_path=/boot/vmlinuz-turing
ramdisk_path=/boot/initramfs-turing
fdt_path=/boot/dtbs-turing/rockchip/rk3588-turing-rk1.dtb

root_device=/dev/mmcblk0p2

env set bootargs initrd=${ramdisk_path} rootfstype=ext4 root=${root_device} clk_ignore_unused
load ${devtype} ${devnum}:${distro_bootpart} ${kernel_addr_r} ${kernel_path}
load ${devtype} ${devnum}:${distro_bootpart} ${fdt_addr_r} ${fdt_path}
bootefi ${kernel_addr_r} ${fdt_addr_r}
EOF

# chroot /tmp/apk mkimage -c none -A arm64 -T script -d boot.cmd boot.scr
# mv /tmp/apk/boot.* /tmp/rootfs/boot
```

### Cleanup and reboot!

```
# umount /tmp/rootfs
# rmdir /tmp/rootfs
# rm -r /tmp/apk
# tpi power -n 1234 reset
```

Congratulations on your new Alpine installation! Go watch the UART output to see it boot.
You may log in as root. By default, no password is set.
You will want to edit `/etc/network/interfaces` to bring the network up.
