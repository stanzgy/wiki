# CI 系统迁移小结

由于之前我们使用了四年的 Gerrit 服务器过于老旧需要下线，加上之前田田离职、Jenkins 等测试服务无人维护的缘故，我接手并借着这个机会将我们目前使用的整个 CI 系统升级迁移到了新服务器上。下面是这次迁移过程的小结。

> TL;DR: 请大家尽快登录新 Gerrit，在个人设置页面设置 corp 邮箱前缀作为 Gerrit 用户名，方便我为大家的账号添加权限。
> [https://review.cloud.netease.com](https://review.cloud.netease.com)


## Openstack CI 系统简介

Openstack 社区的 CI 系统 workflow 如下图所示：

![CI_workflow_community](CI_workflow_community.png)

图中涉及到的 CI 服务/组件说明：

* Gerrit: 代码 Review 工具
* Jenkins: 测试核心服务，图中特指 Jenkins master 节点
* Zuul: Openstack 社区开发的新服务，用来关联 Gerrit 和 Jenkins 服务。Zuul 通过 gerrit stream-events 监听 Gerrit 上的各种事件，并根据配置触发 Jenkins 运行对应的测试任务
* CI Cloud: 所有 Jenkins 的 slave 运行在一个独立维护 Openstack 环境中，每个 slave 实际上就是一个 VM
* Nodepool: 维护 Openstack 环境中 slave VM 的服务，会自动创建 slave、并在 slave 运行完一次测试后删除并重建 VM
* Jenkins Job Builder: 由于 Openstack 下子项目众多，几乎不可能为每个项目手工维护这么多 Jenkins 测试任务的配置，社区开发了这个工具自动生成、注入测试任务的设置到 Jenkins 中
* 其他还有一些不是很重要的 PyPI/Puppet/Ansible/Gearman 等服务 ，以及 Jenkins slave 运行完各种任务进行文档/安装包/日志等内容发布的外围服务器，这里不做详细说明


## 老 CI 系统存在的问题

我们目前运行的 CI 系统是大概三年前从无到有徒手建立，陪我们逐步成长，慢慢发展完善到目前的样子，不过也存在一些主要问题：

1. Jenkins 纯手工维护。目前 Jenkins 有接近 100 个测试任务，每个任务都有复杂的测试步骤与各种环境变量的配置。如果当前 Jenkins 服务器损坏，需要手工重建或迁移的话，工作量与难度不可想象。
2. 慢！目前每个 Gerrit 提交几乎都要等一个小时才能获得 Jenkins 运行结果，比较影响开发效率。
3. 目前的 Openstack CI Cloud 问题很大。目前我们的 Openstack CI Cloud 是用一个很老的 Openstack 版本搭建的，纯手工维护，几乎不敢做任何操作，一旦出现问题很难解决；由于Openstack 版本太老，没有安装包、配置麻烦，非常难扩展新的物理节点；并且我们没有使用 Nodepool 服务，所有 slave 不会在跑完一次测试后重建，如果某个测试对 slave vm 造成了破坏，会一直保留下来，这些破坏累积起来可能导致 slave 不能工作。
4. 各个项目非常错误的通过维护私有 PyPI 源中的软件包版本来控制测试环境的建立，这个私有 PyPI 源一旦损坏将导致整个 CI 环境瘫痪。


## 新 CI 系统如何解决这些问题

有了是用老 CI 系统的这些经验，我们在迁移升级新 CI 系统时就可以吸取经验，针对性解决掉这些问题：

1. 引入社区的 Jenkins Job Builder 工具，自动化生成管理 Jenkins 的任务配置，不管是 100 还是 1000 个 Jenkins 任务，都只需要一条命令行就可以生成管理。
2. 引入几乎是为运行 CI 测试量身定做的 Docker 服务，使用 Docker containers 代替目前老旧的 Openstack VMs 运行测试任务，能能充分利用宿主机性能，并且省掉了删除建立虚拟机的时间。以 Neutron 项目为例，实测 py27 单元测试时间从原来的 24 分钟缩短到 7 分钟、tempest full 测试从 46 分钟缩短到了 18 分钟。
3. 引入 Jenkins Docker Plugin，只需要使用 Dockerfile 生成了运行测试的 docker slave image，在 Jenkins 中简单设置以后就可以运行测试了。扩展新物理节点及其方便，只需要在新节点安装 Docker 服务，再在 Jenkins 中做简单配置就可以完成。同时 Docker container 一次性使用的天然特性直接帮我们省略掉了 Nodepool 服务去创建/删除/重建 slave vm 的大量耗时工作。
4. 花了近一个月的时间整理我们各个项目的 requirement.txt，并且搭建新的私有 PyPI server。使用符合 PEP-440 规范的版本命名方法重新整理了 requirements.txt 和新 PyPI server 中的软件包，使 requirements.txt 中的依赖可以同时兼容社区版本与我们自己的二次开发版本。


## 新 CI 系统 workflow

![CI_workflow_netease](CI_workflow_netease.png)

和社区的 CI 系统相比，主要区别是我们使用 Jenkins Docker Plugin 和 Docker 服务替换了社区使用的 Nodepool 和 Openstack Cloud，其他部分基本一样。

和我们目前的 CI 系统相比，除了与社区的区别，主要是将所有组件的版本从三年前的老版本同步到与社区一致的最新版本，并且引入了 Jenkins Job Builder 自动管理 Jenkins 任务。

## 新 Jenkins 测试任务说明

* {project}-pep8: pep8检查，自动在项目目录运行 tox -epep8
* {project}-python27: 单元测试，自动在项目目录运行 tox -epy27
* {project}-pylint: pylint检查，部分项目有，自动在项目目录运行 tox -epylint，目前由于 docker 节点较少暂时关闭
* gate-tempest-dsvm-umbrella: tempest 的 umbrella 冒烟用例
* gate-tempest-dsvm-neutron: tempest 的 neutron 冒烟用例
* gate-tempest-dsvm-full: tempest 的 full 冒烟用例

## 新 Gerrit 使用说明

### 身份认证

新 Gerrit 身份认证从以前的 LDAP 切换到了新的 OpenID 上，大家第一次登录后请在 Gerrit 个人设置页面设置自己的 corp 邮箱前缀为自己的 Gerrit 用户名。

> NOTE: 由于 Gerrit 的用户账号要在用户第一次登录后才会创建，请大家尽快登录新 Gerrit，方便我为大家的账号添加权限。
> [https://review.cloud.netease.com](https://review.cloud.netease.com)

### 仓库管理

1. Openstack 社区已有的项目，完全镜像社区的命名方法镜像到我们的 Gerrit 上，如 openstack/nova 、 openstack-dev/devstack
2. 我们的自研项目，以及一些 patch 过的 openstack 相关项目，放入 openstack-netease 命名空间，如 openstack-netease/umbrella 、 openstack-netease/libvirt
3. 非 Openstack 的项目放入各自的命名空间，如 docker/* 

### 分支管理

保持之前的管理方法，Openstack 社区的分支完全保留，我们自己的开发分支，根据使用的社区 codename 加 netease/ 前缀，如 netease/havana。

### 权限管理

之前的 Gerrit 权限比较混乱，目前新的 Gerrit 和 Openstack 社区一致，统一改为每个项目 {project} 会有一个 {project}-core 用户组为其 Owner ，具有 Code-Review +2 和 Workflow +1 的权限。

> 需要注意新的 Gerrit 代码合入方式改为和社区一致，core 用户 Workflow +1 后，Jenkins 会运行一些测试，通过后由 Zuul 将代码合入仓库。（而不是现在的直接 Submit 按钮手工合入）

云主机相关的项目 core 为王盼，云网络相关的项目 core 为徐城利。具体项目可单独调整。

### 新的 Zuul Status 页面

从社区搬过来了新的 Zuul status page ，使用 graphite 绘制，代替之前非常朴素的 plain text 页面。

预览：

![zuul_status](zuul_status.png)


## 遗留问题

由于一些之前田田留下的打包任务，在新的打包方案使用前仍需保留，所以老的 Jenkins 和 Gerrit 仍然继续运行一段时间，田田打包相关的任务请提交到老 CI 系统触发。在新的打包系统投入使用后，老的 Gerrit 和 Jenkins 将下线。

## Links

* Gerrit: 
[https://review.cloud.netease.com](https://review.cloud.netease.com)
* Gerrit 性能监控插件: [https://review.cloud.netease.com/monitoring](https://review.cloud.netease.com/monitoring) 
* Jenkins: [https://jenkins.cloud.netease.com](https://jenkins.cloud.netease.com)
* Zuul status: [https://zuul.cloud.netease.com/](https://zuul.cloud.netease.com/)
* PyPI Server: [https://pypi.cloud.netease.com](https://pypi.cloud.netease.com)