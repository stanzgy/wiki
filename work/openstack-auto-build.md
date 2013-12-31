# Openstack 自动构建服务说明

## 简介

Openstack 自动构建服务使用buildbot搭建, 目前针对Openstack的Folsom与Havana两个
发行版做不同策略的自动构建.

Folsom发行版Openstack自动构建任务为:

* 每日构建(QA环境)
* 触发构建(演练环境)

Folsom发行版进行自动构建的项目: nova, glance, keystone, umbrella, postman,
billing, sentry, python-novaclient, python-glanceclient, python-keystoneclient,
python-nosclient.

Havana发行版Openstack自动构建任务为:

* 每日构建(QA环境)
* 每日构建(联调环境)
* 触发构建(演练环境)

Havana发行版目前进行自动构建的项目: nova, glance, keystone, monitor, umbrella.


## 每日构建

每日构建, 每天晚上2:34会触发一次和project对应的构建任务.

如果当天项目对应分支的git仓库有新提交的commit, 则构建任务会自动为项目制作新
的deb安装包并导入每日构建的源中; 项目git仓库没有提交新的commit时则不会制作新的
deb 安装包.

每日构建自动生成的deb包版本的模板为

    PREFIX+`date +%Y%m%dT%H%M`.git`git rev-parse --short HEAD`.nightly.SUFFIX-1

其中PREFIX为上游版本号(如果有的话), SUFFIX为st或it(表示QA环境/联调环境)

例如

    2013.2+netease.20131231T0236.git35bd7bf.nightly.st-1
    20131231T1107.git678535b.nightly.it-1


## 触发构建

触发构建, 每当project提交的commit中修改了`nt_version.py`文件内的版本号时,
会触发构建任务, 为项目制作新的deb安装包.

触发构建自动生成的deb包版本的模板为

    PREFIX+`find . -type f -name nt_version.py -exec sed -r "s/.*([0-9]+\\.[0-9]+\\.[0-9]+).*/\\1/" {} \+`-1

其中PREFIX为上游版本号(如果有的话).

例如

    2013.2+netease.2.0.7-1
    1.0.3-1


## 自动部署

SA在QA环境, 联调环境和演练环境都部署了puppet, 并配置puppet自动更新openstack
相关的包到最新的版本.

自动构建生成的deb安装包由buildbot自动制作完成以后, 会通过puppet每日清晨自动
更新部署到相应的环境上.


## 源配置

需要使用打过我们补丁的1.1.4版本libvirt, 需要单独添加源

    deb http://114.113.199.8:82/debian wheezy-libvirt-backports main contrib non-free

**请注意, 添加该源安装libvirt, 会导致libc升级到2.17**

### Folsom版本QA环境(24, 25)

    deb http://114.113.199.8:82/debian wheezy-folsom-backports main contrib non-free
    deb http://114.113.199.8:82/debian wheezy-folsom-nightly main contrib non-free

### Folsom版本联调环境(obsolete)

    deb http://114.113.199.8:82/debian wheezy-folsom-backports main contrib non-free
    deb http://114.113.199.8:82/debian wheezy-folsom-testing main contrib non-free

### Folsom版本演练环境(28, 29, 40)

    deb http://114.113.199.8:82/debian wheezy-folsom-backports main contrib non-free
    deb http://114.113.199.8:82/debian wheezy-folsom-rc main contrib non-free

### Havana版本QA环境(7, 21, 22)

    deb http://114.113.199.8:82/debian wheezy-havana-backports main contrib non-free
    deb http://114.113.199.8:82/debian wheezy-havana-nightly main contrib non-free

### Havana版本联调环境(11, 12, 13, 37, 38, 39)

    deb http://114.113.199.8:82/debian wheezy-havana-backports main contrib non-free
    deb http://114.113.199.8:82/debian wheezy-havana-testing main contrib non-free

### Havana版本演练环境(暂无)

    deb http://114.113.199.8:82/debian wheezy-havana-backports main contrib non-free
    deb http://114.113.199.8:82/debian wheezy-havana-rc main contrib non-free


## Links

> buildbot waterfall page: http://114.113.199.8:8088/waterfall

> build archives: http://114.113.199.8:82/archives/

> opstanck ppa源的使用可以参考[这篇文章](debian ppa howto)

