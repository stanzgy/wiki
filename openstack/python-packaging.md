# Python包管理工具小结

- [Python包管理工具小结](#)
    - [setuptools](#)
        - [pbr](#)
    - [pip](#)
    - [virtualenv](#)
    - [wheel](#)
    - [Linux distros](#)
    - [总结](#)
        - [直接源码安装](#)
        - [virtualenv+源码直接安装](#)
        - [virtualenv+源码打包发布](#)
        - [使用Linux发行版本身的安装包](#)
        - [virtualenv+源码+Linux发行版安装包打包发布](#)
    - [References](#)


作为一名接触Python有一段时间的初学者，越来越体会到Python的方便之处，它使人能更
多的关注业务本身的逻辑，而不用太纠结语言层面的技巧与细节。然而，随着服务的规模
变得越来越大，如何方便快速地制作与发布一个Python软件包则越来越成为一个让人头疼
地问题，特别是像Openstack这种相对复杂、各种依赖也很多的Python项目，到目前也没有
发现特别完美的解决方案。这里将尝试对Python的包管理工具做一个小结，为将来更好的
解决这个问题提供思路。

## setuptools

setuptools[1] 是Python提供的最基本的包管理工具，后面提到的其他更高层次的Python
包管理工具都是在它的基础上实现的。曾经Python还有其他各种各样类似功能的包管理工
具，比如distribute、distutils等等，对有选择困难症的同学非常不友好，不过目前绝大
多数Python项目都开始统一使用setuptools，其他工具已经被慢慢合并/遗弃了。

setuptools的常见使用方式是在Python项目中新建一个`setup.py`:

    from setuptools import setup, find_packages
    setup(
        name = "HelloWorld",
        version = "0.1",
        packages = find_packages("helloworld"),
    )

在这个文件中通过setuptools提供的各种关键字描述项目的元信息，包括名字、版本、
文件等等。写好后就可以通过`python setup.py [command]`执行setuptools提供的各种子
命令，比如编译、安装、测试、制作各种格式的软件安装包等等。

当你拿到一份Python项目的源代码，如果它已经写好了`setup.py`，你就可以方便的通过
setuptools将它安装到系统：

    $ python setup.py install

目前，使用setuptools制作的软件安装包常见格式有：

source distro

    $ python setup.py sdist

binary distro: egg

    $ python setup.py bdist_egg

binary distro: wheel

    $ python setup.py bdist_wheel

以openstack/nova为例，对应的三种格式软件安装包分别为：

    nova-12.0.0.0b2.dev251.tar.gz
    nova-12.0.0.0b2.dev251-py2.7.egg
    nova-12.0.0.0b2.dev251-py2.py3-none-any.whl

sdist和wheel格式的安装包都可以使用最常见的pip命令安装，egg还只能通过
比较老得easy\_install命令安装。

    $ pip install nova-12.0.0.0b2.dev251.tar.gz
    $ easy_install nova-12.0.0.0b2.dev251-py2.7.egg
    $ pip install nova-12.0.0.0b2.dev251-py2.py3-none-any.whl

### pbr

从上面的例子里可以看到，`setup.py`本质上还是一个Python脚本，有诸多语法的限制。
当Python项目比较庞大和复杂时，`setup.py`就会变的非常难写与维护。
在Openstack中，新建了一个pbr[2] 项目来处理这个问题。通过使用pbr项目提供的维护与
`setup.py`对应的`setup.cfg`文本配置文件来自动生成`setup.py`从而解决这个问题。

比如上面的例子，切换到pbr就会变成这样：

setup.py

    from setuptools import setup

    setup(
        setup_requires=['pbr'],
        pbr=True,
    )

setup.cfg

    [metadata]
    name = HelloWorld
    version = 0.1

    [files]
    packages =
        helloworld

当然，实际工程中，`setup.cfg`文件的内容会比这个例子复杂很多。

## pip

setuptools解决了Python软件包的制作问题，那如何分发它们呢？Python本身提供了PyPI
基础设施与相应的pip[3] 命令行工具来做这件事。

你可以将生成好的软件包通过

    $ python setup.py upload

通过网络上传到PyPI。
上传成功后，其他人就可以直接通过pip命令安装你上传的软件包：

    $ pip install ${package_name}

## virtualenv

提到了pip，就不得不提一下virtualenv[4] ，这两个工具经常是在一起搭配使用的。

virtualenv的功能有点像Linux系统下的chroot命令，运行

    $ virtualenv .venv

后，他会将最小安装一个Python运行环境到当前目录的.venv文件夹，内容包括.venv/bin
下的python、pip命令，.venv/lib下的python库目录等等。当你使用

    $ . .venv/bin/activate

激活刚刚创建出来的这个virtualenv环境后，你执行的python相关的二进制命令其实都是
.venv/bin目录下（而不是系统路径下的），安装/引用的python库也都是在.venv/lib目录
下的。

这样子，使用virtualenv后可以让每个Python项目使用独立的Python运行环境，不会产生
几个项目间依赖冲突、污染操作系统Python库的问题。

## wheel

wheel[5] 是众多Python软件安装包格式的一种，这里专门要把它单独拿出来说是因为它的
优点和好处确实很多，是Python官方推荐的新一代Python项目发布格式，个人也非常希望
目前所有PyPI上项目都能尽快提供wheel格式的安装包。新版本的pip已经可以直接从PyPI
上下载安装wheel格式的安装包了。

和老的egg[6] 格式相比，wheel能让人直观感受到的先进之处：

* 不包含.pyc文件（预编译的.pyc文件偶尔会导致各种奇怪的问题，我们更希望他能在
  每次安装时在本地生成更新），当项目是纯Python项目时wheel和source distro基本
  没有什么区别。
* 官方支持，pip命令可以直接安装wheel，但是不能处理egg。
* 更丰富的软件包元信息，甚至包中的每个文件都有版本记录。
* 更细致的包命名规则

和egg格式一样，wheel拥有的优点：

* 单安装文件。你是希望从源码文件夹下执行一系列命令编译安装一个Python项目，还是
  直接通过一个文件安装呢？
* 依赖处理。安装程序会帮你自动安装相关依赖的Python库。
* 二进制发布格式。当项目包含需要编译生成的extension时，可以将编译好的
  .so/.dylib/.dll等动态链接库直接一并打包，实现“一次编译，到处运行”（在相同的
  平台上）的效果，而不是每次都在终端重新编译生成。这点在大规模服务器上批量部署
  Python程序项目非常重要。可惜的是，目前wheel仅在windows和mac os平台支持这个特
  性，Linux平台由于各种原因还不支持，希望PyPA能尽快解决吧。

这里有一份官方文档详细比较了wheel和egg: [7]

## Linux distros

上面总结了Python打包社区PyPA（Python Packaging Authority）本身提供的各种工具，
但是在各个Linux发行版中，出于和系统集成（比如各种配置文件、脚本的管理）、加强
包的管理（PyPI上的软件包都是开发者自行自由上传的）等等原因，很多Linux发行版通常
也会制作维护各自平台上专有的Python项目安装包，比如Debian系的.deb格式安装包
等等。

由于各个Linux发行版的管理风格、系统路径、甚至默认的Python版本等等都大相径庭，
所以各种安装包的规格、内容也差别很大。

常见的Linux发行版里，Debian系的.deb格式和Redhat系的.rpm格式安装包属于比较严格和
保守的一派。它们通常会做严格的版本、依赖控制；维护项目相关的服务配置文件、
SysVinit/systemd、syslog、logrotate脚本等等；安装包会包含任何需要编译的二进制
文件（如.pyc文件、各种需要编译的c extension等），不需要在安装时进行本地编译；
使用Python2作为Python的默认版本，甚至在某些较老的Debian、CentOS的发行版中，
还在使用Python2.6/2.3。

而像ArchLinux这样比较激进的Linux发行版，安装包的管理就非常奔放。安装包通常只是
帮你指定一下依赖、任何软件和库都用最新版本（非常令人讨厌的，Python也默认使用
Python3）、Python项目的`PKGBUILD`里package()通常就一行
“python2 setup.py install ...”。

## 总结

上面基本把最近自己接触到的和Python相关的包管理工具介绍了一遍，而在实际操作中去
在服务器上批量发布部署Python项目，利用上面提到的这些工具，能想到的通常有下面
几种方式。

每次到这个时候，真的是非常羡慕Java/Go这些不用纠结这种问题的编程语言。

### 直接源码安装

适合在自己的开发调试机器上瞎搞的时候使用。

由于Python/pip本身的包管理功能比较弱，直接在服务器上用这种方法会给服务器的系统
安装一系列不可控制的Python依赖，并且可能破坏其他Python程序的依赖，导致其他
Python程序无法正常运行。装的时候方便，当你碰到问题想清理的时候可就蛋疼了，相信
维护服务器的SA也不太可能会让你这样做的。

### virtualenv+源码直接安装

比较靠谱的办法，但是只适合单机部署。当服务器数量众多，如何实现批量部署发布就是
个困难的问题了。

### virtualenv+源码打包发布

在virtualenv环境里安装好所有的Python文件，把整个virtualenv目录打包成一个单独的
tar包。发布时直接将这个tar包拷贝到目标服务器上解压安装。

目前一些知名的Openstack厂商，如Rackspace，就是通过这种方式在服务器上部署Python
项目的。

然而，这样做的缺点是，还需要做很多额外的工作去维护服务的配置文件、脚本、系统上
非Python相关的软件依赖（如kvm、libvirt、openvswitch）等等。

类似方法还有自建私有PyPI源，提前将Python项目打包上传到自建的PyPI源里。然后在
服务器上的virtualenv环境里通过自建的PyPI源安装。

### 使用Linux发行版本身的安装包

比如，Debian系的.deb安装包、Redhat系的.rpm安装包。它们能比较好的维护各种服务
配置文件、脚本，以及像kvm这样的非Python相关的依赖，我们目前也在使用这种方式。

然而，它也有很多问题，以我比较熟悉的Debian发行版为例：

* 所有Python项目共用一套Python依赖。当有的Python项目想升级某个第三方库时，会
  因为破坏其他Python项目的依赖版本而无法实现。
* 官方提供的软件版本比较旧。对于一个追求稳定的Linux发行版，这么做本身并无可厚
  非，但是有时候当你就是想用新一点版本时，就会感觉非常无奈，基本都要自己动手制
  作安装包。而且由于依赖的关系，可能你为了给一个软件制作新版本的安装包，需要再
  为其他10个依赖的项目制作新版本的安装包，然后又要为这10个依赖的项目再制作30个
  依赖的依赖的新版本安装包。。。
* 学习成本比较高。Debian的安装包制作方法，除了官方十年前写的几份文档外，网上
  其他能找到的资料很少，学起来比较费力。Debian作为一个装机量较大的Linux发行版都
  是这样，相信其他发行版也是类似的情况。

### virtualenv+源码+Linux发行版安装包打包发布

在virtualenv环境里安装好所有的Python文件，然后将整个virtualenv目录打包到单独的
一个Linux发行版的安装包内，同时在这个安装包内设置好相关的配置文件、脚本、系统
依赖。

理论上这种方法综合了上面两种方式的优点，目前正在调研中，然而相信实践过程中肯定
会踩到不少坑。

spotify公布了他们在Debian下使用这种方式解决Python项目发布部署问题的工具
`dh-virtualenv`[8] ，感兴趣的同学可以参考一下。

## References

[1]: http://pythonhosted.org/setuptools/index.html
[2]: http://docs.openstack.org/developer/pbr
[3]: https://pip.pypa.io/en/stable
[4]: https://virtualenv.pypa.io/en/latest
[5]: https://www.python.org/dev/peps/pep-0427
[6]: http://peak.telecommunity.com/DevCenter/PythonEggs
[7]: https://packaging.python.org/en/latest/wheel_egg.html
[8]: https://github.com/spotify/dh-virtualenv

* [Python Packaging User Guide](https://packaging.python.org/en/latest/index.html)
* [Python on Wheels](http://lucumr.pocoo.org/2014/1/27/python-on-wheels)
* [Python 包管理工具解惑](http://zengrong.net/post/2169.htm)
