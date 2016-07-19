## Devstack 开发环境搭建指南

搭建 Devstack 开发环境一直是一件挺头疼的事情，特别是对于新人来说，
容易遇到各种各样的问题而失败。
最近借着整理 Jenkins 的机会把我们的 Havana 版本 Devstack 重新整理了一遍，
修复了一些问题，使其可以比较简单快速的搭建起来一个 Openstack 开发环境。

下面详细说明。

### 单节点 Devstack 安装

#### 准备一台 Ubuntu 14.04 虚拟机

我们目前建议直接在云计算线上环境 (yun.163.com) 创建你的虚拟机开发环境。

> 请注意一定要使用 Ubuntu 14.04 的镜像创建虚拟机

创建好虚拟机后，需要你做一些最基本的系统设置、初始化工作，
比如创建你自己的账号、安装 vim git sudo 等基础软件等工作，这里不做赘述。

> 如果你经常创建虚拟机，可以使用 ansible/puppet
> 等工具来初始化虚拟机，非常方便

#### 设置 Python 环境

确认安装必要的依赖包

    $ aptitude install python-dev libffi-dev libssl-dev

创建/修改你的 pip.conf 配置文件，将 index-url 修改为我们的私有 PyPI 源

    $ cat ~/.pip/pip.conf
    [global]
    index-url = https://pypi.cloud.netease.com/jenkins/dev/+simple
    timeout=30

安装/更新 pip tox virtualenv 等常用软件

    $ pip install -U 'pip<8.1.2' tox virtualenv ndg-httpsclient pyopenssl pyasn1

> 请注意，目前发现 pip 版本 >=8.1.2 时某些情况下会有兼容性问题。

#### 获取 Devstack 源码

Gitlab

    $ git clone ssh://git@g.hz.netease.com:22222/openstack-dev/devstack -b netease/havana

或 Gerrit

    $ git clone ssh://${your_gerrit_acct}@review.cloud.netease.com:2222/openstack/devstack -b netease/havana

> 请注意要使用 netease/havana 分支

#### 创建你的 localrc 配置文件

这里可以直接使用我刚刚更新的 localrc 文件，见 devstack/samples/localrc

    # General settings
    export OS_NO_CACHE=True

    DEST=/opt/stack/new
    DATA_DIR=/opt/stack/data

    VERBOSE=True
    USE_SCREEN=True
    SYSLOG=False
    LOG_COLOR=True
    SCREEN_LOGDIR=/opt/stack/new/screen-logs
    LOGFILE=/opt/stack/new/devstacklog.txt
    DATABASE_QUERY_LOGGING=True

    IMAGE_URLS=https://review.cloud.netease.com/static/cirros-0.3.0-x86_64-disk.img
    TEMPEST_HTTP_IMAGE=https://review.cloud.netease.com/static/cirros-0.3.0-x86_64-disk.img

    ACTIVE_TIMEOUT=90
    BOOT_TIMEOUT=90
    ASSOCIATE_TIMEOUT=60
    TERMINATE_TIMEOUT=60
    ROOTSLEEP=0

    MYSQL_PASSWORD=secretmysql
    DATABASE_PASSWORD=secretdatabase
    RABBIT_PASSWORD=secretrabbit
    ADMIN_PASSWORD=secretadmin
    SERVICE_PASSWORD=secretservice
    SERVICE_TOKEN=111222333444
    SWIFT_HASH=1234123412341234

    ENABLED_SERVICES=dstat,g-api,g-reg,key,mysql,n-api,n-cond,n-cpu,n-crt,n-sch,q-agt,q-dhcp,q-meta,q-svc,q-vpn,quantum,rabbit
    SKIP_EXERCISES=boot_from_volume,bundle,client-env,euca
    SERVICE_HOST=127.0.0.1
    # Don't reset the requirements.txt files after g-r updates
    UNDO_REQUIREMENTS=False
    TRACK_DEPENDS=False


    # Git settings
    GIT_BASE=ssh://git@g.hz.netease.com:22222
    ERROR_ON_CLONE=False
    LIBS_FROM_GIT=
    REQUIREMENTS_BRANCH=netease/havana
    PBR_BRANCH=netease
    NOVA_BRANCH=netease/havana
    GLANCE_BRANCH=netease/havana
    KEYSTONE_BRANCH=netease/havana
    NEUTRON_BRANCH=netease/havana
    CINDER_BRANCH=netease/havana
    NOVACLIENT_BRANCH=netease/havana
    GLANCECLIENT_BRANCH=netease/havana
    KEYSTONECLIENT_BRANCH=netease/havana
    NEUTRONCLIENT_BRANCH=netease/havana
    CINDERCLIENT_BRANCH=netease/havana
    OPENSTACKCLIENT_BRANCH=netease/havana
    TEMPEST_BRANCH=netease/havana
    UMBRELLA_BRANCH=netease/havana
    SENTRY_BRANCH=netease/havana


    # Neutron settings
    NETWORK_GATEWAY=10.1.0.1
    ENABLE_TENANT_TUNNELS=True
    TENANT_TUNNEL_RANGE=50:100
    Q_PLUGIN=ml2
    Q_USE_DEBUG_COMMAND=False
    Q_USE_SECGROUP=False
    Q_SERVICE_PLUGIN_CLASSES='router,vpnaas'
    Q_SERVICE_PROVIDER=VPN:sslvpn:neutron.services.vpn.service_drivers.sslvpn.SSLVPNDriver:default
    Q_ML2_TENANT_NETWORK_TYPE=vxlan,gre
    Q_ML2_PLUGIN_VXLAN_TYPE_OPTIONS=(vni_ranges=400:500)
    Q_ML2_PLUGIN_MECHANISM_DRIVERS=openvswitch,svlan,l2population
    Q_ML2_PLUGIN_TYPE_DRIVERS=vxlan,gre,svlan,flat


    # Nova settings
    LIBVIRT_TYPE=qemu
    VIRT_DRIVER=libvirt
    LIBVIRT_INJECT_PARTITION=-2
    LIBVIRT_FIREWALL_DRIVER=nova.virt.firewall.NoopFirewallDriver
    NOVA_VIF_DRIVER=nova.virt.libvirt.vif.LibvirtGenericVIFDriver
    LINUXNET_VIF_DRIVER=nova.network.linux_net.LinuxOVSInterfaceDriver
    # set this until all testing platforms have libvirt >= 1.2.11
    # see bug #1501558
    EBTABLES_RACE_FIX=True
    FORCE_CONFIG_DRIVE=True

    FIXED_RANGE=10.1.0.0/20
    FLOATING_RANGE=172.24.5.0/24
    PUBLIC_NETWORK_GATEWAY=172.24.5.1
    FIXED_NETWORK_SIZE=4096

    CINDER_PERIODIC_INTERVAL=10
    CINDER_SECURE_DELETE=False
    CINDER_VOLUME_CLEAR=none
    CEILOMETER_BACKEND=mysql
    SWIFT_REPLICAS=1
    VOLUME_BACKING_FILE_SIZE=24G

这份配置将从 g.hz.netease.com 拉取各个项目的代码并安装一个 nova/neutron 的最小
openstack 开发环境，其中

    ENABLED_SERVICES=dstat,g-api,g-reg,key,mysql,n-api,n-cond,n-cpu,n-crt,n-sch,q-agt,q-dhcp,q-meta,q-svc,q-vpn,quantum,rabbit

变量控制 devstack 开启的服务，如果需要开启 umbrella/sentry 等服务，
将这些服务加入该变量即可。

所有的项目源码安装在 `/opt/stack/new` 目录下，
所有服务的 data 文件夹在 `/opt/stack/data` 目录下，
服务日志在 `/opt/stack/new/screen-logs` 目录下。

请注意其中的这段配置设置了各个服务的密码，请将其更新为你需要的内容

    MYSQL_PASSWORD=secretmysql
    DATABASE_PASSWORD=secretdatabase
    RABBIT_PASSWORD=secretrabbit
    ADMIN_PASSWORD=secretadmin
    SERVICE_PASSWORD=secretservice
    SERVICE_TOKEN=111222333444
    SWIFT_HASH=1234123412341234

如果你需要修改各个项目的 git 仓库地址与使用的分支，则需要修改下面这段内容

    GIT_BASE=ssh://git@g.hz.netease.com:22222
    ...

这份配置默认将 screen 输出自动着色，在日志文件中会出现许多乱码一样的颜色格式字符，
如果你不想这样，设置变量

    LOG_COLOR=False

其他许多配置项请自行研究修改，有不明白的地方可以提出来和大家交流。

#### 运行 Devstack

准备好 localrc 文件后，将其放到 devstack 根目录下，就可以直接运行 devstack 了。

    $ cd devstack
    $ ./stack.sh

视机器性能与网络条件，一般过 20 分钟到 1 个小时不等，不报任何错误执行完的话，
一个单节点的 devstack 开发环境就建立好了。

如果运行 `stack.sh` 脚本的过程中发生了任何错误，请根据错误提示解决后，
清理好环境再重新运行 `stack.sh` 脚本。

### 多节点 Devstack 环境搭建

多节点 Devstack 环境，说白了就是几个单节点 Devstack 环境放在一起公用相同的
database/rabbitmq/keystone 等公共服务实现，通过修改配置即可实现，这里不做详细
说明。

### Devstack 环境的使用

Devstack 环境建立成功后，默认会创建两个 tenant: admin/demo
可以通过 Devstack 提供的 openrc 文件获取对应 tenant 的身份信息。

    $ cd devstack
    $ source openrc admin admin
    (run commands as admin)
    ...
    $ source openrc demo demo
    (run commands as demo)

### 其他

如果使用 Devstack 过程中发现 Devstack 源码有任何问题/Bug，欢迎直接通过 Gerrit
提交 patchset 修复。
