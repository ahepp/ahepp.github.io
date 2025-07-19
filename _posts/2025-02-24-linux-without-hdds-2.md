---
layout: post
title:  "Linux Without Hard Drives: Root on NFS and Beyond"
---

## Introduction

In my last post, I discussed making an RHEL 8 system without any direct attached storage.
At that time I had decided "network hard drive" was a better abstraction than "network file system".
However, thinking about it some more, it sounds interesting to ditch the block device abstraction.
I'm curious if we could even build a kernel without a block device driver.

The reader may be interested to know that I began this article a mere 5 days after [the last article in this series][lwhd1].
I got quite far in my attempt.
The [RHEL 7 Storage Administration Guide][rhel7sag] contains a lot of great information to this end.
However, there is no indication that RHEL 8 supports this configuration.
In fact, towards the end of the installation process, I ran into mysterious errors with the RHEL package manager, DNF.
It turns out that DNF uses a sqlite database backed by the filesystem, and sqlite notoriously does not run well on NFS.

I managed to create a bootable RHEL 8 system on NFS, but the package manager was pretty broken, and I never felt like the blog post was worth sharing.
But six months later, I'm feeling up for another shot with Debian.

## Motivation

I have a couple motivations for exploring root-on-nfs.

As I mentioned above, the idea of running a kernel without a block device driver sounds fun.

I'm also interested in the integrations this may unlock between PXE boot and NFS.
One drawback of iSCSI was that I had to manually rsync updates to /boot on the iSCSI "disk" over to the PXE server.
I could add some kind of hook with DNF to push updates to the server, but I like the idea of this process being transparent to the client.

On similar lines, it was always a pain to mount an ext4 filesystem on the FreeBSD system that serves all my storage.
Being able to use ZFS snapshots on the rootfs is a nice perk.

And finally, how many times have you had to go through that song and dance with a virtual machine where you expand the block device, carefully delete and recreate larger partitions (probably just giving up on your swap partition entirely), then expand your filesystem, just to grow your rootfs a bit and recover from an out-of-space condition?
With root-on-nfs this should be as simple as expanding a quota.

## Installing

First I'll need to obtain a Debian installer. 
Given the context it would be pretty pathetic to resort to a USB drive.
Digging around on Debian's website turns up a tarball named "netinst" tarball that sounds pretty exciting.

```
root@tlon:~ # fetch https://deb.debian.org/debian/dists/bookworm/main/installer-amd64/current/images/netboot/netboot.tar.gz
netboot.tar.gz                                          49 MB   42 MBps    02s
root@tlon:~ # sha256sum netboot.tar.gz 
3429fa77a3823da5bfb9235a956d8b89b181999f2e23a6a4194c4379630106de  netboot.tar.gz
root@tlon:~ # zfs create zpa/pxe/debian-installer 
root@tlon:~ # tar -C /zpa/pxe/debian-installer -xzf netboot.tar.gz 
root@tlon:~ # tree -L 3 /zpa/pxe/debian-installer
/zpa/pxe/debian-installer
├── debian-installer
│   └── amd64
│       ├── boot-screens
│       ├── bootnetx64.efi
│       ├── grub
│       ├── grubx64.efi
│       ├── initrd.gz
│       ├── linux
│       ├── pxelinux.0
│       └── pxelinux.cfg
├── ldlinux.c32 -> debian-installer/amd64/boot-screens/ldlinux.c32
├── pxelinux.0 -> debian-installer/amd64/pxelinux.0
├── pxelinux.cfg -> debian-installer/amd64/pxelinux.cfg
├── splash.png -> debian-installer/amd64/boot-screens//splash.png
└── version.info
root@tlon:~ # zfs snapshot zpa/pxe/debian-installer@20250224-bookworm
```

Looks promising.

I have dnsmasq configured to tell PXE clients to look for a file named "bootfile".

```
root@tlon:~ # tail -n 4 /usr/local/etc/dnsmasq.conf
enable-tftp
tftp-unique-root=mac
tftp-root=/zpa/pxe/
dhcp-boot=bootfile
```

In conjunction with `tftp-unique-root=mac`, this avoids the need to clutter up my dnsmasq configuration files.
I create a symlink named for the client MAC and point it to a directory containing boot files
Inside the directory, I create a symlink named "bootfile" and point it to the file I want the PXE firmware to load.

```
root@tlon:/zpa/pxe # ln -s debian-installer 5c-ed-8c-5b-bc-ee
root@tlon:/zpa/pxe # pushd debian-installer/
/zpa/pxe/debian-installer /zpa/pxe ~ 
root@tlon:/zpa/pxe/debian-installer # ln -s debian-installer/amd64/grubx64.efi bootfile
```

I made a couple more changes to ease installation via serial console

```
root@tlon:/zpa/pxe/debian-installer # diff \
? .zfs/snapshot/20250225-bootfile/debian-installer/amd64/grub/grub.cfg \
? debian-installer/amd64/grub/grub.cfg
34c35
<     linux    /debian-installer/amd65/linux vga=788 --- quiet
---
>     linux    /debian-installer/amd64/linux console=ttyS0,115200 vga=788 --- quiet
```

How amazing is it that we can diff files against that zfs snapshot we made earlier?
That's the kind of thing that excites me about root-on-nfs (with zfs backing!)

Now we can fire up the corporeal husk of our system and do a pretty typical debian install.






Let's create a new dataset on our storage server that will hold this machine's rootfs, and then export that dataset:

```
root@tlon:~ # zfs create zpa/nfs/orbis3
root@tlon:~ # vi /etc/exports 
root@tlon:~ # cat /etc/exports 
V4: /zpa/nfs/
/zpa/nfs/orbis3 -maproot=root
root@tlon:~ # service nfsd restart
Stopping nfsd.
Waiting for PIDS: 2174 2175.
Starting nfsd.
root@tlon:~ # service mountd restart
Stopping mountd.
Waiting for PIDS: 1430.
Starting mountd.
```

Now we'll mount our new rootfs on our root-on-iscsi system, and install the RHEL 8 base along with a couple packages we will need for root-on-nfs:

```
[admin@uqbar ~]$ sudo mount -t nfs4 -o vers=4.2 tlon.san:/orbis3 /mnt
[admin@uqbar ~]$ sudo dnf install @Base kernel dracut-network nfs-utils --installroot /mnt --releasever=8
```

I had to run this `dnf` command three times before it succeeded without error.
It complained about not being able to read an rpm database in /var/lib/rpm, presumably because /var didn't exist yet...
I don't know why this eventually did succeed, maybe there is some kind of race condition.
The `dnf` confirmation I eventually reached looked like:

```
Transaction Summary
================================================================================
Install  495 Packages

Total download size: 732 M
Installed size: 2.2 G
Is this ok [y/N]:
```


## Beyond Root on NFS

Let's talk briefly about some other ways we could have a system that doesn't write to any local storage, without involving either iscsi or nfs.
I'm changing the rules a bit here, because I'm going to be talking about systems that don't allow you to make any _persistent_ changes to the root filesystem.

You probably work with a system like this every time you boot linux: the initramfs.
Just like there's no rule against a dog playing basketball, there's no rule that your initramfs needs to mount some other block device.
You can write to your initramfs, but it's all in RAM so the changes will not persist across a reboot.

I've experimented a bit with systems that run out of a PXE booted initramfs.
I'm sure you could abuse dracut modules to build a functional initramfs from a binary distribution, but I am more familiar with source based distributions so that's the path I took.
Changing a couple settings in buildroot can spit out that rootfs in cpio form.
You can even compress it, which may be useful since grub seems to refuse to load an initramfs larger than about 450M (that goes by fast if all your software is in it).

Keep in mind that you could also 

[rhel7sag]: https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/7/html/storage_administration_guide/diskless-nfs-config
