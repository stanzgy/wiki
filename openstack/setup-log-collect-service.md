# 日志收集服务

由于目前openstack测试环境上开放的服务, 涉及的物理机越来越多, 登录到不同的机器上查看日志
寻找问题越来越成为让人困扰的问题.

本文主要描述如何使用开源软件logstash建立日志收集, 查看, 搜索, 过滤服务.

本文前面主要说明如何使用现有的已经搭建好的日志服务, 或者使用已经配置好的tarball搭建自己的日志服务.
后面主要说明日志服务的结构以及如何使用开源软件从头配置类似的日志服务.

由于自己刚刚接触logstash和其他相关的开源软件不久, 文中提到的日志收集流程设计, 各个服务的
配置不一定合理, 仅作参考用, 欢迎大家和我讨论.


## Have a try

目前已经在测试环境的虚拟机里建立了相关的日志服务, 可以通过下面的地址查看, 用户名/密码和测试环境上的horizon一样.

    http://114.113.199.243/

收集的日志文件为11/12服务器上下面所列的共8个文件

  * nova-compute.log
  * nova-network.log
  * nova-scheduler.log
  * nova-api.log

每一条日志信息都有如下的属性:

  * (field) hostname: 日志所在物理机hostname
  * (field) path: 日志路径
  * (field) file: 日志文件名


  * (field) timestamp: 日志时间戳
  * (field) level: 日志LEVEL
  * (field) message: 日志内容
  * (field) module: 日志涉及的模块
  * (field) request: 日志涉及的request


  * (metadata) @message: 元数据, 从input获得的原始的日志数据
  * (metadata) @source: 元数据, 日志来源
  * (metadata) @source_host: 元数据, 日志来源
  * (metadata) @tags: 元数据, 日志tag
  * (metadata) @timestamp: 元数据, 日志时间戳
  * (metadata) @type: 元数据, 日志类型


  * (field) destination: stomp相关, 可忽略
  * (field) message-id: stomp相关, 可忽略
  * (field) subscription: stomp相关, 可忽略


比如想搜索11服务器上从9月4日到13日中nova-compute.log的ERROR log信息, 输入下面的搜索命令即可

    hostname:"114-113-199-11" file:"nova-compute.log" level:"ERROR" @timestamp:[2012-09-04 TO 2012-09-13]

或

    114-113-199-11 nova-compute.log ERROR @timestamp:[2012-09-04 TO 2012-09-13]

更详细的搜索语法请参考[lucene query syntax](http://lucene.apache.org/core/old_versioned_docs/versions/3_4_0/queryparsersyntax.html)


## Build your own log collect service

### Server side

* 下载已经配置好的tarball

    <pre>
    stanzgy % wget http://netease.stanzgy.org/~stanzgy/misc/log_service.tar.gz
    </pre>

* 将压缩包解压到/data目录下(由于日志存储需要空间比较大, 需要将虚拟机较大的一块硬盘格式化并mount到/data上)

* 检查apollo, logstash, elasticsearch三个服务的配置文件, (比如host/user的配置等)

  - apollo

    <pre>
    {APOLLO_BROKER_HOME}/etc/apollo.xml
    {APOLLO_BROKER_HOME}/etc/groups.properties
    {APOLLO_BROKER_HOME}/etc/users.properties
    </pre>

  - logstash

    <pre>
    {LOGSTASH_HOME}/logstash.conf
    </pre>

  - elasticsearch

    <pre>
    {ES_HOME}/elasticsearch-0.19.9/config/elasticsearch.yml
    </pre>

* 开启虚拟机上的apollo, logstash, elasticsearch服务

  - apollo

    <pre>
    stanzgy % {APOLLO_BROKER_HOME}/bin/apollo-broker-service start
    </pre>

  - elasticsearch

    <pre>
    stanzgy % {ES_HOME}/elasticsearch-0.19.9/bin/elasticsearch
    </pre>

  - logstash

    <pre>
    stanzgy % java -jar {LOGSTASH_HOME}/logstash-1.1.1-monolithic_mod.jar agent -f {LOGSTASH_HOME}/logstash.conf -- web --port 8080 --backend 'elasticsearch://localhost:7300/stanzgy'
    </pre>

    注意logstash的启动非常慢, 视机器性能可能需要几分钟才能启动完成, 所以logstash启动后
    最好不要随意kill. 运行后看到logstash输出调试信息后可以根据启动logstash时配置的地址和端口访问web界面.

### Client side

  * 获得用来自动将日志中更新的内容发送到apollo的agent

    <pre>
    git clone git://github.com/stanzgy/stomp-agent.git
    </pre>

  * 安装依赖的python package `stomp.py`

    <pre>
    stanzgy % sudo pip install stomp.py
    </pre>

  * 修改agent.conf. 其中logfile改为自己想要监视的日志文件的路径, stomp中的内容修改为
  虚拟机中apollo服务所设置的相应参数. 比如,

    <pre>
    [config]
    interval=0.5
    logfile=logfile=/var/log/nova/nova-compute.log, /var/log/nova/nova-network.log

    [stomp]
    server = "localhost"
    port = 61613
    destination = "/topic/log.localhost.test"
    user = "admin"
    password = "admin"
    </pre>

  * 运行agent

    <pre>
    stanzgy % python agent.py -c agent.conf &
    </pre>


## Service Architecture

### Summary

目前基于logstash搭建的日志收集服务的服务器端结构如下面所示

      client    |                               server       web interface
     -----------+---------------------------------------------------^-------------------------------
                                                                    |
      _______        __________                __________           |         _______________
     | agent | ---> |          |    input     |          | <--------+        |               |
     |_______|      | ActiveMQ | -----------> |          |    query logs     |               |
      _______       | (apollo) |    pull      | Logstash | <---------------> | ElasticSearch |
     | agent | ---> |          | <----------- |          |                   |               |
     |_______|      |__________|              |__________|              +--> |_______________|
                         ^                         |             output |            ^
        ...              |     ____________        |     ________       |            |     ________
                         +--> | /queue/log |       +--> | filter | ---+ |            +--> | indice |
                         |    |____________|            |________|    | |            |    |________|
                         |     ____________              ________     | |            |     ________
                         +--> | /topic/log |       +--- | filter | <--+ |            +--> | indice |
                         |    |____________|       |    |________|      |            |    |________|
                         |     ____________        |                    |            |
    store raw logs here  +--> | /dsub/log  |       +-->    ...     -----+            +-->    ...
                              |____________|      process logs via filters          store logs here

![logstash arch](http://netease.stanzgy.org/~stanzgy/pics/gollum/logstash_arch.png)


在服务端, 主要运行3个独立的服务: `apollo`(activemq), `logstash`, `elasticsaerch`.

* `apollo`负责以消息队列的方式收集不同物理机上agent发来的日志数据. logstash支持很多种
  数据输入形式, 这里选择从`apollo`输入数据主要是希望使用`apollo`作为日志集中汇总收集的
  工具, 当需要增删更改日志数据源时, 不需要停掉`logstash`进行配置.

* `logstash`负责将从消息队列中拿到的raw数据进行抽取特征, 匹配出日志的时间戳等信息,
  然后将整理过的数据存入backend.

* `elasticsearch`是当前的后端存储服务. `logstash`支持很多输出形式, 选择`elasticsearch`
  作为后端存储的主要原因是:

  - `logstash`自带的web界面主要是为`elasticsearch`设计的
  - `elasticsearch`本身是一个开源搜索引擎的实现, 对日志数据进行过滤/搜索都很方便
  - `elasticsearch`支持在线自动扩容, 扩展性很好, 十分灵活.


### apollo configuration

#### get source tarball

    stanzgy % wget http://mirrors.tuna.tsinghua.edu.cn/apache/activemq/activemq-apollo/1.4/apache-apollo-1.4-unix-distro.tar.gz
    stanzgy % tar -zxvf apache-apollo-1.4-unix-distro.tar.gz

#### create apollo broker

    stanzgy % ${APOLLO_HOME}/bin/apollo create mybroker

#### mod config files

mod config files in `{APOLLO_BROKER_HOME}/etc/`, you could refer to apollo config files in
[this package](http://netease.stanzgy.org/~stanzgy/misc/log_service.tar.gz).

#### run apollo broker

    stanzgy % {APOLLO_BROKER_HOME}/bin/apollo-broker run

or

    stanzgy % {APOLLO_BROKER_HOME}/bin/apollo-broker-service start


### elasticsearch configuration

#### get source tarball

    stanzgy % wget https://github.com/downloads/elasticsearch/elasticsearch/elasticsearch-0.19.9.tar.gz
    stanzgy % stanzgy % tar -zxvf elasticsearch-0.19.9.tar.gz

#### mod config files

mod config files in `{ES_HOME}/config/`, you could refer to elasticsearch config files in
[this package](http://netease.stanzgy.org/~stanzgy/misc/log_service.tar.gz).

#### run elasticsearch

    stanzgy % {ES_HOME}/elasticsearch-0.19.9/bin/elasticsearch


### logstash configuration

#### get logstash jar package

    stanzgy % wget http://semicomplete.com/files/logstash/logstash-1.1.1-monolithic.jar

#### mod logstash

    stanzgy % mkdir mod
    stanzgy % cp logstash-1.1.1-monolithic.jar mod/
    stanzgy % cd mod
    stanzgy % unzip -q logstash-1.1.1-monolithic.jar
    stanzgy % rm logstash-1.1.1-monolithic.jar

##### add nova log parse pattern

mod/patterns/nova

    NOVA_TIMESTAMP %{TIMESTAMP_ISO8601:timestamp}
    NOVA_LOGLEVEL (%{LOGLEVEL}|TRACE|AUDIT)
    NOVA_MODULE %{DATA:module}
    NOVA_REQUEST (\[%{DATA:request}\])
    NOVA_MSG %{GREEDYDATA:message}

    NOVA_LOG %{NOVA_TIMESTAMP} %{NOVA_LOGLEVEL:level} %{NOVA_MODULE} (%{NOVA_REQUEST} )?%{NOVA_MSG}

##### add stomp log file reading agent support in logstash inputs plugins

mod/logstash/inputs/stomp_mod.rb

```ruby
    require "logstash/inputs/base"
    require "logstash/namespace"

    class LogStash::Inputs::Stomp < LogStash::Inputs::Base
      config_name "stomp_mod"
      plugin_status "beta"

      # The address of the STOMP server.
      config :host, :validate => :string, :default => "localhost", :required => true

      # The port to connet to on your STOMP server.
      config :port, :validate => :number, :default => 61613

      # The username to authenticate with.
      config :user, :validate => :string, :default => ""

      # The password to authenticate with.
      config :password, :validate => :password, :default => ""

      # The destination to read events from.
      #
      # Example: "/topic/logstash"
      config :destination, :validate => :string, :required => true

      # Enable debugging output?
      config :debug, :validate => :boolean, :default => false

      private
      def connect
        begin
          @client.connect
          @logger.debug("Connected to stomp server") if @client.connected?
        rescue => e
          @logger.debug("Failed to connect to stomp server, will retry", :exception => e, :backtrace => e.backtrace)
          sleep 2
          retry
        end
      end

      public
      def register
        require "onstomp"
        @client = OnStomp::Client.new("stomp://#{@host}:#{@port}", :login => @user, :passcode => @password.value)
        @stomp_url = "stomp://#{@user}:#{@password}@#{@host}:#{@port}/#{@destination}"

        # Handle disconnects
        @client.on_connection_closed {
          connect
          subscription_handler # is required for re-subscribing to the destination
        }
        connect
      end # def register

      private
      def subscription_handler
        @client.subscribe(@destination) do |msg|
          e = to_event(msg.body, @stomp_url)
          msg.headers.each { |k, v| e[k] = v }
          @output_queue << e if e
        end
      end

      public
      def run(output_queue)
        @output_queue = output_queue
        subscription_handler
      end # def run
    end # class LogStash::Inputs::Stomp
```

#### rebuild logstash jar

    stanzgy % cd mod
    stanzgy % zip -q -r logstash.jar *
    stanzgy % mv logstash.jar ../logstash-1.1.1-monolithic_mod.jar

#### mod logstash config file

mod config files in `{LOGSTASH_HOME}`, you could refer to logstash config files in
[this package](http://netease.stanzgy.org/~stanzgy/misc/log_service.tar.gz).

NOTE that you must use input source `stomp_mod` to support stomp log file reading agent.
For example:

in logstash.conf

    ...

    input {
      ...

      stomp_mod {
        debug => false
        type => "nova"
        host => "localhost"
        port => 61613
        user => "xxx"
        password => "xxx"
        destination => "/dsub/log"
        tags => [ "stomp" ]
      }

      ...
    }

    ...

#### run logstash

    stanzgy % java -jar logstash-1.1.1-monolithic_mod.jar agent -f logstash.conf -- web --address 0.0.0.0 --port 8080 --backend 'elasticsearch://localhost:7300/stanzgy' 2>~/logstash.err.log
