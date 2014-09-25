L4T Nouveau Installer Scripts
=============================
This manifest repository should allow anyone to cross-compile all the required kernel components and user-space libraries to run the OSS graphics stack (Mesa + Nouveau) under L4T on the Jetson TK1. It will install the new kernel, modules, and libraries on either an existing L4T image or a freshly downloaded one.

The following components can be installed using the scripts downloaded by this manifest:
- Linux kernel
- Nouveau modules
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

    $ sudo apt-get install git build-essential curl phablet-tools autoconf automake libtool libc6-i386 lib32stdc++6 lib32z1 pkg-config libwayland-dev bison flex bc u-boot-tools


Syncing
-------
The first step is to sync all the requires projects using repo:

    mkdir l4t-nouveau
    cd l4t-nouveau
    repo init -u https://github.com/Gnurou/tegra-nouveau-rootfs.git -m l4t-nouveau.xml
    repo sync -j4 -c

Once all the sources are downloaded, set the TOP environment variable:

    export TOP=$PWD

then download the cross-compilation toolchain that we will use:

    ./scripts/download-gcc

Preparing the Target Filesystem
-------------------------------
All the scripts expect to find your L4T filesystem under `out/target/L4T`. If you wish to use an already-existing rootfs, simply create a from `out/target/L4T` to the root of your L4T filesystem.

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

They will end in `/lib/modules/KERNEL_VERSION/extra`.

Compiling User-space Components
-------------------------------
The essential user-space components can be compiled as follows:

    ./scripts/build-drm
    ./scripts/build-wayland
    ./scripts/build-mesa

Then you can choose to add kmscube (useful for quickly testing that the graphics stack is working) and Weston (to enable a graphical UI):

    ./scripts/build-kmscube
    ./scripts/build-weston

Note that the binaries and libraries will all be installed under `/home/ubuntu/usr`, so make sure to add `/home/ubuntu/usr/bin` to the PATH and to set LD_LIBRARY_PATH to `/home/ubuntu/usr/lib`. One easy way to do this is to add the following lines:

    export PATH="$HOME/usr/bin:$PATH"
    export LD_LIBRARY_PATH="$HOME/usr/lib:$LD_LIBRARY_PATH"

to `/home/ubuntu/.profile` on the target FS.

Conclusion
----------
You should be all set now - copy the contents of `out/target/L4T` to your boot device (be it SD card or internal eMMC), and U-boot should start the kernel we just cross-compiled. The Nouveau modules will then be loaded in turn, and if you set your environment variables properly, you will be able to run both `kmscube` and `weston-launch`.
