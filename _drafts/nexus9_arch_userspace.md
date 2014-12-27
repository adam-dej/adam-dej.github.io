---
layout: post
title: Nexus 9&#58 Arch linux userspace in chroot
categories: Nexus9-hacking
---

This post is about getting Arch linux userspace up and running on an Android tablet (or phone). This was done and tested on the Nexus 9, 
but it should (with some or none) modification work on other Android phones and tablets.

Disclaimer: Although the risk in this procedure is minimal (rooting is much riskier), this guide is provided "as is", without any waranty. 
I'm not responsible for any damage to you or your handset.

## Why would you want to do that?
And why not? Just think of it: All your favorite userspace utilities running on your tablet. And pacman to install what you need. Plain 
awesome.
(e. g. this post was written on the Nexus with arch userspace, compiled with jekyll, uploaded with git)

### How does this work?
In contrast to booting Linux kernel on the Nexus, this procedure utilizes existing Android system and it's kernel. The original android 
will stay untouched. You can launch a terminal emulator app from within Android (I recommend JuiceSSH) and chroot to a directory where you 
have arch installed. From your home directory you will have access to the contents of your SD card. This way terminal with Arch is just an 
app you can switch to and switch back from. Lately an usable X server application for android was released, so we can use it as X server 
for Arch, and the system will have graphical interface.

### What will this get me?
A fully working Arch linux installation, even with X server so that graphical applications can run! You can switch to and from Arch as you 
would do with any other Android application, as X server and Terminal Emulator which you use to interact with Arch are typical Android 
applications. This guide does not cover getting the sound to work. Also, systemd is unusable in chroot, therefore you will have to launch 
daemons manually.

## Installation

First, backup. Just to be sure :)

### Rooting Nexus 9
The first step is rooting your device. There are many guides on the internet how to do that, follow them. I've done this on Android 5.0.1 
(Build LRX22C) by using Chainfire's image. The image is intended for LRX21L (Android 5.0), but worked fine. The process basically 
consists of unlocking your bootloader and booting a special image which will mount your system and do the job for you.

After you have rooted your device, please install Busybox, we will need it.

### Extracting Arch image

When you see some commands here, the convention is as follows:

~~~
(android) $ echo 'Execute this as normal user on Android'
(android) # echo 'Execute this as root on Android'
(arch) $ echo 'Execute this as normal user in arch chroot'
(arch) # echo 'Execute this as root in arch chroot'
~~~

Now let's extract Arch linux image to the emulated SD card (or the real one, if you have one). However, we cannot do this directly. Most 
Android phones and tablets have FAT-formatted SD-cards, but for Arch to work we need unix-compatible filesystem. Nexus 9 is using f2fs 
(Flash-Friendly Filesystem), to which in theory Arch can be installed. However, this filesystem is for apparent reasons mounted with 
nosuid, noexec and other restrictive options, therefore our installation wouldn't work properly. To get around this we will create an 
image file, loopback-mount it and install arch there. So:

~~~
(android) # mkdir arch

# Note: alarm stands for ArchLinux ARM
(android) # dd if=/dev/zero of=alarm.img bs=1024 count=3145728 # Create 3GB empty file

(android) # mkfs.ext2 -F alarm.img

(android) # mount -t ext2 -o loop alarm.img arch/
~~~

Now we will download and extract generic armv7 ArchLinux ARM image (if you are doing this on other device, match this to your processor 
architecture).

~~~
(android) # wget http://os.archlinuxarm.org/os/ArchLinuxARM-armv7-latest.tar.gz
(android) # cd arch
(android) # tar -xvzf ../ArchLinuxARM-armv7-latest.tar.gz .
~~~

Now the arch is installed. Let's mount and configure it.

### Chroot-ing to Arch

To chroot to our new installation we must first populate /dev, /sys and /proc directories.

~~~
(android) # mount -o bind /dev/ arch/dev/
(android) # mount -t devpts devpts arch/dev/pts
(android) # mount -t proc proc /proc
(android) # mount -t sysfs sysfs /sys
(android) # chroot arch /bin/su -
~~~

OK, so we are up and running. There are still more things to do, however. First, you may notice messages like

~~~
ERROR: ld.so: object 'libsigchain.so' from LD_PRELOAD cannot be preloaded (cannot open shared object file): ignored.
~~~

This is caused by LD_PRELOAD variable being set and containing 'libsigchain.so' which does not exist in our Arch installation. Just run 
`unset LD_PRELOAD`.

### Getting network up and running, permissions for SD card

Now let's get network and SD-card permissions. Because of `ANDROID_CONFIG_PARANOID_NETWORK` kernel configuration parameter, in order to 
access the network user must belong to groups with `gid`s 3003, 3004 and 3005. Similairly, in order to be able to read and write to the
SD card we must be in 1015 and 1028.

~~~
(arch) # groupadd -g 3003 aid_inet
(arch) # groupadd -g 3004 aid_net_raw
(arch) # groupadd -g 3005 aid_inet_admin
(arch) # groupadd -g 1015 sdcard_rw
(arch) # groupadd -g 1028 sdcard_r
~~~

And add root user and your user (which we will create later) to those groups

~~~
(arch) # gpasswd -a root root aid_inet
...
~~~

The last thing to do to make the network work is to create `resolv.conf` file. First, remove existing symlink, which is broken anyway, and 
from chroot run

~~~
(arch) # echo "nameserver 8.8.8.8" > /etc/resolv.conf
(arch) # chmod 644 /etc/resolv.conf
~~~

### Upgrade and cleanup

Now when the network is up you can upgrade your Arch. Then check what packages are installed, and remove those you don't need (who needs 
kernel anyway? :P)

~~~
(arch) # pacman -Syyu
~~~

### Adding a user

It is inadvisable to use the system as root all the time, therefore let's add a new user under which you'll be using Arch.

~~~
(arch) # useradd -m /home/username -s /bin/bash username
(arch) # groupadd -a aid_inet username
...
~~~

If you wish you can install sudo and add this user to the sudoers file.

Don't forget to add this user to all groups we have created earlier. Now let's mount Android's SD card so that you can access and use it 
from within the Arch chroot

~~~
(arch) $ mkdir Android
(android) # mount -o bind /storage/emulated/0 /storage/emulated/0/arch/home/username/Android
~~~

### X server

We can install X server as Android application. I recommend using [XServer 
XSDL](https://play.google.com/store/apps/details?id=x.org.server) application, I've 
sucessfully used it in this example. After installation just install some graphical applications, window manager or your favorite desktop 
environment the usual way on Arch:

~~~
(arch) $ sudo pacman -S lxde
~~~

Now launch X server on the android, and set `DESKTOP` environment variable according to the instructions displayed in the server app 
(except for the address, you can use 127.0.0.1):

~~~
(arch) $ export DESKTOP=127.0.0.1:0
~~~

And now you can launch whatever you wish the usual way:

~~~
(arch) $ startlxde
~~~

## Conclusion

Some last-minute ideas:
Even as root you may be getting Permission denied errors when attempting to create sockets. Run `# su -` to create a clean environment for 
root, that should solve the problem.

When you have created your user, the last command of chrooting process can be `chroot arch/ /bin/su - username`

By now you should have a fully working Arch linux on your handset. Don't forget to cleanup all the mess (kill all processes from 
chroot, umount `/sys`, `/dev`, `/proc`...) after you are finished. Enjoy :)
