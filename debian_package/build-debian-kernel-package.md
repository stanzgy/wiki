# Build Debian Kernel package

## Debootstrap to target environment

    # DIST=wheezy ARCH=amd64 git-pbuilder login

## Install essential tools

    # aptitude install fakeroot kernel-package build-essential

## Get kernel source

    # aptitude install linux-source-3.2

    # tar jxf /usr/src/linux-source-3.2.tar.bz2

## Apply your changes

    # cd linux-source-3.2

    # make menuconfig

    # patch ...

## Update package maintainer information

    # sed -i 's/^maintainer \:=.*//' /etc/kernel-pkg.conf

    # sed -i 's/^email \:=.*//' /etc/kernel-pkg.conf

    # echo "maintainer := Zhang Gengyuan" >> /etc/kernel-pkg.conf

    # echo "email := stan.zgy@gmail.com" >> /etc/kernel-pkg.conf

## Build kernel package

    # make-kpkg -j24 --rootcmd fakeroot --initrd --revision 3.2.41-2+netease1
      --append_to_version -openstack-amd64 kernel_image
