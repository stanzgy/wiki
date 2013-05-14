# Openstack 每日构建说明

Openstack 每日构建服务使用buildbot搭建, 每天晚上2:34分会触发一次构建任务.

如果当天project对应的git仓库有新提交的commit, 则构建任务会自动为项目制作新的
deb安装包并导入每日构建的ppa源中.

目前做了每日构建的项目: nova, glance, keystone, umbrella, postman, billing,
sentry

自动生成的包的版本会包含
```date +%Y%m%d`.git`gitrev-parse --short HEAD`-1``的输出结果,
e.g. `2012.2+netease.20130513.git940d99a-1`.

如果添加了多个源时, 请注意确认openstack相关的包的版本.

仓库没有新的commit不会制作新的安装包.

后续会添加根据nt_version变化触发的构建任务.


> buildbot waterfall page: http://114.113.199.8:8088/waterfall

> nightly build archives: http://114.113.199.8:88/archives

> nightly ppa source: `deb http://114.113.199.8:88/debian wheezy-nightly main`

ppa源的使用可以参考[这篇文章](https://gitlab.hz.netease.com/hzzhanggy/wiki/blob/master/work/debian-ppa-howto.md)

