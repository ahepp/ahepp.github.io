---
layout: post
title:  "Exploring Buildroot with the Raspberry Pi"
---

## Introduction

In [a previous post][last-post] I promised a future post detailing how I created my custom Raspberry Pi Zero W based PID controller.
This post will discuss the basics of creating a custom OS for the Raspberry Pi using [Buildroot](https://buildroot.org).

## Getting started
Starting from a Debian Bookworm machine, we'll need a few packages:

    $ apt-get update
    $ apt-get install -y --no-install-recommends \
        bc \
        build-essential \
        ca-certificates \
        cpio \
        file \
        libncurses-dev \
        rsync \
        unzip \
        wget

We'll create a new git project, and add Buildroot as a submodule.

    $ mkdir os && pushd os && git init
    $ git submodule add https://gitlab.com/buildroot.org/buildroot/
    $ git commit -m "add buildroot"

Many Raspberry Pi devices are supported out of the box.

    $ ls -1 buildroot/configs/raspberrypi*
    buildroot/configs/raspberrypi0_defconfig
    buildroot/configs/raspberrypi0w_defconfig
    buildroot/configs/raspberrypi2_defconfig
    buildroot/configs/raspberrypi3_64_defconfig
    buildroot/configs/raspberrypi3_defconfig
    buildroot/configs/raspberrypi3_qt5we_defconfig
    buildroot/configs/raspberrypi4_64_defconfig
    buildroot/configs/raspberrypi4_defconfig
    buildroot/configs/raspberrypi_defconfig
    buildroot/configs/raspberrypicm4io_64_defconfig
    buildroot/configs/raspberrypicm4io_defconfig
    buildroot/configs/raspberrypizero2w_defconfig

We could build a bootable rootfs for the Raspberry Pi Zero W right now.

    $ make -C buildroot raspberrypi0w_defconfig
    $ make -C buildroot

The config even has [post-image][br-post-image] scripts that create a block image we can `dd` straight onto an SD card and boot.
Another such script enables the UART serial console, so we have some way of interacting with the device.

## Customizing

Since you're reading a post about the Zero W, you're probably interested in getting this thing connected to wifi.
That will require a little customization of our Buildroot project.

The [Buildroot manual](https://buildroot.org/downloads/manual/manual.html) discusses two methods of customizing a buildroot project.
You can simply fork Buildroot, and add commits on top of it, or you can use a br2-external tree.
Coming from a Yocto background, I have a slight preference towards maintaining my changes in an external tree.
It's worth noting that while this bears a superficial resemblance to a Yocto layer, an external tree is not as complex (or powerful), and there really isn't a *huge* difference between Buildroot's two supported approaches.
Using the br2-external-tree can help you review the totality of your modifications without cooking up a `git` incantation.

As an aside, Buildroot's manual is a great resource.
If you're used to Yocto's endless reams of understructured, duplicative, yet somehow incomplete documentation, don't assume Buildroot is the same.

We'll [add a br2-external-tree][os-tree-commit] and copy our device's defconfig into our tree to modify down the road.

    $ mkdir -p tuxpresso/config
    $ cp buildroot/config/raspberrypi0w_defconfig tuxpresso/config/tuxpresso0w_defconfig

Now we build with

    $ make -C buildroot BR2_EXTERNAL=$(pwd)/tuxpresso tuxpresso0w_defconfig
    $ make -C buildroot

We'll [enable the wifi drivers and WPA supplicant][os-wifi-commit].
We can now run `modprobe brcmfmac` to enable the wifi driver.
If we want this done automatically, we can [enable `mdev`][os-mdev-commit], a stripped down version of the device event manager `udev`.

Now let's configure our wifi settings.
We'll want to [add a rootfs overlay][os-overlay-commit], and I've chosen to copy the WPA supplicant configuration from the FAT formatted `/boot` partition of the SD card every time the system boots.
I probably should have mounted the filesystem as read only, since we would really be unhappy if the system somehow decided to corrupt `/boot`.
Since our interface configuration doesn't have any secrets in it, I've simply [added it to the overlay][os-interfaces-commit].

I haven't checked recent versions of Buildroot, but I had to [disable wifi power saving][os-powersaving-commit] in order to get [ssh login][os-ssh-commit] working reliably.
`haveged` is installed to speed up boot by providing faster random number generator initialization.
I also [copied my ssh public keys onto the system][os-key-commit] in a manner similar to the WPA supplicant configuration.

## What's next?

I see a lot of fun directions to go from here.
We haven't added any of our secret sauce to this custom operating system yet!
Another interesting topic would be stripping this kernel down to the bones, and seeing how fast we can get this thing to boot.

[last-post]: /2023/03/08/coffee-linux-2.html
[br-post-image]: https://gitlab.com/buildroot.org/buildroot/-/blob/2024.02.1/board/raspberrypi/post-image.sh?ref_type=tags
[os-tree-commit]: https://github.com/tuxpresso/os/commit/db993c1dc1052dd3dcf800249b0c161a24071917
[os-defconfig-commit]: https://github.com/tuxpresso/os/commit/30eeae44fc04c57bdb1c72846a35bcaab3f1ea5d
[os-wifi-commit]: https://github.com/tuxpresso/os/commit/5c7c14ed3525ee38270b9affb44122d2f6847e48
[os-mdev-commit]: https://github.com/tuxpresso/os/commit/372158316621ce2c1b47233c0e9bea618a1a54f8
[os-overlay-commit]: https://github.com/tuxpresso/os/commit/f554a27fbdbe440c58d769881bfb06c783575bb5
[os-interfaces-commit]: https://github.com/tuxpresso/os/commit/738fc88b56962d3399ec67707f96a35295595682
[os-powersaving-commit]: https://github.com/tuxpresso/os/commit/be8f9beb8dd5e59c7aedfb7489745e532f66fcdd
[os-ssh-commit]: https://github.com/tuxpresso/os/commit/70cea68a79e560d18ea114dad1289e24e4f7108f
[os-key-commit]: https://github.com/tuxpresso/os/commit/d306bf94a9c800957e74ca11d37133adaf137c47
