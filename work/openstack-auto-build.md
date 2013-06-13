# Openstack 自动构建服务说明

Openstack 自动构建服务使用buildbot搭建, 构建任务分为两类:

* 每日构建
* 每周构建

目前进行自动构建的项目: nova, glance, keystone, umbrella, postman, billing,
sentry
## 每日构建

每日构建, 每天晚上2:34会触发一次和project对应的构建任务.

如果当天project对应的git仓库有新提交的commit, 则构建任务会自动为项目制作新的
deb安装包并导入每日构建的ppa源中; project没有新提交的commit时则不会制作新的deb
安装包.

每日构建自动生成的deb包的版本模板是
[prefix]+`` `date +%Y%m%d`.git`git rev-parse --short HEAD`.nightly-1``,
e.g. `2012.2+netease.20130513.git940d99a.nightly-1`.

如果添加了多个源时, 请注意确认openstack相关的包的版本.

## 每周构建

每周构建, 每当project提交的commit中修改了`nt_version.py`文件时,
会触发对应的构建任务, 为相应的project制作新的deb安装包.

我们每周固定的时间更新`nt_version.py`文件, 就能每周自动触发相应的构建任务.

每日构建自动生成的deb包的版本模板是
`[prefix]+[nt_version].weekly-1`, e.g. `2012.2+netease.1.0.2.weekly-1`.

如果添加了多个源时, 请注意确认openstack相关的包的版本.

## 自动部署

SA在联调环境和QA测试环境部署了puppet,
并配置puppet自动更新openstack相关的包到最新的版本.

每日构建的deb安装包自动制作完成以后, 会在半小时内自动更新到QA测试环境.

每周构建的deb安装包自动制作完成以后, 会在半小时内自动更新到联调环境.

## Links

> buildbot waterfall page: http://114.113.199.8:8088/waterfall

> auto build archives: http://114.113.199.8:83

> stable repo: `deb http://114.113.199.8:82/debian wheezy main`

> nightly repo: `deb http://114.113.199.8:88/debian wheezy-nightly main`

> weekly repo: `deb http://114.113.199.8:89/debian wheezy-weekly main`

ppa源的使用可以参考[这篇文章](debian ppa howto)

