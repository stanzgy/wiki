# 114.113.199.11服务器上出现nova心跳任务(report_state)超时问题的分析


## 问题

114.113.199.11服务器上的nova会间歇性的出现类似下面的心跳超时的情况(默认心跳间隔为10s),
导致某些服务不能正常运行.

    2012-08-31 14:09:06 DEBUG nova.service [-] Running report state from (pid=29975) report_state /usr/lib/python2.7/dist-packages/nova/service.py:299
    2012-08-31 14:09:16 DEBUG nova.service [-] Running report state from (pid=29975) report_state /usr/lib/python2.7/dist-packages/nova/service.py:299
    2012-08-31 14:09:26 DEBUG nova.service [-] Running report state from (pid=29975) report_state /usr/lib/python2.7/dist-packages/nova/service.py:299
    2012-08-31 14:09:36 DEBUG nova.service [-] Running report state from (pid=29975) report_state /usr/lib/python2.7/dist-packages/nova/service.py:299
    2012-08-31 14:10:00 DEBUG nova.service [-] Running report state from (pid=29975) report_state /usr/lib/python2.7/dist-packages/nova/service.py:299
    2012-08-31 14:10:10 DEBUG nova.service [-] Running report state from (pid=29975) report_state /usr/lib/python2.7/dist-packages/nova/service.py:299
    2012-08-31 14:10:20 DEBUG nova.service [-] Running report state from (pid=29975) report_state /usr/lib/python2.7/dist-packages/nova/service.py:299


## 分析

nova中的心跳任务, 和定时任务一样, 都是由utils.LoopingCall()实现. 唯一的区别的是心跳
的interval为10s(report_interval), 而定时任务为60s(periodic_interval).

```python
    # in nova.service.py

    cfg.IntOpt('report_interval',
               default=10,
               help='seconds between nodes reporting state to datastore'),
    cfg.IntOpt('periodic_interval',
               default=60,
               help='seconds between running periodic tasks'),

    ...

    if self.report_interval:
        pulse = utils.LoopingCall(self.report_state)
        pulse.start(interval=self.report_interval, now=False)
        self.timers.append(pulse)

    if self.periodic_interval:
        periodic = utils.LoopingCall(self.periodic_tasks)
        periodic.start(interval=self.periodic_interval, now=False)
        self.timers.append(periodic)
```

经过查看日志, 可以初步判断为, nova中某些定时任务执行时间过长, 并且没能及时切换到心跳的
定时任务上, 导致心跳函数没能按照预订的时间执行, 从而导致上面的问题.

之前在[CLOUD-1487](http://jira.hz.netease.com/browse/CLOUD-1487)中已经简单讨论
了nova中的定时任务(periodic_task)的运行原理. 在分析11服务器上出现的心跳超时的问题前,
还需要再深入讨论一下nova中的定时任务运行原理.

nova中的定时任务是靠python eventlet这个package实现的. 按照我们的理解, eventlet是一个
实现函数并发执行的库, 不应该出现定时任务影响心跳任务的现象. 为了深入了解eventlet的并发实现和
nova的定时任务, 我们先做几个有趣的实验.


### example #1

[raw file](http://netease.stanzgy.org/~stanzgy/script/periodic_task/sleep_test.py)

```python
    #!/usr/bin/env python2

    import eventlet


    def foo():
        count = 0

        while True:
            print "fooooo"
            #eventlet.greenthread.sleep(0)


    def bar():
        count = 0

        while True:
            print "baaaar"
            #eventlet.greenthread.sleep(0)


    pool = eventlet.GreenPool()
    pool.spawn(foo)
    pool.spawn(bar)
    pool.waitall()
```

如果直接运行上面的程序, 可以发现程序将一直执行`foo()`输出"fooooo", 而不会切换到函数
`bar()`上输出"baaaar". 如果去掉代码中`#eventlet.greenthread.sleep(0)`两行里的
注释, 程序可以按我们设想的同时输出"fooooo"和"baaaar".

由此可见, eventlet中spawn产生的greenthread和真正意义上的线程还是有本质区别的, 它
并不像线程是靠抢占CPU时间片的方式来实现并发执行, 而是靠互相主动交出CPU控制权(`sleep(0)`)
的方式实现不同greenthread的并发执行. 因此, 这里的greenthread也被称作coroutine(协程).


注: 实际上, 对某些函数使用`eventlet.spawn`出的greenthread, 在某些条件下确实可以不用对
函数做任何修改就能实现并发运行的效果, 但是不适用这个例子里的情况.

关于eventlet并发实现的详细信息请参考[CLOUD-1561](http://jira.hz.netease.com/browse/CLOUD-1561)


### example #2

[raw file](http://netease.stanzgy.org/~stanzgy/script/periodic_task/sleep_test_with_sleep.py)

```python
    #!/usr/bin/env python2

    import eventlet
    import inspect
    from pprint import pprint

    def foo():
        count = 0

        while True:
            print "fooooo"
            count += 1

            # get greenthread wrapper for this func from stack
            f = inspect.getmembers(inspect.stack()[1][0])
            f_l = filter(lambda x: x[0] == 'f_locals', f)[0][1]
            gt = f_l['self']

            if count >= 6:
                # eventlet.greenthread.GreenThread.kill()
                gt.kill()

            eventlet.greenthread.sleep(0)


    def bar():
        count = 0

        while True:
            print "baaaar"
            count += 1

            # get greenthread wrapper for this func from stack
            f = inspect.getmembers(inspect.stack()[1][0])
            f_l = filter(lambda x: x[0] == 'f_locals', f)[0][1]
            gt = f_l['self']

            if count >= 9:
                # eventlet.greenthread.GreenThread.kill()
                gt.kill()

            eventlet.greenthread.sleep(0)


    pool = eventlet.GreenPool()
    pool.spawn(foo)
    pool.spawn(bar)
    pool.waitall()
```

这个例子是在上面的程序稍作修改得来. 在程序中添加了循环次数计数器, 并且从stack内拿到了封装
函数的greenthread, 在函数循环不同次数以后将自身的greenthread kill掉. 输出结果:

    fooooo
    baaaar
    fooooo
    baaaar
    fooooo
    baaaar
    fooooo
    baaaar
    fooooo
    baaaar
    fooooo
    baaaar
    baaaar
    baaaar
    baaaar

可以看出, 程序开始还是按我们设想的交替执行, 在循环到6次时, 我们kill掉`foo()`的greenthread
可以发现`bar()`独自执行完了后三次的运行. 从这里可以发现, `eventlet.greenthread.sleep(0)`的
作用是将CPU的控制权交给greenthread所在`GreenPool`中其他的greenthread, 若`GreenPool`
中只有自己, 那CPU的控制权还是会回到自己的手里.


### example #3

这个例子, 对[nova_periodic_task_example](http://localhost:4567/hzzhanggy/inside-nova-periodic-task#Examples)
里的例程做了一些修改, 来模拟114.113.199.11服务器上出现的定时任务影响心跳任务的情况.

[raw file](http://netease.stanzgy.org/~stanzgy/script/periodic_task/report_state_test.py)

```python
    #!/usr/bin/env python2

    from nova import manager
    from nova import utils
    import time
    import inspect
    import eventlet


    class TestManager(manager.Manager):
        def __init__(self, *args, **kwargs):
            self.count = 0
            self.n = 100000000
            super(TestManager, self).__init__(*args, **kwargs)

        @manager.periodic_task
        def panda(self, context=None):
            print "Panda. Mae teleheshka."

        @manager.periodic_task
        def loop(self, context=None):
            x = time.time()
            while self.n > 0:
                self.n -= 1

            if self.n == 0:
                self.n = 100000000

            self.count += 1
            y = time.time()
            print "Loop #", self.count, "  ", "time used: ", (y-x)

        @manager.periodic_task
        def foo(self, context=None):
            print "Just you know why..."


    def report_state():
        print time.ctime(), "report state"

    test_manager = TestManager()
    periodic = utils.LoopingCall(test_manager.periodic_tasks, context=None)
    periodic.start(interval=0.1)

    pulse = utils.LoopingCall(report_state)
    pulse.start(interval=1)

    periodic.wait()
    pulse.wait()
```

输出结果:

    stanzgy % python report_state_test.py
    Panda. Mae teleheshka.
    Loop # 1    time used:  18.5929889679
    Just you know why...
    Sun Sep  2 22:51:31 2012 report state
    Panda. Mae teleheshka.
    Loop # 2    time used:  18.8424971104
    Just you know why...
    Sun Sep  2 22:51:50 2012 report state
    Panda. Mae teleheshka.


从结果可以看出, 本来应该每秒执行一次的心跳任务`report_state()`, 在这个例子里受
`TestManager()`中的一个需要执行时间较长的定时任务`loop()`的影响, 过了19秒才执行了一次.

这也是114.113.199.11服务器上目前的问题, 分析日志可以知道11服务器上的心跳任务主要被计费模块
的某些定时任务影响导致不能按时运行.


### example #4

一个暂时的解决办法是在CPU bound或IO bound的定时任务里手动添加一些`sleep(0)`, 可以解决这个问题.
添加sleep(0)的时机需要根据编写相关代码的攻城湿的经验判断, 如下例中的

    if self.n % 1000 == 0:
        eventlet.greenthread.sleep(0)

注:    这里的IO bound特指读写普通文件. 读写socket/named pipe等不需要添加sleep(0).
    具体请参考[CLOUD-1561](http://jira.hz.netease.com/browse/CLOUD-1561)

[raw file](http://netease.stanzgy.org/~stanzgy/script/periodic_task/report_state_test_with_sleep.py)

```python
    #!/usr/bin/env python2

    from nova import manager
    from nova import utils
    import time
    import inspect
    import eventlet


    class TestManager(manager.Manager):
        def __init__(self, *args, **kwargs):
            self.count = 0
            self.n = 100000000
            super(TestManager, self).__init__(*args, **kwargs)

        @manager.periodic_task
        def panda(self, context=None):
            print "Panda. Mae teleheshka."

        @manager.periodic_task
        def loop(self, context=None):
            x = time.time()
            while self.n > 0:
                self.n -= 1
                if self.n % 1000 == 0:
                    eventlet.greenthread.sleep(0)
                    pass

            if self.n == 0:
                self.n = 100000000

            self.count += 1
            y = time.time()
            print "Loop #", self.count, "  ", "time used: ", (y-x)

        @manager.periodic_task
        def foo(self, context=None):
            print "Just you know why..."


    def report_state():
        print time.ctime(), "report state"

    test_manager = TestManager()
    periodic = utils.LoopingCall(test_manager.periodic_tasks, context=None)
    periodic.start(interval=0.1)

    pulse = utils.LoopingCall(report_state)
    pulse.start(interval=1)

    periodic.wait()
    pulse.wait()
```

输出:

    Panda. Mae teleheshka.
    Sun Sep  2 23:36:19 2012 report state
    Sun Sep  2 23:36:20 2012 report state
    Sun Sep  2 23:36:21 2012 report state
    Sun Sep  2 23:36:22 2012 report state
    Sun Sep  2 23:36:23 2012 report state
    Sun Sep  2 23:36:24 2012 report state
    Sun Sep  2 23:36:25 2012 report state
    Sun Sep  2 23:36:26 2012 report state
    Sun Sep  2 23:36:27 2012 report state
    Sun Sep  2 23:36:28 2012 report state
    Sun Sep  2 23:36:29 2012 report state
    Sun Sep  2 23:36:30 2012 report state
    Sun Sep  2 23:36:31 2012 report state
    Sun Sep  2 23:36:32 2012 report state
    Sun Sep  2 23:36:33 2012 report state
    Sun Sep  2 23:36:34 2012 report state
    Sun Sep  2 23:36:35 2012 report state
    Sun Sep  2 23:36:36 2012 report state
    Sun Sep  2 23:36:37 2012 report state
    Sun Sep  2 23:36:38 2012 report state
    Sun Sep  2 23:36:39 2012 report state
    Sun Sep  2 23:36:40 2012 report state
    Sun Sep  2 23:36:41 2012 report state
    Sun Sep  2 23:36:42 2012 report state
    Sun Sep  2 23:36:43 2012 report state
    Sun Sep  2 23:36:44 2012 report state
    Sun Sep  2 23:36:45 2012 report state
    Sun Sep  2 23:36:46 2012 report state
    Sun Sep  2 23:36:47 2012 report state
    Sun Sep  2 23:36:48 2012 report state
    Sun Sep  2 23:36:49 2012 report state
    Sun Sep  2 23:36:50 2012 report state
    Sun Sep  2 23:36:51 2012 report state
    Sun Sep  2 23:36:52 2012 report state
    Loop # 1    time used:  33.5270152092
    Just you know why...

可以看到, `report_state()`已经可以正常运行了, 不过由于上下文切换的开销, loop()的运行时间从~18s
涨到了~33s, 几乎张了一倍, 所以sleep(0)的运行时机需要攻城湿自己权衡.


**注意**: 目前nova中的定时任务(`nova.manager.Manager.periodic_tasks`)本身是作为
一个单独函数被LoopCall调用, 他会把所有用修饰符`@periodic_task`标记过的函数 **顺序** 执行
一遍.

在被`@periodic_task`标记的函数内部使用`sleep(0)`的作用仅仅是把CPU控制权交给
心跳任务(`report_state`), 并 **不会** 在标记了`@periodic_task`函数之间做切换.


## 结论

- 114.113.199.11上出现的nova心跳任务超时的原因是计费模块的定时任务影响导致


## 可能的解决方法

- 在可能导致超时的定时任务中, 手动插入sleep(0)
- 新功能独立实现, 不要放到nova中, 避免对定时任务和心跳任务可能的影响
- 加强code review, 添加修改nova代码时使用`@periodic_task`修饰符要谨慎
