---
layout: post
title:  "Linux Without Hard Drives: Root on NFS and Beyond"
---

## Introduction

In my last post, I discussed making an RHEL 8 system without any direct attached storage.
I said I didn't want to use nfs for this because I thought a "network hard drive" was a better abstraction than "network filesystem".
However, after thinking about the issue some more, I'm not convinced I was right.
At the end of the day, you don't need a block device to run an operating system, you do need a filesystem.
In any case, I think it would be interesting to explore another approach to running a diskless system.

One issue with our root-on-iscsi setup is that updates to /boot won't make it to our tftp directory.
We could probably set up some kind of cron job for the storage server peek into the iscsi backing store and sync the RHEL system's /boot with the storage server's tftp directory.
This sounds pretty convoluted, and could get really hairy if we somehow synced while the RHEL system was updating (unlikely to happen, but still a dirty thought).
We could also set up some kind of hooks on the RHEL system to push new files to the tftp directory when updates happen.
This idea seems a bit more promising, although I would really prefer to not change anything on the RHEL system.

I think running root-on-nfs could simplify this a bit.
Since NFS exports a filesystem on the storage server, we could simply change the directory served by the tftp server to `$NFS_ROOT/boot` and serve files straight out of the RHEL system's /boot.

We can also get some nice benefits by not having to worry about the abstraction of a block device.
Allocating more space to the RHEL rootfs would be as simple as adjusting an NFS quota.
However we manage to grow our iscsi volume (probably zfs sending it to a larger zvol), we will need to grow the underlying filesystem as well.

## Installing

The interactive RHEL 8 installer doesn't seem to have any options for root-on-nfs.
That means we have a few options.

One option would be to sync files from our existing RHEL system onto the NFS share.
In broad strokes, we could copy the files from our root-on-iscsi system, or we could mount our nfs share and use the dnf package manager to bootstrap files onto that.
I'm going to try the latter option.

There is a lot of relevant reading in the [RHEL 7 Storage Administration Guide][rhel7sag].
I can't find an equivalent document for RHEL 8, but the information seems like it is still relevant.

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
