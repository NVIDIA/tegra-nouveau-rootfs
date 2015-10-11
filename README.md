Linux For Tegra (L4T) Nouveau Installer Scripts
===============================================
These scripts aim at providing an simple way to enable the open-source graphics stack (Nouveau/Mesa) on Jetson TK1. It does so by automating the process of cross-compiling the necessary software and adapting a new or already-existing L4T root filesystem to run Nouveau/Mesa as an alternative to the closed-source graphics stack.

The following components can be installed to a L4T root FS by following the instructions given in this document:
- Linux kernel
- Nouveau modules
- GPU firmware
- drm
- Wayland
- Mesa
- kmscube
- Weston

Host System Preparation
-----------------------
You will need a bunch of tools to download and cross-compile the various projects needed to enable Nouveau on L4T. Depending on your distribution, you will need to install:

- repo
- git
- autotools
- uboot-tools
- basic 32-bit libraries
- Wayland (for the wayland-scanner program)

Under Ubuntu 14.04, the following command will get you all set:

    $ sudo apt-get install git build-essential wget phablet-tools autoconf automake libtool libc6-i386 lib32stdc++6 lib32z1 pkg-config libwayland-dev bison flex bc u-boot-tools glib-2.0 realpath libffi-dev

Under Archlinux (2015-10-09), run the following commands:

    $ yaourt -S aur/repo # Or install the package by downloading it from AUR yourself
    $ sudo pacman -Sy base-devel wget git crypto++ libffi uboot-tools wayland

U-Boot
------
The first prerequisite is that you must use an up-to-date U-Boot as bootloader. Jetson TK1 comes with another bootloader flashed ; make sure to follow the instructions on https://github.com/NVIDIA/tegra-uboot-flasher-scripts/blob/master/README-developer.txt to easily flash U-Boot and be safe.

**Warning:** running an outdated U-Boot will cause the kernel to silently crash when loading the GPU driver!

Syncing
-------
All the required projects are synced using Google's `repo` tool:

    mkdir l4t-nouveau
    cd l4t-nouveau
    repo init -u https://github.com/NVIDIA/tegra-nouveau-rootfs.git -m l4t-nouveau.xml
    repo sync -j4 -c

Once all the sources are downloaded, set the TOP environment variable:

    export TOP=$PWD

then download the cross-compilation toolchain that we will use:

    ./scripts/download-gcc

Preparing the Target Filesystem
-------------------------------
All the scripts expect to find your L4T filesystem under `out/target/L4T`. If you wish to use an already-existing rootfs, simply create a link from `out/target/L4T` to the root of your L4T filesystem.

If you prefer to operate on a fresh ArchLinux installation, define the DISTRO value before downloading rootfs, as here:

    export DISTRO=ArchLinuxArm

If you prefer to operate on a fresh L4T installation, then run the following script:

    ./scripts/download-rootfs

It will download the latest L4T base image as well as (optionally) the proprietary L4T graphics stack, and extract both under `out/target/L4T`. You will need the ability to run `sudo` in order to preserve the permissions of the target filesystem.


Now that the target filesystem can be accessed under the expected location, we will need to make sure it contains all the libraries requires to let us cross-compile the OSS stack against it:

    ./scripts/prepare-rootfs

This script downloads a static qemu-arm binary and uses it under a chroot to run `apt-get` under the target filesystem and install all the required libraries. It also requires `sudo` to run.

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
    ./scripts/build-libinput
    ./scripts/build-wayland
    ./scripts/build-mesa

Then you can choose to add kmscube (useful for quickly testing that the graphics stack is working) and Weston (to enable a graphical UI):

    ./scripts/build-kmscube
    ./scripts/build-weston

The binaries and libraries will all be installed under `/opt/nouveau` by default. The `prepare-rootfs` script ran previously added the necessary environment variables to `/etc/profile.d/nouveau.sh` to make them available in the PATH.

Note that the `build-weston` script requires `sudo` in order to set the SUID bit to the `weston-launch` script.

Installing to Boot Device
-------------------------
You will then need to Copy the contents of `out/target/L4T` to your boot device (which could be a SD card or internal eMMC).

To copy the root filesystem to a mounted (and empty) ext4-formatted SD card:
(replace the L4T target folder by ArchLinuxArm or $DISTRO if you have chosen ArchLinux rootfs)

    sudo rsync -aAXv $TOP/out/target/L4T/* /path/to/mount/point/ 


If you prefer to sync to the internal eMMC, do the following:

1. turn your Jetson TK1 on, and on the serial console press a key to enter the U-boot menu.
2. connect a USB cable to the Jetson's micro-USB port.
3. type `ums 0 mmc 0` in the U-boot console. A new mass storage device should be detected on your host PC: this is the eMMC of your Jetson board.
4. partition the eMMC to have one single ext4 partition.
5. mount the eMMC partition and run the rsync command above to copy the system to the eMMC.
6. unmount the eMMC partition. This might take a while as cached data is flushed to the device.
7. press ctrl+c in the U-boot console and switch the board off.

Then turn your board on (after inserting the SD card if you synced to it!). U-boot should start the kernel we just cross-compiled. The Nouveau modules will then be loaded in turn, and you should be presented with a login prompt (login with user `ubuntu` and password `ubuntu`).

Errors During Boot
------------------
During boot you may encounter the following errors while Nouveau is probed:

    nouveau E[    PBUS][57000000.gpu] MMIO read of 0x00000000 FAULT at 0x17e8dc
    ...
    nouveau E[   PFIFO][57000000.gpu] unsupported engines 0x00000030
    nouveau E[     DRM] failed to create ce channel, -22

These errors are expected for the moment and won't prevent the GPU to work. If you can see the `card1` and `renderD128` nodes in `/dev/dri`, then you know the module is properly probed.

Running Programs
----------------
Once your FS is booted and the correct environment variables set, you can run kmscube as follows:

    kmscube /dev/dri/card0 /dev/dri/renderD128

As for weston, run `weston-launch` from a physical tty (e.g. keyboard and display, not ssh or serial). 

Tuning the GPU Frequency
------------------------
By default, the GPU will run at a low frequency of 198 Mhz. You can increase this by writing into `/sys/class/drm/card1/device/pstate`. Run `cat` on this file to display the valid frequencies and their code. Then, if you want to run the GPU at 684 Mhz:

    echo 0b >/sys/class/drm/card1/device/pstate

Authors
-------
These scripts have been crafted with love by Lauri Peltonen and Alexandre Courbot, two NVIDIA engineers. Please report bugs and issues on Github.
