# Openstack CI 环境搭建手册

本次 Openstack CI 系统迁移，在前一篇 "CI 系统迁移小结" 文章中已经较详细的说明了系统迁移的背景、动机，以及一些改动的前因后果，本文则主要描述搭建各个 CI 服务的具体步骤。

整个 Openstack CI 环境需要配置下面的服务：

* Gerrit
* Zuul
* Jenkins
* Docker
* Jenkins Job Builder
* PyPI Server
* Nginx

## Gerrit

### 获取 Gerrit 源码

	git clone https://github.com/openstack-infra/gerrit.git
	cd gerrit
	git checkout openstack/2.11.9


### 安装 Buck

	git clone https://github.com/facebook/buck
	cd buck
	git checkout $(cat ../gerrit/.buckversion)
	ant

### 编译 Gerrit

	buck build release

生成 Gerrit 服务的 war 文件在下面的路径

	buck-out/gen/gerrit.war

### 配置 Database

	sudo su postgres
	createuser --username=postgres -RDIElPS gerrit2
	createdb --username=postgres -E UTF-8 -O gerrit2 reviewdb

### 初始化 Gerrit

	java -jar buck-out/gen/gerrit.war init -d $PATH_TO_GERRIT_SITE

程序会提示各个需要配置的配置项让我们填写，一些重要配置比如数据库账号不能填错。
完成以后在 `$PATH_TO_GERRIT_SITE` 目录下会初始化好整个 Gerrit 服务。

我们关心的 Gerrit 配置文件在 `$PATH_TO_GERRIT_SITE/etc` 目录下。

### 启动 Gerrit 服务

	$PATH_TO_GERRIT_SITE/bin/gerrit.sh start
	
其他常用的命令还有

	$PATH_TO_GERRIT_SITE/bin/gerrit.sh stop
	$PATH_TO_GERRIT_SITE/bin/gerrit.sh restart


## Zuul

### 获取 Zuul 源码

	git clone https://github.com/openstack-infra/zuul.git

### 安装 Zuul

	cd zuul
	(optional) virtualenv .venv && source .venv/bin/activate
	pip install -r requirements.txt
	python setup.py install

### 配置 Zuul

Zuul 服务主要有两个配置文件，可参考 Openstack 社区配置：

* zuul.conf
* layout.yaml

zuul.conf 配置 Zuul 服务的基本设置，比如域名、gerrit账号等信息。

layout.yaml 描述 Zuul 服务如何触发 Jenkins 任务，我们每当新增 Jenkins 任务时，也需要在 layout.yaml 增加对应的配置，Jenkins 才能正常工作。

### 启动 Zuul 服务

Zuul 主要有两个服务 zuul-server 和 zuul-merger。

zuul-server 就是连接 Gerrit 与 Jenkins 的主服务，触发 Jenkins 运行测试任务等大部分工作都是 zuul-server 完成的。

zuul-server 会运行一个内置的 gearman-server 与 Jenkins 通信、调度执行任务。

zuul-merger 会测试每个 Gerrit patchset 能否顺利 merge 到当前分支中，并且为 Jenkins slave 提供测试代码的 Git 仓库。

	zuul-server -c zuul.conf -l layout.yaml
	zuul-merger -c zuul.conf

Zuul status page 是一个 HTML website，在 Nginx 中配好即可，不需要启动额外服务。

## Jenkins

### 安装 Jenkins

	cat > /etc/apt/sources.list.d/jenkins.list <<EOF
	deb http://pkg.jenkins-ci.org/debian-stable binary/
	EOF
	
	aptitude update
	aptitude install jenkins

### 安装 gearman-plugin

由于社区的 gearman-plugin 运行模式与我们使用的 Jenkins Docker Plugin 冲突，需要使用我修改过的 gearman-plugin 才能和 Docker Plugin 正确配合工作。

获取 gearman-plugin 源码

	git clone https://github.com/openstack-infra/gearman-plugin.git


我的补丁

	From 1e8b63b6094c527e166879512831b792dbeecc06 Mon Sep 17 00:00:00 2001
	From: stanzgy <stanzgy@gmail.com>
	Date: Wed, 15 Jun 2016 17:07:53 +0800
	Subject: [PATCH 1/1] Force schedule jobs on master
	
	---
	 src/main/java/hudson/plugins/gearman/ExecutorWorkerThread.java | 9 ++++++++-
	 src/main/java/hudson/plugins/gearman/StartJobWorker.java       | 8 +++++++-
	 2 files changed, 15 insertions(+), 2 deletions(-)
	
	diff --git a/src/main/java/hudson/plugins/gearman/ExecutorWorkerThread.java b/src/main/java/hudson/plugins/gearman/ExecutorWorkerThread.java
	index f84a20f..ff8b3b3 100644
	--- a/src/main/java/hudson/plugins/gearman/ExecutorWorkerThread.java
	+++ b/src/main/java/hudson/plugins/gearman/ExecutorWorkerThread.java
	@@ -114,6 +114,13 @@ public class ExecutorWorkerThread extends AbstractWorkerThread{
	
	         HashMap<String,GearmanFunctionFactory> newFunctionMap = new HashMap<String,GearmanFunctionFactory>();
	
	+        String hostName = "";
	+        try {
	+            hostName = computer.getHostName();
	+        } catch (Exception e) {
	+            logger.warn("Exception while getting hostname", e);
	+        }
	+
	         if (!computer.isOffline()) {
	             Node node = computer.getNode();
	             logger.info("---- Executor Worker " + this + " starting register jobs");
	@@ -150,7 +157,7 @@ public class ExecutorWorkerThread extends AbstractWorkerThread{
	
	                     // Register functions iff the current node is in
	                     // the list of nodes for the project's label
	-                    if (projectLabelNodes.contains(node)) {
	+                    if (projectLabelNodes.contains(node) || hostName == masterName) {
	                         String jobFunctionName = "build:" + projectName;
	                         // register without label (i.e. "build:$projectName")
	                         newFunctionMap.put(jobFunctionName, new CustomGearmanFunctionFactory(
	diff --git a/src/main/java/hudson/plugins/gearman/StartJobWorker.java b/src/main/java/hudson/plugins/gearman/StartJobWorker.java
	index 33bfd83..b3d5763 100644
	--- a/src/main/java/hudson/plugins/gearman/StartJobWorker.java
	+++ b/src/main/java/hudson/plugins/gearman/StartJobWorker.java
	@@ -176,7 +176,13 @@ public class StartJobWorker extends AbstractGearmanFunction {
	         Action runNode = new NodeAssignmentAction(runNodeName);
	         // create action for parameters
	         Action params = new NodeParametersAction(buildParams, decodedUniqueId);
	-        Action [] actions = {runNode, params};
	+        Action [] actions;
	+
	+        if (runNodeName == "master") {
	+            actions = new Action[] {params};
	+        } else {
	+            actions = new Action[] {runNode, params};
	+        }
	
	         AvailabilityMonitor availability =
	             GearmanProxy.getInstance().getAvailabilityMonitor(computer);
	--
	2.1.4


应用补丁

	git am -3 docker_plugin.patch


编译 gearman-plugin 插件

	mvn -Dproject-version=0.2.0-netease-2 -DskipTests=true clean package
	
将生成的 hpi 文件安装到 Jenkins Plugin 

### 配置 Jenkins

配置 Jenkins 的重点是要安装好 Git plugin/SCP publisher plugin/Docker plugin 几个插件。由于插件变化较快，这里不做详细描述

### 启动 Jenkins

	systemctl start jenkins


## Docker

### 安装 Docker

	cat > /etc/apt/sources.list.d/docker.list <<EOF
	deb https://apt.dockerproject.org/repo debian-jessie main
	EOF
	
	aptitude update
	aptitude install docker-engine

### 配置 Docker

修改 Docker 启动参数，加入 `-H tcp://$DOCKER_HOST:2375` ，开启 HTTP API 支持 

样例配置

	cat /etc/default/docker
	# Docker Upstart and SysVinit configuration file
	
	#
	# THIS FILE DOES NOT APPLY TO SYSTEMD
	#
	#   Please see the documentation for "systemd drop-ins":
	#   https://docs.docker.com/engine/articles/systemd/
	#
	
	# Customize location of Docker binary (especially for development testing).
	#DOCKER="/usr/local/bin/docker"
	
	# Use DOCKER_OPTS to modify the daemon startup options.
	#DOCKER_OPTS="--dns 8.8.8.8 --dns 8.8.4.4"

	DOCKER_OPTS="-H tcp://$DOCKER_HOST:2375 --dns 223.5.5.5 --dns 114.114.114.114 --registry-mirror=https://docker.mirrors.ustc.edu.cn"

	# If you need Docker to use an HTTP proxy, it can also be specified here.
	#export http_proxy="http://127.0.0.1:3128/"
	
	# This is also a handy place to tweak where Docker's temporary files go.
	#export TMPDIR="/mnt/bigdrive/docker-tmp"

### 初始化 Docker slave image

我将 slave image 分为运行 python 单元测试与运行 devstack tempest 测试两种，他们的 Dockerfile 分别为

ubuntu-trusty
	
	FROM ubuntu:14.04
	MAINTAINER stanzgy <stanzgy@gmail.com>
	
	# Update apt source.list
	RUN echo 'deb http://mirrors.ustc.edu.cn/ubuntu/ trusty main restricted universe multiverse\n\
	deb http://mirrors.ustc.edu.cn/ubuntu/ trusty-security main restricted universe multiverse\n\
	deb http://mirrors.ustc.edu.cn/ubuntu/ trusty-updates main restricted universe multiverse\n\
	deb http://mirrors.ustc.edu.cn/ubuntu/ trusty-proposed main restricted universe multiverse\n\
	deb http://mirrors.ustc.edu.cn/ubuntu/ trusty-backports main restricted universe multiverse\n\
	deb-src http://mirrors.ustc.edu.cn/ubuntu/ trusty main restricted universe multiverse\n\
	deb-src http://mirrors.ustc.edu.cn/ubuntu/ trusty-security main restricted universe multiverse\n\
	deb-src http://mirrors.ustc.edu.cn/ubuntu/ trusty-updates main restricted universe multiverse\n\
	deb-src http://mirrors.ustc.edu.cn/ubuntu/ trusty-proposed main restricted universe multiverse\n\
	deb-src http://mirrors.ustc.edu.cn/ubuntu/ trusty-backports main restricted universe multiverse'\
	> /etc/apt/sources.list
	
	# Set the env variable DEBIAN_FRONTEND to noninteractive
	ENV DEBIAN_FRONTEND noninteractive
	
	# Install necessary packages
	RUN apt-get update && apt-get upgrade -y && apt-get install -y \
	  sudo \
	  bash \
	  curl \
	  git \
	  facter \
	  python \
	  python-dev \
	  openssh-server \
	  openssh-client \
	  openjdk-7-jdk \
	  build-essential \
	  gettext \
	  libffi-dev \
	  libgmp3-dev \
	  libldap2-dev \
	  libmysqlclient-dev \
	  libpq-dev \
	  libsasl2-dev \
	  libsqlite3-dev \
	  libssl-dev \
	  libxml2-dev \
	  libxslt-dev \
	  && rm -rf /var/lib/apt/lists/*
	
	
	# Add jenkins user
	RUN useradd -m -d /home/jenkins -s /bin/bash jenkins && \
	  mkdir -p /home/jenkins/.ssh && \
	  echo 'jenkins:jenkins' | chpasswd && \
	  mkdir -p /etc/sudoers.d && \
	  echo 'jenkins ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/jenkins-sudo
	
	# Add SSH private key
	ADD slave_key /home/jenkins/.ssh/id_rsa
	RUN chown jenkins:jenkins /home/jenkins/.ssh/id_rsa
	
	# SSH login fix. Otherwise user is kicked off after login
	RUN mkdir /var/run/sshd  && \
	  sed 's@session\s*required\s*pam_loginuid.so@session optional pam_loginuid.so@g' -i /etc/pam.d/sshd
	
	# Disable SSH host key checking
	RUN sed -i "s/^.*StrictHostKeyChecking.*$/StrictHostKeyChecking no/" /etc/ssh/ssh_config
	
	# Setup python environment
	ADD get-pip.py /root/get-pip.py
	RUN echo '[global]\n\
	index-url = https://pypi.cloud.netease.com/jenkins/dev/+simple'\
	> /etc/pip.conf && \
	echo '[easy_install]\n\
	index_url = https://pypi.cloud.netease.com/jenkins/dev/+simple'\
	> /root/.pydistutils.cfg && \
	  python /root/get-pip.py && \
	  pip install --upgrade 'pip<8.1.2' tox virtualenv ndg-httpsclient pyopenssl pyasn1
	
	# NOTE(stanzgy): The jenkins docker plugin will run sshd by default.
	EXPOSE 22
	#CMD ["/usr/sbin/sshd", "-D"]

	
ubuntu-trusty-dsvm

	FROM ubuntu:14.04
	MAINTAINER stanzgy <stanzgy@gmail.com>
	
	# Update apt source.list
	RUN echo 'deb http://mirrors.ustc.edu.cn/ubuntu/ trusty main restricted universe multiverse\n\
	deb http://mirrors.ustc.edu.cn/ubuntu/ trusty-security main restricted universe multiverse\n\
	deb http://mirrors.ustc.edu.cn/ubuntu/ trusty-updates main restricted universe multiverse\n\
	deb http://mirrors.ustc.edu.cn/ubuntu/ trusty-proposed main restricted universe multiverse\n\
	deb http://mirrors.ustc.edu.cn/ubuntu/ trusty-backports main restricted universe multiverse\n\
	deb-src http://mirrors.ustc.edu.cn/ubuntu/ trusty main restricted universe multiverse\n\
	deb-src http://mirrors.ustc.edu.cn/ubuntu/ trusty-security main restricted universe multiverse\n\
	deb-src http://mirrors.ustc.edu.cn/ubuntu/ trusty-updates main restricted universe multiverse\n\
	deb-src http://mirrors.ustc.edu.cn/ubuntu/ trusty-proposed main restricted universe multiverse\n\
	deb-src http://mirrors.ustc.edu.cn/ubuntu/ trusty-backports main restricted universe multiverse'\
	> /etc/apt/sources.list
	
	# Set the env variable DEBIAN_FRONTEND to noninteractive
	ENV DEBIAN_FRONTEND noninteractive
	
	
	# Install necessary packages
	RUN apt-get update && apt-get upgrade -y && apt-get install -y \
	  sudo \
	  dbus \
	  bash \
	  curl \
	  git \
	  facter \
	  python \
	  python-dev \
	  openssh-server \
	  openssh-client \
	  openjdk-7-jdk \
	  build-essential \
	  gettext \
	  libffi-dev \
	  libgmp3-dev \
	  libldap2-dev \
	  libmysqlclient-dev \
	  libpq-dev \
	  libsasl2-dev \
	  libsqlite3-dev \
	  libssl-dev \
	  libxml2-dev \
	  libxslt-dev \
	  bridge-utils \
	  iptables \
	  openvswitch-switch \
	  libvirt0 \
	  libvirt-bin \
	  python-libvirt \
	  qemu-kvm \
	  && rm -rf /var/lib/apt/lists/*
	
	# Add jenkins user
	RUN mkdir /var/run/dbus && \
	  useradd -m -d /home/jenkins -s /bin/bash jenkins && \
	  mkdir -p /home/jenkins/.ssh && \
	  echo 'jenkins:jenkins' | chpasswd && \
	  mkdir -p /etc/sudoers.d && \
	  echo 'jenkins ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/jenkins-sudo
	
	# Add SSH private key
	ADD slave_key /home/jenkins/.ssh/id_rsa
	RUN chown jenkins:jenkins /home/jenkins/.ssh/id_rsa
	
	# SSH login fix. Otherwise user is kicked off after login
	RUN mkdir /var/run/sshd  && \
	  sed 's@session\s*required\s*pam_loginuid.so@session optional pam_loginuid.so@g' -i /etc/pam.d/sshd
	
	# Disable SSH host key checking
	RUN sed -i "s/^.*StrictHostKeyChecking.*$/StrictHostKeyChecking no/" /etc/ssh/ssh_config
	
	# Setup python environment
	ADD get-pip.py /root/get-pip.py
	RUN echo '[global]\n\
	index-url = https://pypi.cloud.netease.com/jenkins/dev/+simple'\
	> /etc/pip.conf && \
	echo '[easy_install]\n\
	index_url = https://pypi.cloud.netease.com/jenkins/dev/+simple'\
	> /root/.pydistutils.cfg && \
	  python /root/get-pip.py && \
	  pip install --upgrade 'pip<8.1.2' tox virtualenv ndg-httpsclient pyopenssl pyasn1
	
	# NOTE(stanzgy): The jenkins docker plugin will run sshd by default.
	EXPOSE 22
	#CMD ["/usr/sbin/sshd", "-D"]

构建 Docker image

	cd ubuntu-trusty
	docker build -t ci/ubuntu-trusty .
	
	cd ../ubuntu-trusty-dsvm
	docker build -t ci/ubuntu-trusty-dsvm .
	
在 Jenkins 中配置 python 单元测试任务使用 ubuntu-trusty 镜像，devstack tempest 测试使用 ubuntu-trusty-dsvm 镜像。


## Jenkins Job Builder

### 获取 Jenkins Job Builder 源码

	git clone https://github.com/openstack-infra/jenkins-job-builder.git
	cd jenkins-job-builder
	git checkout 1.6.1

### 安装 Jenkins Job Builder

	(optional) virtualenv .venv && source .venv/bin/activate
	pip install -r requirements.txt
	python setup.py install

### 配置 Jenkins Job Builder

Jenkins Job Builder 的配置文件分为两种，一种是其本身的配置文件
`jenkins_jobs.ini`，包含了 Jenkins 账号等信息。另一种是 Jenkins 任务描述文件，需要我们自己编写。

例如，下面的 test.yaml 文件就描述了一个名为 node-test 的任务，他会在 Jenkins slave 上执行一条 `echo 'ok'` 的 shell 命令。

	cat test.yaml
	- job:
	    name: node-test
	
	    parameters:
	      - label:
	          name: NODE
	          description: Node to test
	
	    builders:
	      - shell: 'echo ok'

### 将测试任务导入 Jenkins

假设我们的 Jenkins 测试任务描述文件都放在 jenkins_jobs 文件夹下

	cd jenkins_jobs
	jenkins-jobs --conf jenkins_jobs.ini update .
	
即可将所有测试任务导入 Jenkins


## PyPI Server

我们的 PyPI Server 使用 devpi 搭建，使用其他类似功能的服务也是可以的。

这里以 devpi 做说明。

### 安装 devpi

	(optional) virtualenv .venv && source .venv/bin/activate
	pip install devpi-client devpi-server devpi-web

### 启动 devpi-server

	devpi-server --start --port 4040 --serverdir $PATH_TO_DEVPI
	
### 初始化 devpi

	devpi use http://localhost:4040
	devpi login root --password ''
	devpi user -m root password=$ROOT_PASSWD
	devpi logoff
	
	devpi user -c jenkins password=$JK_PASSWD
	devpi login jenkins
	
	devpi index -c dev bases=root/pypi

## Nginx

Nginx 的配置很多，上面各个服务或多或少都有一些 Nginx 配置，包括上面没有提到的日志、文档服务器等。

由于 Nginx 配置都是静态文件，这里不在过多赘述。

所有的配置文件都维护在 `https://g.hz.netease.com/openstack-netease/ci-config.git` 项目下，可以直接参考其中的配置内容。