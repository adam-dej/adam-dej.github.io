---
layout: post
title: OLinuXIno Arch with btrfs root on external HDD
categories: misc
---

This post explains how to install Arch Linux on OLinuXIno LIME 2, with root on
external btrfs HDD.

Here comes a long one... If you are not interested in **why** I'm doing what I'm
doing, only in **how**, read only `Installing the system` section.

For a year I've been using rPI as my home server. It has performed adequately,
but for various reasons I've decided to do a HW upgrade and to choose a little
bit different approach for setup.

Price, power consumption, size and noise were (and still are) the key factors
that need to be considered when choosing a new hardware.

My choise was
[A20-OLinuXIno-LIME2](https://www.olimex.com/Products/OLinuXino/A20/A20-OLinuXIno-LIME2/open-source-hardware)
. It is very cheap, has dual-core 1GHz armv7 CPU, 1GB RAM and onboard SATA.
As a neat bonus, it is completely open hardware platform. Onboard SATA was the
key feature that convinced me to use this board. On my previous rPI setup root
has also been on the external HDD, connected using SATA <-> USB converter. This
was a huge bottlenect, since Ethernet on rPI is also connected to the same USB
hub.

The previous server setup was a single Arch system on which all my services were
running. This was OK for most of them, but Gitlab... that thing broke **every**
time there was a Ruby update. With no spare time on my hands I've simply stopped
updating the server.

This time I'm going to address this issue. The base system will be still Arch
Linux. But it will only be a minimal, bare system, host to several
[LXC containers](https://wiki.archlinux.org/index.php/Linux_Containers). In
those containerized isolated environments services themselves will run. This
approach remedies the problem with problematic software dependent on a
particular version of many things. And if I were to use btrfs, root of each
Container can be a btrfs subvolume. This has some neat implications:
Container cloning with Copy-On-Write support, snapshotting before messing with
the container (so that changes can be reverted rapidly if something goes
wrong)...

This is the reason I'm installing Arch Linux with btrfs root on external HDD.
This post covers the installation of base system.

Arch installation
=================

Choosing the right approach
---------------------------

There are several possible ways to achieve this. Arch on OLinuXIno uses the
u-boot bootloader. It can load kernel image from serveral filesystems, but btrfs
is not one of them. That automatically means a separate boot partition will be
required.

How about root? Arch Linux ARM is installed from prepared .tar.gz file
containing the whole root filesystem. But, attempting to extract it to the
btrfs partition causes errors like

~~~
./usr/foo/bar bsdtar: Failed to set file flags
~~~

OK, so extract it to an ext4 partition and then convert it to the btrfs file
system? That would work, but [btrfs wiki](https://btrfs.wiki.kernel.org/index.php/Conversion_from_Ext3)
warns about this: **Warning: As of 4.0 kernels this feature is not much used or
well tested anymore, and there have been some reports that the conversion
doesn't work reliably. Feel free to try it out, but make sure you have backups**
. Uh, that's not something I would rely on when it comes to a root of a server.

I've chosen  to install Arch Linux ARM from scratch, in the same way Arch Linux
is installed on desktops. That means we will prepare an installation medium on a
SD card, boot it and install arch to HDD from it, therefore avoid all those
problems.

Installing the system
---------------------

This process is heavily based upon these instructions, combined and changed
where necessary:

[http://archlinuxarm.org/platforms/armv7/allwinner/a20-olinuxino-lime2](http://archlinuxarm.org/platforms/armv7/allwinner/a20-olinuxino-lime2)
[https://wiki.archlinux.org/index.php/Installation_guide](https://wiki.archlinux.org/index.php/Installation_guide)

### Preparing the installation medium

Zero out the beginning of the SD card:

~~~
# dd if=/dev/zero of=/dev/sdX bs=1M count=8
~~~

Now we will partition the card in an interesting way. I suppose you know how to
use `fdisk`, if not, read the manual and some tutorials. We will create two
partitions, one for the whole installation system, and one for the new `/boot`
partition. Start `fdisk` and create primary partition number 1 **starting on
sector 104448**. Accept the default end. Now create primary partition number 2
starting on sector 2048 and with end on 104447. This will create a 50M
partition.

Why have we done this? U-boot looks at the **first** partition and looks for the
(among other things) `/boot/zImage`. For now, we want our installation partition
to be the first, so that our installation medium will boot. Later, when the
system is installed, we will swap those partitions and our new system will boot.

After `fdisk`-ing the card, it should look like this:

~~~
Device         Boot  Start      End  Sectors  Size Id Type
/dev/sdX1           104448 15353855 15249408  7.3G 83 Linux
/dev/sdX2             2048   104447   102400   50M 83 Linux
~~~

Format and mount the installation partition:

~~~
# mkfs.ext4 /dev/sdX1
# mount /dev/sdX1 /mnt/tmp
~~~

Download and extract Arch Linux ARM image to the installation partition:

~~~
# wget http://archlinuxarm.org/os/ArchLinuxARM-armv7-latest.tar.gz
# bsdtar -xpf ArchLinuxARM-armv7-latest.tar.gz -C /mnt/tmp
# sync
~~~

Download and install bootloader to the card:

~~~
# wget http://archlinuxarm.org/os/sunxi/boot/a20-olinuxino-lime2/u-boot-sunxi-with-spl.bin
# dd if=u-boot-sunxi-with-spl.bin of=/dev/sdX bs=1024 seek=8
# wget http://archlinuxarm.org/os/sunxi/boot/a20-olinuxino-lime2/boot.scr -O /mnt/tmp/boot/boot.scr
# umount /mnt/tmp
# sync
~~~

Now insert the card to your OLinuXIno (with connected HDD you will install Arch
on) and boot. When the the boot finishes log in to the installation system (root
pw is `root` (change it!)). Install those packages:

~~~
# pacman -Syu arch-install-scripts btrfs-progs
~~~

OK, now we are ready to use this image to install Arch Linux ARM from scratch.

### Installing (for real now, I promise...)

The routine stuff. Partition the HDD using `fdisk`:

~~~
Device     Boot   Start       End   Sectors   Size Id Type
/dev/sda1          2048   4196351   4194304     2G 82 Linux swap / Solaris
/dev/sda2       4196352 976773167 972576816 463.8G 83 Linux
~~~

Create boot, root and swap filesystems:

~~~
# mkfs.ext2 /dev/mmcblk0p2
# mkfs.btrfs /dev/sda2
# mkswap /dev/sda1
~~~

Mount it:

~~~
# mount /dev/sda2 /mnt
# mkdir /mnt/boot
# mount /dev/mmcblk0p2 /mnt/boot
# swapon /dev/sda1
~~~

Install base system packages:

~~~
# pacstrap /mnt base btrfs-progs
~~~

Note: The above command won't install kernel itself (and I was wondering why
it would't boot...). We will install it later.

Now for the basic configuration. Create `fstab`:

~~~
# genfstab -p /mnt >> /mnt/etc/fstab
~~~

Now, our `/boot` partition is `mmcblk0p2`. We will change it later to
`mmcblk0p1`, so let's tell `fstab` that. Change line containing `mmcblk0p2` to:

~~~
/dev/mmcblk0p1          /boot       ext2        rw,relatime 0 2
~~~

Chroot to the newly created system for even more configuration :) :

~~~
# arch-chroot /mnt
~~~

Set the hostname and timezone:

~~~
# echo 'your_hostname' > /etc/hostname
# ln -sf /usr/share/zoneinfo/<Zone>/<Subzone> /etc/localtime
~~~

Uncomment your locale in `/etc/locale.gen` and generate locales, set your
language:

~~~
# locale-gen
# echo LANG=en_US.UTF-8 > /etc/locale.conf
~~~

Install the kernel:

~~~
# pacman -S linux-armv7
~~~

Set the root password using `passwd`. This should be enough, but don't reboot
your system yet!

I'm connecting to the OLinuXIno using ssh, therefore network must be up and sshd
must be running in the new installation.

~~~
# pacman -S openssh
# systemctl enable sshd
~~~

My device has static IP, so I've written `systemd-networkd` config file:

~~~
[Match]
Name=eth0

[Network]
DNS=your_preferred_dns (maybe 8.8.8.8)

[Address]
Address=desired_ip/netmask

[Route]
Gateway=gateway
~~~

and followed
[Arch systemd-networkd](https://wiki.archlinux.org/index.php/Systemd-networkd)
wiki page, all this is explained there.

~~~
# systemctl enable systemd-networkd
# systemctl enable systemd-resolved
# rm /etc/resolv.conf
# ln -s /run/systemd/resolve/resolv.conf /etc/resolv.conf
~~~

Also, according to the wiki: replace `dns` with `resolve` in
`/etc/nsswitch.conf`:

~~~
hosts: files resolve myhostname
~~~

### Making the system bootable

While still in the Arch chroot install a bootloader:

~~~
# pacman -S uboot-a20-olinuxino-lime2
~~~

It asks you whether you want to flash a new bootloader. Let it do so.

Also install

~~~
# pacman -S uboot-tools
~~~

we will need them for the final bootloader configuration.

By default u-boot installed by the above package looks for the kernel and
everything else in `/boot`. Since we have a separate boot partition to u-boot it
appears as if the kernel was in `/`. So let's modify `boot.txt` to make it look
for it there. Add those lines right after

~~~
setenv bootargs console=${console} root=/dev/sda2 rw rootwait
~~~

It's basically a copy of existing script, but with `/boot` removed.

~~~
if load ${devtype} ${devnum}:${bootpart} ${kernel_addr_r} /zImage; then
  if load ${devtype} ${devnum}:${bootpart} ${fdt_addr_r} /dtbs/${fdtfile}; then
    if load ${devtype} ${devnum}:${bootpart} ${ramdisk_addr_r} /initramfs-linux.img; then
      bootz ${kernel_addr_r} ${ramdisk_addr_r}:${filesize} ${fdt_addr_r};
    else
      bootz ${kernel_addr_r} - ${fdt_addr_r};
    fi;
  fi;
fi

if load ${devtype} ${devnum}:${bootpart} 0x48000000 /uImage; then
  if load ${devtype} ${devnum}:${bootpart} 0x43000000 /script.bin; then
    setenv bootm_boot_mode sec;
    bootm 0x48000000;
  fi;
fi
~~~

Next, run `mkscr` to make a boot script from this file.

~~~
# cd /boot
# ./mkscr
~~~

Get out of the chroot now (`Ctrl + D`) and unmount the new system

~~~
# umount -R /mnt
~~~

Last thing there is to do is to swap those two partitions on the SD card. Use
`fdisk`, print existing partitions, note where they start and end, then erase
them. Create new two partitions, but the first will start and end where the
second did before.

Before:

~~~
Device         Boot  Start      End  Sectors  Size Id Type
/dev/mmcblk0p1      104448 15353855 15249408  7.3G 83 Linux
/dev/mmcblk0p2        2048   104447   102400   50M 83 Linux
~~~

After:

~~~
Device         Boot  Start      End  Sectors  Size Id Type
/dev/mmcblk0p1        2048   104447   102400   50M 83 Linux
/dev/mmcblk0p2      104448 15353855 15249408  7.3G 83 Linux
~~~

Write the new partition table to the card. Note: `fdisk` will complain

~~~
Calling ioctl() to re-read partition table.
Re-reading the partition table failed.: Device or resource busy
~~~

That's OK, since one partition is still mounted as a root of our current system.

That's it. `sync` and go ahead and reboot. Enjoy your new system :).

### Last-minute thoughts

A message appeared:

~~~
[  408.655073] systemd-journald[128]: Creating journal file /var/log/journal/bd00c0bbc8e94c7197c9d945a65fe627/user-1000.journal on a btrfs file system, and copy-on-write is enabled. This is likely to slow down journal access substantially, please consider turning off the copy-on-write file attribute on the journal directory, using chattr +C.
~~~

So let's disable CoW for for `/var/log/journal` and all existing files (this
must be done from other system, files mustn't be used during this process):

~~~
# mv /var/log/journal /var/log/journal_old
# mkdir /var/log/journal
# chattr +C /var/log/journal
# cp -a /var/log/journal_old/* /var/log/journal
# rm -rf /var/log/journal_old
~~~

Optionally: create yourself a user, upload your public key and disable `ssh` password and
root login.

And since we have btrfs, let's make a snapshot of our fresh new install:

~~~
# btrfs subvolume snapshot / root_snapshot_2015-07-10_postinstall
~~~

In the next series of posts I will show you how I've installed various services
into Linux Containers.
