# 从源码安装配置Neutron开发环境

## 安装配置havana版本nova/glance/keystone服务

安装配置过程略

### 修改nova.conf中与neutron相关的配置

    network_api_class=nova.network.neutronv2.api.API
    neutron_url=http://%neutron_host%:9696
    neutron_admin_auth_url=http://%neutron_admin_host%:5000/v2.0
    neutron_admin_username=%neutron_admin_user%
    neutron_admin_password=%neutron_admin_pw%
    neutron_admin_tenant_name=service=%neutron_admin_tenant%
    neutron_region_name=neutron=%neutron_region%
    neutron_auth_strategy=keystone
    neutron_ovs_bridge=br-int

    libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtOpenVswitchDriver

## 编译安装Open vSwitch

### 确认内核版本

确认kernel版本   =3.8, kernel在3.8版本以后才支持vxlan

### 安装依赖

请根据源码目录下的INSTALL文件确认build-essential, libc, libssl, clang,
iproute2, linux-headers, autoconf, automake, python,
perl等包正确安装并且版本符合要求.

### 获得源代码

    % git clone git://git.openvswitch.org/openvswitch

### 通过制作deb安装包安装openvswitch(推荐)

#### 将openvswitch源码编译成deb安装包

    % cd openvswitch

    % dpkg-buildpackage -b -rfakeroot

打包完成后, 生成的openvswitch deb安装包默认存放在上一级目录下

#### 安装openvswitch安装包

    % dpkg -i openvswitch-common_1.12.90-1_amd64.deb

    % dpkg -i openvswitch-controller_1.12.90-1_amd64.deb

    % dpkg -i openvswitch-datapath-dkms_1.12.90-1_all.deb

    % dpkg -i openvswitch-pki_1.12.90-1_all.deb

    % dpkg -i openvswitch-switch_1.12.90-1_amd64.deb

    % dpkg -i openvswitch-test_1.12.90-1_all.deb

#### 启动openvswitch服务

    % sudo service openvswitch-switch restart

### 通过编译源代码手动安装openvswitch

#### configure

    % ./boot.sh

    % ./configure --prefix=/usr --with-linux=/lib/modules/`uname -r`/build

#### make & install

    % make

    % make install

    % make modules_install

    % modprobe openvswitch

#### 启动openvswitch服务

    % mkdir -p /usr/local/etc/openvswitch

    % ovsdb-tool create /usr/local/etc/openvswitch/conf.db vswitchd/vswitch.ovsschema

    % ovsdb-server --remote=punix:/usr/local/var/run/openvswitch/db.sock
                   --remote=db:Open_vSwitch,Open_vSwitch,manager_options
                   --private-key=db:Open_vSwitch,SSL,private_key
                   --certificate=db:Open_vSwitch,SSL,certificate
                   --bootstrap-ca-cert=db:Open_vSwitch,SSL,ca_cert
                   --pidfile --detach

    % ovs-vsctl --no-wait init

    % ovs-vswitchd --pidfile --detach

### 确认openvswitch正常运行

* ovsdb-server, ovs-vswitchd两个进程正常运行
* sudo ovs-vsctl show命令无错误输出

## 编译安装Neutron服务

### 获得源代码

    % git clone https://github.com/openstack/neutron.git

### 安装neutron

请根据需要选择neutron的版本, 这里使用的是master分支的最新代码

    % cd neutron

    % git checkout master

    % python setup.py build

    % sudo python setup.py install

### 修改neutron.conf中的相关选项

常规选项请自行修改, 这里列举一些关键选项的配置

#### ML2 configs

    [ml2]
    # ml2 plugin configs
    type_drivers = vxlan,gre,vlan,local
    tenant_network_types = vxlan,gre,vlan,local

    [ml2_type_vxlan]
    vni_ranges=1:1000
    vxlan_group=239.1.1.1

#### DHCP agent

    [DEFAULT]
    # dhcp-agent configs
    ovs_use_veth = True
    enable_isolated_metadata = True
    use_namespaces = True

#### L3 agent

    [DEFAULT]
    # l3-agent configs
    # OVS based plugins (OVS, Ryu, NEC) that supports L3 agent
    interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver

#### Open vSwitch agent

    [agent]
    # openvswitch plugin configs
    tunnel_types = vxlan,gre

    [ovs]
    # openvswitch plugin configs
    local_ip = 10.120.120.31
    #bridge_mappings = physnet1:br-eth1.301
    tenant_network_type = vxlan
    # vlan type
    #network_vlan_ranges =
    enable_tunneling = True
    tunnel_type = vxlan
    tunnel_id_ranges = 1:1000

### 启动neutron服务

#### 主节点

* neutron-server
* neutron-dhcp-agent
* neutron-openvswitch-agent

#### 从节点

* neutron-openvswitch-agent

### 建立L2网络

#### 建立vxlan隧道网络

    % neutron net-create --provider:network_type vxlan
                         --provider:segmentation_id 10
                         --tenant-id TENANT_ID --shared
                         net1

#### 建立gre隧道网络

    % neutron net-create --provider:network_type gre
                         --provider:segmentation_id 20
                         --tenant-id TENANT_ID --shared
                         net2
### 建立L3网络

    % neutron subnet-create net1 10.0.10.0/24

    % neutron subnet-create net2 10.0.20.0/24

### 建立虚拟机, 使用neutron的网络

    % nova boot --image debian-6.0.4-amd64-standard
                --flavor 2
                --nic net-id=NETWORK_ID
                test_neutron

    如果可以正常创建虚拟机, 说明nova和neutron已经可以正常工作了

## 常见问题

### neutron服务提示br-int不存在, 无法启动

手工创建br-int

    % ovs-vsctl add-br br-int

### neutron配置文件不生效

neutron默认只读取/etc/neutron/neutron.conf中的配置项, 对于/etc/neutron/plugins
和/etc/neutron目录下的其他配置文件, 需要在启动neutron服务时手工指定配置文件才可
生效.

推荐将所有配置项都写入到/etc/neutron/neutron.conf中, 这样启动服务时所有配置
不许要指定配置文件就可生效, 比较省事.

### openvswitch网络跨节点无法互通

曾经发现openvswitch隧道不通的问题, 原因是内核的ip_gre模块和openvswitch模块冲突.
将ip_gre模块卸载后, 手工加载gre模块和openvswitch模块可以解决这个问题.

### 使用neutron网络后, 如何连接虚拟机

方法一: 配置L3 agent, 给虚拟机绑定floating ip, 通过floating ip访问虚拟机

方法二: 通过dhcp的net namespace访问

在dhcp agent运行的宿主机上查找虚拟机所在的net namespace, 并切换到相应的
net namespace内

    % ip netns list

    % ip netns exec NETNS_NAME bash

在net namespace内, 可以直接通过虚拟机分配到的IP访问虚拟机.

