# 私有pypi server使用说明

私有pypi server主要用来保存需要使用工具`pip`(Python Package Index)安装管理,
但是不适合上传到公共pypi server的python package,
比如我们项目用到的`python-nosclient`, `nt_version`等.
类似我们目前用的debian ppa源.

## 如何使用

我们私有pypi server的地址是 http://114.113.199.8:8080/simple

### pip

在使用pip时指定extra-index-url参数(extra-index-url不会覆盖官方的pypi index-url)

    $ pip install --extra-index-url http://114.113.199.8:8080/simple ${pakcage_name}

或直接在~/.pip/pip.conf中指定

    [global]
    extra-index-url=http://114.113.199.8:8080/simple

## 如何上传package到pypi server

修改~/.pypirc

    [distutils]
    index-servers =
      ntes

    [ntes]
    repository: http://114.113.199.8:8080
    username: nvs
    password: svn

然后在要上传的package目录下

    $ python setup.py sdist upload -r ntes

这样就把私有的python package上传到私有的pypi server了
