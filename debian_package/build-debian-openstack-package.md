# Build debian package with git-buildpackage

## Get the source

    stanzgy % git clone ssh://SCM/openstack/nova.git

## Checkout the debian package

    stanzgy % cd nova

    stanzgy % git checkout -b debian/folsom origin/debian/folsom

## Commit your changes

    ....

    stanzgy % git commit -a

## Add debian changelog

    stanzgy % git dch

    stanzgy % vi debian/changelog

    stanzgy % git commit -a -m 'build package'

## Build debian package with git-buildpackage

    stanzgy % git-buildpackage --git-dist=wheezy --git-arch=amd64 --git-pbuilder --git-cleaner=/bin/true -nc

## Reference

More information about debian package building toolchains, please refer to

* [Debian New Maintainers' Guide](http://www.debian.org/doc/manuals/maint-guide/)

* [Debian Policy Manual](http://www.debian.org/doc/debian-policy/)
