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
- basic 32-bit libraries
- Wayland (for the wayland-scanner program)

Under Ubuntu 14.04, the following command will get you all set:

    $ sudo apt-get install git build-essential wget phablet-tools autoconf automake libtool libc6-i386 lib32stdc++6 lib32z1 pkg-config libwayland-dev bison flex bc u-boot-tools

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

If you prefer to operate on a fresh L4T installation, then run the following script:

    ./scripts/download-rootfs

It will download the latest L4T base image as well as the proprietary L4T graphics stack, and extract both under `out/target/L4T`. You will need `sudo` rights to preserve the permissions of the target filesystem.

Now that the target filesystem can be accessed under the expected location, we will need to make sure it contains all the libraries requires to let us cross-compile the OSS stack against it:

    ./scripts/prepare-rootfs

This script downloads a static qemu-arm binary and uses it under a chroot to run `apt-get` under the target filesystem and install all the required libraries.

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

    ./scripts/build-drm
    ./scripts/build-wayland
    ./scripts/build-mesa

Then you can choose to add kmscube (useful for quickly testing that the graphics stack is working) and Weston (to enable a graphical UI):

    ./scripts/build-kmscube
    ./scripts/build-weston

Note that the binaries and libraries will all be installed under `/opt/nouveau` by default. The `prepare-rootfs` script ran previously added the necessary environment variables to `/etc/profile.d/nouveau.sh` to make them available in the PATH.

Installing to Boot Device
-------------------------
Copy the contents of `out/target/L4T` to your boot device (be it SD card or internal eMMC), and U-boot should start the kernel we just cross-compiled. The Nouveau modules will then be loaded in turn, and you should be able to run both `kmscube` and `weston-launch`.

Errors During Boot
------------------
During boot you will encounter the following errors while Nouveau is probed:

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

Authors
-------
These scripts have been crafted with love by Lauri Peltonen and Alexandre Courbot, two NVIDIA engineers. Please report bugs and issues on Github.
