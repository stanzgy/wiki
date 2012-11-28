云主机镜像制作时的需求（包括：集成进去的软件包、脚本、需要加载的模块和需要开启的服务等）

    dev:    build-essential, pkg-config, dpkg, gcc, gdb, glibc,
            zlibc, libboost-dev, python, ruby, rubygems, lua
    vcs:    git, subversion, mercurial
    editor: vim, emacs
    vm:     qemu-kvm, libvirt, libvirt-dev, python2.7-libvirt
    utils:  sudo, zsh, uml-utilities, udev, ulogd, rsyslog
            openssh-client, openssh-server, proxychains, p7zip, unrar
    network:    iptables, ipset, bridge-utils, nmap
    web:    apache2, mysql-server, php5, nodejs

    modprobe.d:   kvm

(by hzzhanggy)

在低版本的内核虚拟机镜像中要加载如下两个模块：
acpiphp
pci_hotplug
用于支持磁盘动态挂载、卸载。目前测试发现内核版本为3.2以下的需要加载。
（王盼）