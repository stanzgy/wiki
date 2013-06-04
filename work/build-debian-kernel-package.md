# Build Debian Kernel package

## Chroot to target environment

    # DIST=wheezy ARCH=amd64 git-pbuilder login

## Install essential tools

    # aptitude install fakeroot kernel-package build-essential

## Get kernel source

    # aptitude install linux-source-3.2

## Apply your changes

    # tar jxf /usr/src/linux-source-3.2.tar.bz2 .

    # make menuconfig

    # patch ...

## Build kernel package

    # make-kpkg -j24 --rootcmd fakeroot --initrd --revision 3.2.41-2+netease1
    --append_to_version -openstack-amd64 kernel_image
