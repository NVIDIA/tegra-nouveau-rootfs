Tegra Nouveau Installer Scripts
===============================
These scripts aim at providing an simple way to enable the open-source graphics stack (Nouveau/Mesa) on Jetson TK1/TX1. It does so by automating the process of cross-compiling the necessary software and adapting a new or already-existing Arch Linux root filesystem to run Nouveau/Mesa as an alternative to the closed-source graphics stack.

Following the instructions of this file will perform the following:
- Download an Arch Linux ARM image
- Update the image and install required packages on it
- Compile the Linux kernel and Nouveau kernel driver
- Compile a few DRM applications to play with (kmscube, weston, X)

Linux and Nouveau need to be compiled because there are still out-of-tree patches that we need. DRM applications currently need a few specific changes to work on Tegra systems (this is going to be fixed upstream), and thus also need to be compiled.

Host System Preparation
-----------------------
You will need a bunch of tools to download and cross-compile the various projects required for Nouveau on Tegra. Depending on your distribution, you will need to install:

- repo
- git
- proot
- autotools
- uboot-tools
- basic 32-bit libraries

Under Ubuntu 14.04, the following command will get you all set:

    $ sudo apt-get install git build-essential wget proot phablet-tools autoconf automake libtool libc6-i386 lib32stdc++6 lib32z1 pkg-config libwayland-dev bison flex bc u-boot-tools glib-2.0 realpath libffi-dev

Under Archlinux (2015-10-09), run the following commands:

    $ yaourt -S aur/repo # Or install the packages by downloading them from AUR yourself
    $ yaourt -S aur/proot
    $ sudo pacman -Syu base-devel wget git crypto++ libffi uboot-tools wayland bc python2-mako xorg-util-macros intltool

Note on Root Requirement
------------------------
Root access (using sudo) is required to perform some operations:

- Extracting the root filesystem with proper permissions
- Changing the permissions of the whole root filesystem to the current user (for update using proot) and back
- Installing a few SUID files (like weston-launch)

Make sure that you can run sudo, and be ready to enter the root password if required.

U-Boot
------
The first prerequisite is that you must use an up-to-date U-Boot as bootloader. You Jetson board comes with another bootloader flashed ; make sure to follow the instructions on https://github.com/NVIDIA/tegra-uboot-flasher-scripts/blob/master/README-developer.txt to easily flash U-Boot and be safe.

**Warning:** running an outdated U-Boot will cause the kernel to silently crash when loading the GPU driver!

Syncing
-------
All the required projects are synced using Google's `repo` tool:

    mkdir tegra-nouveau-rootfs
    cd tegra-nouveau-rootfs
    repo init -u https://github.com/NVIDIA/tegra-nouveau-rootfs.git -m tegra-nouveau.xml

Sync all the sources:

    repo sync -j4 -c

Once all the sources are downloaded, set the TOP environment variable:

    export TOP=$PWD

By default an arm32 (for Jetson TK1) image is built. If you want a 64-bit image, set the ARCH environment variable:

    export ARCH=aarch64

then download the cross-compilation toolchain that we will use:

    ./scripts/download-gcc

Preparing the Target Filesystem
-------------------------------
All the scripts expect to find your filesystem under `out/target/arm/ArchLinuxArm`. If you wish to use an already-existing rootfs, simply create a link from `out/target/arm/ArchLinuxArm` to the root of your existing filesystem.

If you prefer to operate on a L4T (Ubuntu-based) installation, define the DISTRO environment variable to "L4T" before doing anything else. Warning, this setup is unsupported and will likely fail in the middle!

Regardless of your choice is distro, run the following script to download the root filesystem:

    ./scripts/download-rootfs

It will download the latest Arch Linux ARM base image, and extract both under `out/target/arm/ArchLinuxArm`. You will need the ability to run `sudo` in order to preserve the permissions of the target filesystem.

Now that the target filesystem can be accessed under the expected location, we will need to make sure it contains all the libraries requires to let us cross-compile the graphics stack against it:

    ./scripts/prepare-rootfs

This script downloads a static qemu-arm binary and uses it under a chroot to run `pacman` under the target filesystem and install all the required libraries. It also requires `sudo` to run.

Compiling Kernel Components
---------------------------
We should now be able to cross-compile the Linux kernel:

    ./scripts/build-linux

This script will install the kernel under /boot/zImage-upstream of the target FS and link /boot/zImage to it, after having renamed any existing kernel binary to /boot/zImage-l4t. Finally, it will add a U-boot script that will boot the upstream kernel in place of the L4T one.

The Nouveau kernel modules can be build and installed similarly:

    ./scripts/build-nouveau

They will end in `/lib/modules/KERNEL_VERSION/extra` on the target FS.

Finally install the required GPU firmware:

    ./scripts/install-firmware

The firmware will be installed in `/lib/firmware/nvidia` on the target FS.

Compiling User-space Components
-------------------------------
The essential user-space components can be compiled as follows:

    ./scripts/build-pthread-stubs
    ./scripts/build-drm
<!---
    ./scripts/build-libinput
    ./scripts/build-wayland
    ./scripts/build-mesa
-->
Mesa also needs to be built:

    ./scripts/build-mesa

Then you can choose to add kmscube (useful for quickly testing that the graphics stack is working) and Weston (to enable a graphical UI):

    ./scripts/build-kmscube
    ./scripts/build-weston

If you are in for more serious business, why not also install X:

    ./scripts/build-xserver

This will build and install a modified X server (again to support the specific Tegra use-case) with libinput support.

The binaries and libraries will all be installed under `/opt/nouveau` by default. The `prepare-rootfs` script ran previously added the necessary environment variables to `/etc/profile.d/nouveau.sh` to make them available in the PATH.

Note that the `build-weston` script requires `sudo` in order to set the SUID bit to the `weston-launch` script.

Installing to Boot Device
-------------------------
At this stage your root filesystem is a standard Arch Linux distro, with some custom-build components in `/opt/nouveau`.

In order to install it to a boot device, copy the contents of `out/target/arm/ArchLinuxArm` to your device (which could be a SD card or internal eMMC).

To copy the root filesystem to a mounted (and empty) ext4-formatted SD card:

    sudo rsync -aAXv $TOP/out/target/arm/ArchLinuxArm/* /path/to/mount/point/ 


If you prefer to sync to the internal eMMC, do the following:

1. turn your board on, and on the serial console press a key to enter the U-boot menu.
2. connect a USB cable to the Jetson's micro-USB port.
3. type `ums 0 mmc 0` in the U-boot console. A new mass storage device should be detected on your host PC: this is the eMMC of your Jetson board.
4. partition the eMMC to have one single ext4 partition.
5. mount the eMMC partition and run the rsync command above to copy the system to the eMMC.
6. unmount the eMMC partition. This might take a while as cached data is flushed to the device.
7. press ctrl+c in the U-boot console and switch the board off.

Then turn your board on (after inserting the SD card if you synced to it!). U-boot should start the kernel we just cross-compiled. The Nouveau modules will then be loaded in turn, and you should be presented with a login prompt.

On Arch Linux, use `root` for both the login and password. If you chose to build a L4T target instead, login with user `ubuntu` and password `ubuntu`.

Errors During Boot
------------------
During boot you may encounter the following errors while Nouveau is probed:

    nouveau E[   PFIFO][57000000.gpu] unsupported engines 0x00000030
    nouveau E[     DRM] failed to create ce channel, -22

These errors are expected for the moment and won't prevent the GPU to work. If you can see the `card1` and `renderD128` nodes in `/dev/dri`, then you know the module is properly probed.

Running Programs
----------------
Once your FS is booted and the correct environment variables set, you can run kmscube as follows:

    kmscube /dev/dri/card0 /dev/dri/renderD128

Warning: on Jetson TX1, replace "card0' with "card1"!

As for weston, run `weston-launch` from a physical tty (e.g. keyboard and display, not ssh or serial). 

Tuning the GPU Frequency
------------------------
By default, the GPU will run at a low frequency. You can increase it by writing into `/sys/kernel/debug/dri/128/pstate`. Run `cat` on this file to display the valid frequencies and their code. Then, if you want to run the GPU at 684 Mhz:

    echo 0b >/sys/kernel/debug/dri/128/pstate

Authors
-------
These scripts have been crafted with love by Lauri Peltonen and Alexandre Courbot, two NVIDIA engineers. Please report bugs and issues on Github.
