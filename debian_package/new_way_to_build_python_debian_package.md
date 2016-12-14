# Python服务Debian打包新思路

Debian 打包一直是比较冷僻的技术，大部分同学都不会接触到它。
但是我们 Debian 服务器上安装的各种软件服务，都是通过各种打包工具制作出来的安装包部署到服务器上的。

Debian 打包虽然比较烦琐复杂，但是它提供了比较健全的一整套软件部署、安装、升级、维护的流程，
并有一系列与之配套的自动化工具，可以避免人工操作可能出现各种遗漏、错误，特别是在大规模部署时基本不可能人工操作。

我们云计算使用的 Openstack 基础服务，也是通过自己从头制作安装包、上传到 Debian 仓库、并最终通过 puppet 等自动化工具实现服务的部署、更新。

之前我们一直采用 Debian 官方的流程对 Openstack 的 Python 服务打包，但是在几年的实践中发现了各种无法解决的问题，
不得不自己另外实施一套全新的打包方案。

本文主要介绍 Debian 新打包方案的起因、原理、流程。

## 起因

我们以前使用了很长时间的 Debian 社区官方的 Openstack 服务打包方案，中间还尝试过一段时间的 virtualenv 打包方案，各自都有较大的问题，详见下面。

### 社区打包方案

原来我们从 Debian 社区 Openstack 项目打包仓库 fork 出来的自己做一些修改和 backport 然后打包的方案，
所有服务及其依赖的 Python 模块都通过 Debian 的 `deb` 格式安装包安装，这样存在一个主要问题：Debian 官方仓库中的 Python 模块版本更新太慢了。

比如常用的 Python 数据库第三方模块 SQLAlchemy ，在 Python 的官方 PyPI 中已经更新到了 1.1.4 版本，但在 Debian Wheezy 的仓库中仅有 0.7.8 版本，差了4个大版本。

我们在使用 Openstack 服务中发现的一些数据库相关问题，本来是简单的升级 SQLAlchemy 版本就能搞定的，由于这个问题变得很难解决。

这个问题直接导致我们很难升级一些有问题的 Python 模块依赖版本；有些需要的模块甚至根本没有 Debian 安装包，引进新模块、功能比较困难；升级 Openstack 服务的大版本基本不可能，后续也基本不可能从 Debian Wheezy 升级到 Jessie 了，影响非常大。

### virtualenv 打包方案

在发现社区官方的打包方案的严重问题后，我们后面也尝试了一段时间通过 Python virtualenv 虚拟环境打包的方式，
即在一个 virtualenv 虚拟环境中通过 Python pip 工具安装相关 Python 依赖并将整个安装了服务与依赖的 virtualenv 环境打包成 Debian 安装包。

这个方案在后续使用中也发现很多问题，最大的问题还是本质上 virtualenv 并不能真正隔离系统的 Python 环境和自身虚拟环境的 Python 环境，最终导致服务各种诡异错误。

我们这里可以看一个例子

	$ virtualenv test
	$ source test/bin/activate
    >>> import sys
    >>> sys.path
    ['',
    '/home/stanzgy/workspace/test/lib/python2.7',
    '/home/stanzgy/workspace/test/lib/python2.7/plat-linux2',
    '/home/stanzgy/workspace/test/lib/python2.7/lib-tk',
    '/home/stanzgy/workspace/test/lib/python2.7/lib-old',
    '/home/stanzgy/workspace/test/lib/python2.7/lib-dynload',
    '/usr/lib/python2.7',
    '/usr/lib/python2.7/plat-linux2',
    '/usr/lib/python2.7/lib-tk',
    '/home/stanzgy/workspace/test/local/lib/python2.7/site-packages',
    '/home/stanzgy/workspace/test/lib/python2.7/site-packages']
    >>> import json
	>>> json.__file__
	'/usr/lib/python2.7/json/__init__.pyc'
	>>> import _json
	>>> _json.__file__
	'/home/stanzgy/workspace/test/lib/python2.7/lib-dynload/_json.so'

在这个例子中，我们创建了一个虚拟环境 `test` ，并尝试在虚拟环境 import
Python 自带的 `json` 模块，结果发现引用的模块地址事实上是操作系统而不是虚拟环境的。

从 `sys.path` 的结果可以看到，虚拟环境中的 Python import 模块时会尝试先从虚拟环境中的 Python PATH 搜索，然后会尝试从系统的 Python PATH 搜索。如果 import 的模块二次引用其他的 Python 模块实现，则可能导致系统的 Python 模块和虚拟环境中的 Python 模块交叉使用的情况。

在上面的例子中，可以看到 `json` 模块和实现其部分功能的 `_json` 模块分别属于系统和虚拟环境。如果系统和虚拟环境中的 Python 版本、模块版本不一致，则很容易导致服务出现问题，并且最重导致 Python 进程本身崩溃，并且很难调试、查找原因。

virtualenv 打包方案从原理上并不可靠。

## 新 Debian 打包

### 需求

我们 Openstack 服务 Debian 打包在现有基础上的需求主要有三点：
1. 能自由指定、更新 Python 依赖模块版本
2. 不同 Openstack 服务之间的 Python 环境互相隔离
3. Openstack 服务的 Python 环境和系统的 Python 环境隔离

社区的方案三点都不满足，virtualenv 方案只满足第一、二点。

### 原理

新打包的流程比较复杂，但原理用一句话就能描述清楚：
每次打包独立编译 Python ，编译时通过设置 `RPATH` 变量实现隔离效果。

`RPATH` 是 Python 编译时设置的变量，效果是硬编码指定并限制程序运行时动态链接库的的搜索路径，类似 `LD_LIBRARY_PATH`。关于它的详细信息和讨论可以参考 [Wikipedia][1] 和 [Debian Wiki][2]

[1]: https://en.wikipedia.org/wiki/Rpath
[2]: https://wiki.debian.org/RpathIssue

我们每个服务都使用不同的`RPATH`变量编译 Python 后，相当于每个服务都安装在一个独立的 Python 隔离环境里，使用各自独立的运行时动态链接库搜索路径。这样每个服务既可以随意更新修改自己的 Python 依赖模块版本、也避免了之前 virtualenv 方案存在的严重的系统环境隔离问题，解决了上面的三点需求。

下面是一个采用了新打包方案的 Openstack 服务的 Python 环境。

    >>> import sys
    >>> sys.path
    ['',
    '/srv/stack/nova/lib/python27.zip',
    '/srv/stack/nova/lib/python2.7',
    '/srv/stack/nova/lib/python2.7/plat-linux2',
    '/srv/stack/nova/lib/python2.7/lib-tk',
    '/srv/stack/nova/lib/python2.7/lib-old',
    '/srv/stack/nova/lib/python2.7/lib-dynload',
    '/srv/stack/nova/lib/python2.7/site-packages']
    >>> import json
    >>> json.__file__
    '/srv/stack/nova/lib/python2.7/json/__init__.py'
    >>> import _json
    >>> _json.__file__
    '/srv/stack/nova/lib/python2.7/lib-dynload/_json.so'

可以看到它有独立的 Python sys.path 路径、并且不存在和系统的 Python 交叉调用的问题。

### 流程

新 Debian 打包方案的流程，可简单描述为：
1. 指定 `RPATH`，编译、安装 Python
2. 使用新编译的 Python pip 安装依赖
3. 安装 Python 服务到新编译好的 Python 独立环境
4. 将上面创建的整个 Python 独立环境打包
5. 处理其他配置文件、启动脚本等

下面为每个步骤的详细说明

#### 编译 Python

所有项目编译 Python 使用同一个 `build-python.sh` 脚本

    #!/bin/bash

    ...

    export PROJECT_PREFIX=${PROJECT_PREFIX:-/srv/stack}
    export PROJECT_BASE=$PROJECT_PREFIX/$PROJECT
    export PYTHON_FILE=${PYTHON_FILE:-Python-2.7.12.tar.xz}

    # Get python tarball
    CUR_DIR=$PWD
    TEMP_DIR=$(mktemp -d /tmp/pybuild.XXXX)
    cd $TEMP_DIR
    wget $PYTHON_URL/$PYTHON_FILE
    mkdir -p py27 && tar Jxf $PYTHON_FILE -C py27 --strip-components=1
    cd py27

    # Compile python
    DPKG_CPPFLAGS=$(dpkg-buildflags --get CFLAGS)
    DPKG_CFLAGS=$(dpkg-buildflags --get CPPFLAGS)
    DPKG_LDFLAGS=$(dpkg-buildflags --get LDFLAGS)

    CFLAGS="$DPKG_CFLAGS $DPKG_CPPFLAGS" \
    LDFLAGS="$DPKG_LDFLAGS,-rpath=$PROJECT_BASE/lib" \
    ./configure \
        --prefix=$PROJECT_BASE \
        --enable-shared \
        --enable-unicode=ucs2 \
        --with-ensurepip=install \
        --enable-ipv6 \
        --with-dbmliborder=bdb:gdbm \
        --with-fpectl \
        --with-system-expat \
        --with-system-ffi

    make
    make install
    cd $CUR_DIR
    rm -rf $TEMP_CIR

这个脚本会自动下载 Python 2.7.12 ，并指定 `/srv/stack/${PROJECT}` 为每个 Openstack 项目的 $BASE 目录，`/srv/stack/${PROJECT}/lib` 为其 `RPATH` 目录，设定一些编译选项后编译安装 Python 到 $BASE 目录。

#### 安装 Python 依赖

目前新打包的 Python 依赖在上面编译 Python 后通过 pip 安装。

	export REQUIREMENT_FILE=debian/deb-requirements.txt

	# install pip requirements inside the bootstrap system
	$(PROJECT_PREFIX)/$(PROJECT)/bin/pip install \
    	-U -r $(CURDIR)/$(REQUIREMENT_FILE)

每个 Openstack 项目可以在其项目根目录下 `$REQUIREMENT_FILE` 中指定符合 python-pip 格式的 Python 依赖，这个文件通常是通过 `pip freeze` 生成的。

#### 安装服务

在 Python 依赖安装成功后，安装项目本身到独立 Python 环境中。

	# install the project inside the bootstrap system
	$(PROJECT_PREFIX)/$(PROJECT)/bin/python setup.py clean \
    	build --executable "$(PROJECT_PREFIX)/$(PROJECT)/bin/python" install

需要注意的是，在安装项目时需要指定我们自己编译的 Python 路径 ` --executable "$(PROJECT_PREFIX)/$(PROJECT)/bin/python"` 。

#### 其他

完成上面的步骤后，剩下只需要走正常的 Debian 打包流程，处理分包、配置文件等即可。

## 结语

我们使用的这个新 Python 服务 Debian 打包方案，既能享受到 Debian 包管理系统的各种便利和自动化，又能自由使用 Python PyPI 中的各种最新模块，兼顾了两者的优点又避开了各自的缺点。它已经在我们的测试环境稳定运行了几个月，趋于稳定，希望它后续能在我们服务安装部署中更好的服务我们。

关于本文有任何疑问欢迎与我交流。