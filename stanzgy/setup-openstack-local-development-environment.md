This is a guide for setting up openstack essex local development environment
based on my experience of installing openstack from source code.

Feel free to corrent any mistakes found in this page.


# MySQL installation

## Get mysql

(Archlinux)
> stanzgy % sudo pacman -Sy mysql mysql-python

(Debian/Ubuntu)
> stanzgy % sudo aptitude install mysql-server python-mysqldb


## Create database

> <pre>
  mysql> CREATE DATABASE nova;
  mysql> CREATE DATABASE glance;
  mysql> CREATE DATABASE keystone;
  mysql> CREATE USER 'nova'@'localhost' IDENTIFIED BY 'nova';
  mysql> CREATE USER 'glance'@'localhost' IDENTIFIED BY 'glance';
  mysql> CREATE USER 'keystone'@'localhost' IDENTIFIED BY 'keystone';
  mysql> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost';
  mysql> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost';
  mysql> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost';</pre>


# Keystone installation

## Get source tarball

https://launchpad.net/keystone/+download or some other places


## Install keystone

> stanzgy % ./run_tests.sh

> stanzgy % python setup.py build && sudo python setup.py install


## Import keystone database

> stanzgy % sudo keystone-manage db_sync


## Modify config file

keystone.conf(mod):

    verbose = True
    debug = True

    admin_token = ADMIN
    connection = mysql://keystone:keystone@localhost/keystone
    driver = keystone.catalog.backends.sql.Catalog


## Run keystone daemon

> stanzgy % sudo keystone-all


## Create tenant

> stanzgy % keystone --token ADMIN --endpoint http://localhost:35357/v2.0 tenant-create --name service


    +-------------+----------------------------------+
    |   Property  |              Value               |
    +-------------+----------------------------------+
    | description |                                  |
    |   enabled   |               True               |
    |      id     | b46995d5d4234fecb9f125ffb750751b |
    |     name    |             service              |
    +-------------+----------------------------------+


## Create users


> stanzgy % keystone --token ADMIN --endpoint http://localhost:35357/v2.0 user-create --name admin --tenant_id b46995d5d4234fecb9f125ffb750751b --pass admin

    +----------+-------------------------------------------------------------------------------------------------------------------------+
    | Property |                                                          Value                                                          |
    +----------+-------------------------------------------------------------------------------------------------------------------------+
    |  email   |                                                                                                                         |
    | enabled  |                                                           True                                                          |
    |    id    |                                             bb19381114174499907c8666efc8e194                                            |
    |   name   |                                                          admin                                                          |
    | password | $6$rounds=40000$DAPMOSD86ud/FwmZ$L4OeLc9zwjnauM6ABOJ2MegVB2SH.LuAoTpd0fibgVU51sSyAFo0r7Uu87HBueKWH/N5mUcKTmEI2gpeqXADU0 |
    | tenantId |                                             b46995d5d4234fecb9f125ffb750751b                                            |
    +----------+-------------------------------------------------------------------------------------------------------------------------+

> stanzgy % keystone --token ADMIN --endpoint http://localhost:35357/v2.0 user-create --name nova --tenant_id b46995d5d4234fecb9f125ffb750751b --pass nova

> stanzgy % keystone --token ADMIN --endpoint http://localhost:35357/v2.0 user-create --name glance --tenant_id b46995d5d4234fecb9f125ffb750751b --pass glance

> stanzgy % keystone --token ADMIN --endpoint http://localhost:35357/v2.0 user-create --name cinder --tenant_id b46995d5d4234fecb9f125ffb750751b --pass cinder

> stanzgy % keystone --token ADMIN --endpoint http://localhost:35357/v2.0 user-create --name user --tenant_id b46995d5d4234fecb9f125ffb750751b --pass user


## Create role

> stanzgy % keystone --token ADMIN --endpoint http://localhost:35357/v2.0 role-create --name admin

    +----------+----------------------------------+
    | Property |              Value               |
    +----------+----------------------------------+
    |    id    | c60f08bf89b44fe7a309192e6a6ecf74 |
    |   name   |              admin               |
    +----------+----------------------------------+

> stanzgy % keystone --token ADMIN --endpoint http://localhost:35357/v2.0 role-create --name user


## Add user role

#### add admin to admin

> stanzgy % keystone --token ADMIN --endpoint http://localhost:35357/v2.0 user-role-add  --user-id bb19381114174499907c8666efc8e194 --role-id c60f08bf89b44fe7a309192e6a6ecf74  --tenant_id b46995d5d4234fecb9f125ffb750751b

#### add nova to admin

> stanzgy % keystone --token ADMIN --endpoint http://localhost:35357/v2.0 user-role-add  --user-id 8705a2bdb8074c878c3b86e5dbb27475 --role-id c60f08bf89b44fe7a309192e6a6ecf74  --tenant_id b46995d5d4234fecb9f125ffb750751b

#### add glance to admin

> stanzgy % keystone --token ADMIN --endpoint http://localhost:35357/v2.0 user-role-add  --user-id 32a529250d034e1c8e5e3192c7b2e777 --role-id c60f08bf89b44fe7a309192e6a6ecf74  --tenant_id b46995d5d4234fecb9f125ffb750751b

#### add keystone to admin

> stanzgy % keystone --token ADMIN --endpoint http://localhost:35357/v2.0 user-role-add  --user-id c4ea91b355b649e0a64b0cb3c465ba6c --role-id c60f08bf89b44fe7a309192e6a6ecf74  --tenant_id b46995d5d4234fecb9f125ffb750751b

#### add user to user

> stanzgy % keystone --token ADMIN --endpoint http://localhost:35357/v2.0 user-role-add  --user-id ceca7bb928f94aa1920f5a9b5dc21098 --role-id f23f2f4c17474a4888baaa066867b6c8  --tenant_id b46995d5d4234fecb9f125ffb750751b


## Create service

> stanzgy % keystone --token ADMIN --endpoint http://localhost:35357/v2.0 service-create --name nova --type compute

    +-------------+----------------------------------+
    |   Property  |              Value               |
    +-------------+----------------------------------+
    | description |                                  |
    |      id     | 6d947dfdc3504349a1456e361efc8845 |
    |     name    |               nova               |
    |     type    |             compute              |
    +-------------+----------------------------------+

> stanzgy % keystone --token ADMIN --endpoint http://localhost:35357/v2.0 service-create --name glance --type image

> stanzgy % keystone --token ADMIN --endpoint http://localhost:35357/v2.0 service-create --name keystone --type identity

> stanzgy % keystone --token ADMIN --endpoint http://localhost:35357/v2.0 service-create --name cinder --type volume


## Create service endpoint

#### create nova endpoint

> <pre>stanzgy % keystone --token ADMIN --endpoint http://localhost:35357/v2.0 endpoint-create \
                          --service-id 6d947dfdc3504349a1456e361efc8845 \
                          --publicurl http://localhost:8774/v1.1/$\(tenant_id\)s \
                          --internalurl http://localhost:8774/v1.1/$\(tenant_id\)s \
                          --adminurl http://localhost:8774/v1.1/$\(tenant_id\)s</pre>

    +-------------+------------------------------------------+
    |   Property  |                  Value                   |
    +-------------+------------------------------------------+
    |   adminurl  | http://localhost:8774/v1.1/$(tenant_id)s |
    |      id     |     c9d8c4d8914d4ec5b82e7e224c2596d6     |
    | internalurl | http://localhost:8774/v1.1/$(tenant_id)s |
    |  publicurl  | http://localhost:8774/v1.1/$(tenant_id)s |
    |    region   |                regionOne                 |
    |  service_id |     6d947dfdc3504349a1456e361efc8845     |
    +-------------+------------------------------------------+

#### create glance endpoint

> <pre>stanzgy % keystone --token ADMIN --endpoint http://localhost:35357/v2.0 endpoint-create \
                          --service-id 7c91d189f83d4a0dbd8f17146ab9edb0 \
                          --publicurl http://localhost:9292/v1 \
                          --internalurl http://localhost:9292/v1 \
                          --adminurl http://localhost:9292/v1</pre>

#### create keystone endpoint

> <pre>stanzgy % keystone --token ADMIN --endpoint http://localhost:35357/v2.0 endpoint-create \
                          --service-id ec161bcbbd9c44169e67f7e0f1dd8eac \
                          --publicurl http://localhost:5000/v2.0 \
                          --internalurl http://localhost:5000/v2.0 \
                          --adminurl http://localhost:35357/v2.0</pre>

#### create cinder endpoint

> <pre>stanzgy % keystone --token ADMIN --endpoint http://localhost:35357/v2.0 endpoint-create \
                          --service-id 064426ec12244127a7b2e49119517e37 \
                          --publicurl http://localhost:8776/v1/$\(tenant_id\)s \
                          --internalurl http://localhost:8776/v1/$\(tenant_id\)s \
                          --adminurl http://localhost:8776/v1/$\(tenant_id\)s</pre>


# Glance installation

## Get source tarball

https://launchpad.net/glance/+download or some other places


## Install glance

> stanzgy % ./run_tests.sh

> stanzgy % python setup.py && sudo python setup.py install


## Import glance database

> stanzgy % sudo glance-manage db_sync


## Modify config file

glance-api.conf(add/mod):

    debug = True
    verbose = True

    [paste_deploy]
    flavor = keystone

glance-api-paste.ini(mod):

    [filter:authtoken]
    admin_tenant_name = service
    admin_user = glance
    admin_password = glance

glance-registry.conf(add/mod):

    debug = True
    verbose = True
    sql_connection = mysql://glance:glance@localhost/glance

    [paste_deploy]
    flavor = keystone

glance-registry-paste.ini(mod):

    [filter:authtoken]
    admin_tenant_name = service
    admin_user = glance
    admin_password = glance


## Run glance daemons

> stanzgy % sudo glance-api

> stanzgy % sudo glance-registry


## Upload a image

### via glance client

> <pre>stanzgy % glance -I admin -K admin -T service -N http://localhost:5000/v2.0 -S keystone \
                    add name="debian-6.0.4-amd64-standard" is_public=true container_format=bare disk_format=qcow2 &#60; \
                    /home/stanzgy/workspace/images/debian-6.0.4-amd64-standard.qcow2</pre>

    Uploading image 'debian-6.0.4-amd64-standard'
    ===============================================================[100%] 113.683414M/s, ETA  0h  0m  0s
    Added new image with ID: c79695d5-9109-489b-9434-5252834a14f6


### via glance api

#### get token

> <pre>stanzgy % curl -X 'POST'  -v http://localhost:5000/v2.0/tokens -H 'Content-type: application/json' \
                    -d '{"auth":{"passwordCredentials":{"username": "admin", "password":"admin"}, "tenantName":"service"}}'  | python -mjson.tool</pre>

    ...
    "token": {
        "expires": "2012-08-21T12:20:57Z",
        "id": "59f6509ca0e643b68a3fe89efcd9b5de",
        "tenant": {
            "description": null,
            "enabled": true,
            "id": "b46995d5d4234fecb9f125ffb750751b",
            "name": "service"
        }
    },
    ...

#### upload image

> <pre>stanzgy % curl http://localhost:9292/v1/images -X POST \
                    -H 'x-auth-token:59f6509ca0e643b68a3fe89efcd9b5de' \
                    -H 'x-image-meta-name: debian-6.0.4-amd64-standard' \
                    -H 'x-image-meta-is-public: true' \
                    -H 'content-type:application/octet-stream' \
                    -H 'x-image-meta-disk-format:qcow2' \
                    -H 'x-image-meta-container-format:bare' \
                    -T /home/stanzgy/workspace/images/debian-6.0.4-amd64-standard.qcow2 | python -mjson.tool</pre>

    {
        "image": {
            "checksum": "67ebd94e6f164293a3768200810657aa",
            "container_format": "bare",
            "created_at": "2012-08-20T12:29:15",
            "deleted": false,
            "deleted_at": null,
            "disk_format": "qcow2",
            "id": "14ab1a42-f605-4504-a88d-bb0eac0a3ffd",
            "is_public": true,
            "min_disk": 0,
            "min_ram": 0,
            "name": "debian-6.0.4-amd64-standard",
            "owner": "b46995d5d4234fecb9f125ffb750751b",
            "properties": {},
            "protected": false,
            "size": 768868352,
            "status": "active",
            "updated_at": "2012-08-20T12:29:25"
        }
    }


# Nova installation

## Get source tarball

https://launchpad.net/nova/+download or some other places


## Install nova

> stanzgy % ./run_tests.sh

> stanzgy % python setup.py && sudo python setup.py install


## Modify config file

api-paste.ini(mod):

    [filter:authtoken]
    admin_tenant_name = service
    admin_user = nova
    admin_password = nova

nova.conf(add/mod):

    debug = True
    verbose = True

    instances_path = /home/stanzgy/lib/nova/instances
    sql_connection = mysql://nova:nova@localhost/nova

    multi_host = False
    network_manager = nova.network.manager.FlatDHCPManager
    fixed_range = 10.233.0.0/22

    auth_strategy = keystone
    connection_type = libvirt
    libvirt_type = kvm

    vncserver_listen=0.0.0.0

sample [[nova.conf|http://netease.stanzgy.org/~stanzgy/conf/nova.conf]]


## Import nova database

> stanzgy % sudo nova-manage db sync


## Create network in database

> stanzgy % sudo nova-manage network create private 10.233.0.0/22 1 1024 --bridge=br100


## Run nova daemons

> stanzgy % sudo nova-compute

> stanzgy % sudo nova-network

> stanzgy % sudo nova-scheduler

> stanzgy % sudo nova-api


# Horizon installation

## Get source tarball

https://launchpad.net/horizon/+download or some other places


## Install horizon

> stanzgy % ./run_tests.sh

> stanzgy % python setup.py && sudo python setup.py install


## Run horizon daemon

> stanzgy % sudo ./run_tests.sh --runserver


# Troubleshooting

## `anyjson` version conflicts

Some version of glance and keystone depands on `anyjson` 0.2.4, but nova requires
newer version of `anyjson` e.g. 0.3.3 . Thus nova and glance/keystone cannot be
installed in one operating system.

The solution is use `virtualenv`:

- Run `./run_test.sh` in glance/keystone root folder
- It will install all dependencies in `$GLANCE/KEYSTONE_ROOT_FOLDER/.venv`
- To use virtualenv run `source .venv/bin/activate`
- To deactive virtualenv run `deactivate`

Since libvirt python package is included in system libvirt package and cannot
be installed inside virtualenv via pypi, it's recommended to install nova without virtualenv
and install glance/keystone with virtualenv on demand.

refs: http://www.virtualenv.org/en/latest/index.html


## ERROR: Unable to load glance-api-keystone from configuration file /etc/glance/glance-api-paste.ini.

It's anyjson problems. See above.


## nova-manage cannot sync database without any error printed

Add `pdb.traceback()` to nova-manage.

If traceback shows "Specified key was too long; max key length is 767 bytes",
it's nova sqlalchemy migration bug on github stable/essex:

- Table "dns_domains" in mysql causes the problem.
- Primary key "domain" in "dns_domain" with length 512 and UTF-8 encoded will go beyond 767 bytes
which is max length of InnoDB allowed.

Temporally solution is to change 512 to 256 in `nova.db.sqlalchemy.migrate_repo.versions`

Fixed in folsom-1


## nova: XXX command not found

It's `nova-rootwrap` problems:

- Default nova-rootwrap is prepared for Debian/Ubuntu liked operating systems.
- To run openstack on other linux distros like Archlinux/Gentoo modifying nova-rootwrap
is required.

Check and verify `nova.rootwrap.compute/network/volume` match your system.

refs: https://bugs.launchpad.net/nova/+bug/970893


## ImportError: No module named XXX

Install relevant packages.

e.g.:

    ImportError: No module named lockfile

(Archlinux)
> stanzgy % pacman -Sy community/python2-lockfile

or other package with same content



## cannot access vnc page on dashboard

`nova-novncproxy` is not running or `novnc` is not correctly configured.

nova-novncproxy depends on package `websockify` which is not in pypi and you need
to install it manually.

refs: https://github.com/kanaka/websockify


## keystone can't save service data in SQL server

modify keystone.conf:

    [catalog]
    driver = keystone.catalog.backends.sql.Catalog

refs: https://answers.launchpad.net/keystone/+question/201703


# Acknowledgment

OpenStack 安装与配置（单节点）: http://wiki.hz.netease.com/OpenStack/Single-Node
