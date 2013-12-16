# openstack netease ppa源使用方法

## 添加pgp key到aptitude

运行
    root# curl http://114.113.199.8:82/cloudpkg_pub.asc|apt-key add -

然后`apt-key list`看一下, 如果看到下面的内容说明key导入成功了, 可以进行下面得流程

    pub   2048R/36225635 2013-02-26
    uid                  NetEase Openstack PPA Signing Key <stan.zgy@gmail.com>


## 修改/etc/apt/sources.list, 加入我们的ppa源

    root# echo "deb http://114.113.199.8:82/debian wheezy-xxx main contrib non-free" >> /etc/apt/sources.list


## 更新aptitude index

    root# aptitude update


## 确认版本, 安装需要的package

以python-nova为例

    root# aptitude show python-nova

输出的结果中找到这行

    Version: 2012.2+netease.m4.3.4-1

确认是否是想要的package版本. 如果version中有netease, 说明是我们自己的package, 可以直接安装了.
PS: 只有我们自己做了修改和开发的package, version中才会有netease. 比如nova, glanceclient等

    root# aptitude install python-nova

如果版本号不对, 可能是添加了更新的源(experimental)导致, 需要手工指定版本号安装.

查看可用的package list

    root# apt-cache showpkg python-nova

在输出的结果中找到provides list

    Provides:
    2012.2+netease.m4.3.4-1 -
    2012.1.1-13 -

## 指定合适的版本安装

    root# aptitude install python-nova=2012.2+netease.m4.3.4-1
