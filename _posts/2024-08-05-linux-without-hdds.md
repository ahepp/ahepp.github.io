---
layout: post
title:  "Linux Without Hard Drives"
---

## Introduction

I want to make an RHEL 8 system where all storage is abstracted over the network.

We could use network filesystems like NFS or SMB for this, but that's not the abstraction I want to use here.
I want a "network hard drive", not a "network filesystem", so a more appropriate choice would be iSCSI.

What is iSCSI?
Well, the first question you should ask is "what is SCSI?"
SCSI is a protocol computers use to communicate with hard drives.
iSCSI tunnels those commands over a network.
We'll have an iSCSI server on our network, providing a hard drive that can be accessed over TCP/IP.

RHEL supports running off an iSCSI "disk", so that handles a lot of the hard work for us.
The remaining question is, how do we boot a computer without any disks in it?
Well, modern firmware implementations have a feature called Preboot eXecution Environment (PXE).

## More than you wanted to know about PXE

PXE is a feature of your computer's firmware (its BIOS or UEFI).
You're probably familiar with adjusting boot priorities in your firmware.
If the firmware has PXE enabled, one option in that menu will be to PXE boot from a network interface.

The PXE boot process is well supported, but a bit convoluted.
When this option is selected, the firmware will request a DHCP lease on the selected interface.
The DHCP server provides a filename in its response.
The firmware will then request that file from the DHCP server via TFTP.
You may be interested to know that certain DHCP options can be used to direct the client to a different address for the TFTP server, but we aren't going to use them here.

Let's assume we tell the DHCP server to provide the firmware the path to a GRUB bootloader.
The firmware will execute the bootloader, which will then attempt to load a config file from the TFTP server.
That config file will provide further instructions to the bootloader.
In particular, it needs to provide a path for a kernel and initrd for the bootloader to load from the TFTP server.
It will also need to provide any kernel command line arguments.

### Even more obscure thoughts on PXE

I've simplified the discussion of PXE here quite a bit.
Your bootloader will need to have been built with support for whatever network protocols you want it to use.
The paths it will search for config files also depend on how it was built.
I was able to make things work by copying over the bootloader RHEL installed on my disk, but some fancy setups may need to compile their own GRUB.

Your firmware and/or bootloader may also support loading files via HTTPS.
I think there are a lot of cool opportunities here.
It opens the door up for integration with object storage (think S3).
HTTPS is also considerably more "firewall friendly" and secure, meaning it could, perhaps, traverse the public internet?

An additional area to explore further is using Unified Kernel Images or similar technology to cut down on the Rube Goldberg machine I've described here.

## Serving iSCSI

That whole PXE boot thing sounds really complicated.
Let's not worry about it yet!
I told you RHEL supports root on iSCSI out of the box, so let's just set that up to start.

First, let's set up the iSCSI "disk".
I'm doing this on my FreeBSD box that has a large storage pool and a variety of network services enabled.
We'll create a sparse ZFS volume, and configure the iSCSI daemon to serve it:

```
root@tlon:~ # zfs create -s -V 1T zpa/iscsi/uqbar-root
root@tlon:~ # zfs list -r -t all zpa/iscsi
NAME                               USED  AVAIL  REFER  MOUNTPOINT
zpa/iscsi                         1.05T  14.9T    96K  none
zpa/iscsi/uqbar-root                56K  14.9T    56K  -
root@tlon:~ # vi /etc/ctl.conf
root@tlon:~ # cat /etc/ctl.conf
debug 5
isns-timeout 60
portal-group pg0 {
    discovery-auth-group no-authentication
    listen 172.16.0.1
}
target iqn.2022-07.san.iscsi:uqbar-root {
    auth-group no-authentication
    portal-group pg0
    lun 0 {
        path /dev/zvol/zpa/iscsi/uqbar-root
    }
}
root@tlon:~ # service ctld restart
Stopping ctld.
Waiting for PIDS: 2492.
Starting ctld.
ctld: /etc/ctl.conf is world-readable
ctld: opening pidfile /var/run/ctld.pid
ctld: adding lun "iqn.2022-07.san.iscsi:uqbar-root,lun,0"
ctld: adding port "pg0-iqn.2022-07.san.iscsi:uqbar-root"
ctld: not listening on portal-group "default", not assigned to any target
ctld: listening on 172.16.0.1, portal-group "pg0"
ctld: daemonizing
```

## Installing the system

Use whatever hypervisor you want to create a virtual machine with a small virtual hard drive to hold `/boot` and `/boot/efi`.
4GB should be plenty for this.
You'll need at least one network adapter.
You'll probably want to add a virtual CD drive with your installation media.

Boot up the VM and start installing RHEL.
Enable your network adapter(s).
Go to the disk partitioning menu.
This is another are where I'm not going to provide very detailed instructions.
There should be a pretty obvious button to scan for iSCSI targets.

Put `/boot` and `/boot/efi` on your virtual hard drive, and put `/` on the iSCSI device.
Then complete the rest of the installer steps, and run the install.
It really is that easy.

After installation finishes, reboot the system and verify that it's working like any normal linux system.
This is pretty cool!
`/boot` isn't used after the system is booted, so we're already running without any disks!

## Removing the boot drive

We've picked the low hanging fruit, now we're sailing into uncharted territory.

First off, where we're going there is no `/boot` or `/boot/efi`.
Let's comment those lines out of `/etc/fstab`, then regenerate the initramfs with `sudo dracut -f`.
This will replace our old initramfs with one that won't attempt to mount our soon-to-be-detached hard drive.

Now let's further prepare to delete our boot drive by copying the files off our boot drive and onto the root device:

```
[admin@uqbar ~]$ mount | grep boot
/dev/sda2 on /boot type xfs (rw,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,noquota)
/dev/sda1 on /boot/efi type vfat (rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=ascii,shortname=mixed,errors=remount-ro)
[admin@uqbar ~]$ sudo tar -C /boot -cf boot.tar .
[admin@uqbar ~]$ sudo umount -R /boot
[admin@uqbar ~]$ mount | grep boot
[admin@uqbar ~]$ sudo tar -C /boot -xf boot.tar
[admin@uqbar ~]$ sudo tree /boot/
/boot/
├── config-4.18.0-553.8.1.el8_10.x86_64
├── efi
│   └── EFI
│       ├── BOOT
│       │   ├── BOOTX64.EFI
│       │   └── fbx64.efi
│       └── redhat
│           ├── BOOTX64.CSV
│           ├── fonts
│           ├── grub.cfg
│           ├── grubenv
│           ├── grubx64.efi
│           ├── mmx64.efi
│           ├── shimx64.efi
│           └── shimx64-redhat.efi
├── grub2
│   └── grubenv -> ../efi/EFI/redhat/grubenv
├── initramfs-0-rescue-c3c74a7e37de4ad1b274263255e8831c.img
├── initramfs-4.18.0-553.8.1.el8_10.x86_64.img
├── initramfs-4.18.0-553.8.1.el8_10.x86_64kdump.img
├── loader
│   └── entries
│       ├── c3c74a7e37de4ad1b274263255e8831c-0-rescue.conf
│       └── c3c74a7e37de4ad1b274263255e8831c-4.18.0-553.8.1.el8_10.x86_64.conf
├── symvers-4.18.0-553.8.1.el8_10.x86_64.gz -> /lib/modules/4.18.0-553.8.1.el8_10.x86_64/symvers.gz
├── System.map-4.18.0-553.8.1.el8_10.x86_64
├── vmlinuz-0-rescue-c3c74a7e37de4ad1b274263255e8831c
└── vmlinuz-4.18.0-553.8.1.el8_10.x86_64
```

My hope is that doing this will let us update boot files with minimal disruption, even though we will need to copy those files to our TFTP server manually.

Let's try copying those files to our TFTP server now.
We'll want

 * `/boot/vmlinuz-4.18.0-553.8.1.el8_10.x86_64`
 * `/boot/initramfs-4.18.0-553.8.1.el8_10.x86_64.img`
 * `/boot/efi/EFI/redhat/grubx64.efi`

 We'll also want to note our kernel command line, so we can copy the important parts to our bootloader configuration:

```
[admin@uqbar ~]$ cat /proc/cmdline
BOOT_IMAGE=(hd0,gpt2)/vmlinuz-4.18.0-553.8.1.el8_10.x86_64 root=UUID=5fabb8c5-7abe-4b54-9df5-0576c3007aea ro crashkernel=auto netroot=iscsi:@172.16.0.1::3260::iqn.2022-07.san.iscsi:rootoniscsi rd.iscsi.initiator=iqn.1994-05.com.redhat:fe3e80701156 rhgb quiet ip=ens224:dhcp
```

Once the files are in your TFTP directory, create a file there named `grub.cfg`, pointing the bootloader to the kernel and initramfs.
You'll also want to provide those command line arguments we just took note of:

```
root@tlon:/zpa/tftp # tree
.
├── grub.cfg
├── grubx64.efi
├── initramfs-4.18.0-553.8.1.el8_10.x86_64.img
└── vmlinuz-4.18.0-553.8.1.el8_10.x86_64

1 directory, 4 files

root@tlon:/zpa/tftp # cat grub.cfg
linux /vmlinuz-4.18.0-553.8.1.el8_10.x86_64 root=UUID=5fabb8c5-7abe-4b54-9df5-0576c3007aea ro netroot=iscsi:@172.16.0.1::3260::iqn.2022-07.san.iscsi:rootoniscsi rd.iscsi.initiator=iqn.1994-05.com.redhat:fe3e80701156 ip=ens224:dhcp
initrd /initramfs-4.18.0-553.8.1.el8_10.x86_64.img
boot
root@tlon:/zpa/tftp #
```

Set your DHCP server to serve the bootloader:

```
root@tlon:/zpa/tftp # tail -4 /usr/local/etc/dnsmasq.conf

enable-tftp
tftp-root=/zpa/tftp
dhcp-boot=/grubx64.efi
root@tlon:/zpa/tftp # service dnsmasq reload
Stopping dnsmasq.
Waiting for PIDS: 59926.
Starting dnsmasq.
```

Now you can power off your VM and detach the virtual hard drive.
Boot the VM and select the PXE boot option on the appropriate interface.
You should see some interesting activity in the DHCP server logs:

```
root@tlon:/zpa/tftp # tail -0 -f /var/log/daemon.log | grep dnsmasq | sed 's/^.*dnsmasq[^ ]*//'
4153516254 available DHCP range: 172.16.0.100 -- 172.16.0.254
4153516254 vendor class: PXEClient:Arch:00007:UNDI:003000
4153516254 DHCPDISCOVER(mlxen0) 00:0c:29:8c:d4:14
4153516254 tags: mlxen0
4153516254 DHCPOFFER(mlxen0) 172.16.0.130 00:0c:29:8c:d4:14
4153516254 requested options: 1:netmask, 2:time-offset, 3:router, 4, 5,
4153516254 requested options: 6:dns-server, 12:hostname, 13:boot-file-size,
4153516254 requested options: 15:domain-name, 17:root-path, 18:extension-path,
4153516254 requested options: 22:max-datagram-reassembly, 23:default-ttl,
4153516254 requested options: 28:broadcast, 40:nis-domain, 41:nis-server,
4153516254 requested options: 42:ntp-server, 43:vendor-encap, 50:requested-address,
4153516254 requested options: 51:lease-time, 54:server-identifier, 58:T1,
4153516254 requested options: 59:T2, 60:vendor-class, 66:tftp-server, 67:bootfile-name,
4153516254 requested options: 97:client-machine-id, 128, 129, 130, 131,
4153516254 requested options: 132, 133, 134, 135
4153516254 next server: 172.16.0.1
4153516254 broadcast response
4153516254 sent size:  1 option: 53 message-type  2
4153516254 sent size:  4 option: 54 server-identifier  172.16.0.1
4153516254 sent size:  4 option: 51 lease-time  12h
4153516254 sent size: 13 option: 67 bootfile-name  /grubx64.efi
4153516254 sent size:  4 option: 58 T1  6h
4153516254 sent size:  4 option: 59 T2  10h30m
4153516254 sent size:  4 option:  1 netmask  255.255.255.0
4153516254 sent size:  4 option: 28 broadcast  172.16.0.255
4153516254 sent size:  4 option:  6 dns-server  172.16.0.1
4153516254 available DHCP range: 172.16.0.100 -- 172.16.0.254
4153516254 vendor class: PXEClient:Arch:00007:UNDI:003000
4153516254 DHCPREQUEST(mlxen0) 172.16.0.130 00:0c:29:8c:d4:14
4153516254 tags: mlxen0
4153516254 DHCPACK(mlxen0) 172.16.0.130 00:0c:29:8c:d4:14
4153516254 requested options: 1:netmask, 2:time-offset, 3:router, 4, 5,
4153516254 requested options: 6:dns-server, 12:hostname, 13:boot-file-size,
4153516254 requested options: 15:domain-name, 17:root-path, 18:extension-path,
4153516254 requested options: 22:max-datagram-reassembly, 23:default-ttl,
4153516254 requested options: 28:broadcast, 40:nis-domain, 41:nis-server,
4153516254 requested options: 42:ntp-server, 43:vendor-encap, 50:requested-address,
4153516254 requested options: 51:lease-time, 54:server-identifier, 58:T1,
4153516254 requested options: 59:T2, 60:vendor-class, 66:tftp-server, 67:bootfile-name,
4153516254 requested options: 97:client-machine-id, 128, 129, 130, 131,
4153516254 requested options: 132, 133, 134, 135
4153516254 next server: 172.16.0.1
4153516254 broadcast response
4153516254 sent size:  1 option: 53 message-type  5
4153516254 sent size:  4 option: 54 server-identifier  172.16.0.1
4153516254 sent size:  4 option: 51 lease-time  12h
4153516254 sent size: 13 option: 67 bootfile-name  /grubx64.efi
4153516254 sent size:  4 option: 58 T1  6h
4153516254 sent size:  4 option: 59 T2  10h30m
4153516254 sent size:  4 option:  1 netmask  255.255.255.0
4153516254 sent size:  4 option: 28 broadcast  172.16.0.255
4153516254 sent size:  4 option:  6 dns-server  172.16.0.1
error 8 User aborted the transfer received from 172.16.0.130
sent /zpa/tftp/grubx64.efi to 172.16.0.130
file /zpa/tftp/grub.cfg-01-00-0c-29-8c-d4-14 not found for 172.16.0.130
file /zpa/tftp/grub.cfg-AC100082 not found for 172.16.0.130
file /zpa/tftp/grub.cfg-AC10008 not found for 172.16.0.130
file /zpa/tftp/grub.cfg-AC1000 not found for 172.16.0.130
file /zpa/tftp/grub.cfg-AC100 not found for 172.16.0.130
file /zpa/tftp/grub.cfg-AC10 not found for 172.16.0.130
file /zpa/tftp/grub.cfg-AC1 not found for 172.16.0.130
file /zpa/tftp/grub.cfg-AC not found for 172.16.0.130
file /zpa/tftp/grub.cfg-A not found for 172.16.0.130
sent /zpa/tftp/grub.cfg to 172.16.0.130
file /zpa/tftp/EFI/redhat/x86_64-efi/command.lst not found for 172.16.0.130
file /zpa/tftp/EFI/redhat/x86_64-efi/fs.lst not found for 172.16.0.130
file /zpa/tftp/EFI/redhat/x86_64-efi/crypto.lst not found for 172.16.0.130
file /zpa/tftp/EFI/redhat/x86_64-efi/terminal.lst not found for 172.16.0.130
sent /zpa/tftp/grub.cfg to 172.16.0.130
sent /zpa/tftp/vmlinuz-4.18.0-553.8.1.el8_10.x86_64 to 172.16.0.130
sent /zpa/tftp/initramfs-4.18.0-553.8.1.el8_10.x86_64.img to 172.16.0.130
```

And there you have it! You're running a RHEL 8 system with nothing but network storage!

An exercise for the reader is to make this system boot on a different machine.
Hint: make sure your initramfs has the network driver for whatever computer you want to boot from (or it won't be able to mount the iSCSI disk).
Dracut makes this pretty straightforward.
