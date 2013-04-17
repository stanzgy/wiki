使用Gerrit的topic机制提交代码
==========================

参考文档: http://wiki.openstack.org/GerritWorkflow


## 为什么要使用topic?

目前我们向gerrit中提交代码时, 如果本地有多次commit一次性提交, 会导致提交到gerrit上的review
之间有依赖关系(Dependencies), 如果依赖的其他review出现了改动(比如上传了新的patch set, 或者
被abandon等等), 都需要手工rebase才能将review的代码merge到trunk分支中, 十分不方便.

如果使用gerrit的topic-branch机制, 则能保持每个review之间没有依赖关系, 保持独立, 每个review
的改动不会影响到其他review.

另外参考社区的convention, 每个topic的命名必须对应launchpad的一个bug report或者
blueprint等等, 使用类似"bug/1074054"这样的topic命名, 这样进行code review的人员可以根据
topic名字方便的在launchpad上找到和这次commit相关的信息.


## howto使用gerrit的topic机制

下面以我们scm上的nova仓库为例

0. 安装git-review

    stanzgy % pip install review

1. 获得代码

    stanzgy % git clone ssh://hzzhanggy@scm.hz.netease.com:2222/openstack/nova

1.5 添加scm到git remote(如果代码是从官方的github clone的)

    stanzgy % git remote add scm ssh://hzzhanggy@scm.hz.netease.com:2222/openstack/nova

2. 设置.gitreview

在nova仓库根目录下编辑文件.gitreview.
host和project对应scm上的仓库地址, defaultbranch为我们的开发分支, defaultremote为前面添加
的scm或者origin.

    [gerrit]
    host=scm.hz.netease.com
    port=29418
    project=openstack/nova.git
    defaultbranch=folsom-ntse
    defaultremote=scm
    defaultrebase=0

3. 在提交review时使用topic机制

在JIRA上先建立相应的任务说明

更新remote

    stanzgy % git remote update

checkout到我们的开发分支

    stanzgy % git checkout -b folsom-ntse scm/folsom-ntse   #第一次checkout

或者

    stanzgy % git checkout folsom-ntse

新建一个topic

    stanzgy % git checkout -b CLOUD-2600

干活...
...
干完了...

本地commit

    stanzgy % git commit -a

提交review

    stanzgy % git review

这时候可以在gerrit review页面上发现topic信息了
https://scm.hz.netease.com/#/c/412/
同时烦人的Dependencies也都没有了
