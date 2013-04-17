## nova中定时任务(periodic_task)原理分析

在nova源代码中, 可以在很多函数上看到`@periodic_task`这样的修饰符, 我们知道这是nova的定时任务,
可以让这个函数周期性执行, 但是可能不太了解这个修饰符产生作用的原理和用法, 这里将详细说明一下.


### decorator `nova.manager.periodic_task`

```python
    # in nova.manager
    def periodic_task(*args, **kwargs):
        def decorator(f):
            f._periodic_task = True
            f._ticks_between_runs = kwargs.pop('ticks_between_runs', 0)
            return f

        if kwargs:
            return decorator
        else:
            return decorator(args[0])
```

可以看到`@periodic_task`其实只是给被修饰的函数加上了`_periodic_task`和`_ticks_between_runs`
2个attr, 并没有做其他的操作. 周期执行函数的action实际上是`nova.manager.Manager`和`nova.utils.LoopingCall`
配合实现的, 后面将详细说明.


### black magic of `nova.manager.ManagerMeta`

```python
    # in nova.manager
    class ManagerMeta(type):
        def __init__(cls, names, bases, dict_):
            super(ManagerMeta, cls).__init__(names, bases, dict_)

            try:
                cls._periodic_tasks = cls._periodic_tasks[:]
            except AttributeError:
                cls._periodic_tasks = []

            try:
                cls._ticks_to_skip = cls._ticks_to_skip.copy()
            except AttributeError:
                cls._ticks_to_skip = {}

            for value in cls.__dict__.values():
                if getattr(value, '_periodic_task', False):
                    task = value
                    name = task.__name__
                    cls._periodic_tasks.append((name, task))
                    cls._ticks_to_skip[name] = task._ticks_between_runs
```

nova.manager中的`ManagerMeta`类是`nova.manager.Manager`的metaclass(在后面可以看到), 这里可以
简单的认为`ManagerMeta`是`Manager`的父类. 在`nova.manager.Manager`初始化时, 会调用`ManagerMeta`的
`__init__()`方法.

(metaclass属于python里的黑魔法内容, 这里不做详细说明, 大法师们感兴趣可以去看看官方手册
http://docs.python.org/reference/datamodel.html#customizing-class-creation)

    for value in cls.__dict__.values():
        if getattr(value, '_periodic_task', False):

`ManagerMeta.__init__()`中的这两行会将cls(也就是`Manager`对象自己, 注意是 **Manager对象** 不是`Manager`的实例)
中所有使用过`@periodic_task`修饰符修饰的 **函数对象** 过滤出来.
过滤后会将这些函数对象放入`Manager._periodic_tasks`中, 后面定时任务的实现都是从这个变量里取出函数对象并执行.


### `nova.manager.Manager.periodic_tasks`
(注意和`nova.manager.periodic_task`的区别)

```python
    class Manager(base.Base):
        __metaclass__ = ManagerMeta

        def periodic_tasks(self, context, raise_on_error=False):
            for task_name, task in self._periodic_tasks:
                full_task_name = '.'.join([self.__class__.__name__, task_name])

                ticks_to_skip = self._ticks_to_skip[task_name]
                if ticks_to_skip > 0:
                    LOG.debug(_("Skipping %(full_task_name)s, %(ticks_to_skip)s"
                                " ticks left until next run"), locals())
                    self._ticks_to_skip[task_name] -= 1
                    continue

                self._ticks_to_skip[task_name] = task._ticks_between_runs
                LOG.debug(_("Running periodic task %(full_task_name)s"), locals())

                try:
                    task(self, context)
                except Exception as e:
                    if raise_on_error:
                        raise
                    LOG.exception(_("Error during %(full_task_name)s: %(e)s"),
                                  locals())
```

从前面`ManagerMeta`的说明我们已经知道`Manager`对象建立时, 会将所有attr `_periodic_task`为 **True** 的函数对象
放入`self._periodic_tasks`中.

在`Manager`中, 我们可以发现一个和`periodic_task`十分相似的函数`periodic_tasks`, 通过阅读函数代码可以发现
这个函数实际上的作用就是把所有在`self._periodic_tasks`中的函数对象(也就是所有用`@periodic_task`修饰符修饰过的函数)
全部遍历并调用一遍. 如果能定期调用这个函数的话, 就能实现类似linux中crontab的定时任务功能.

下面将说明nova如何定时调用`nova.manager.Manager.periodic_tasks`实现定时任务.


### `nova.utils.LoopingCall`

```python
    class LoopingCall(object):
        def __init__(self, f=None, *args, **kw):
            self.args = args
            self.kw = kw
            self.f = f
            self._running = False

        def start(self, interval, now=True):
            self._running = True
            done = event.Event()

            def _inner():
                if not now:
                    greenthread.sleep(interval)
                try:
                    while self._running:
                        self.f(*self.args, **self.kw)
                        if not self._running:
                            break
                        greenthread.sleep(interval)
                except LoopingCallDone, e:
                    self.stop()
                    done.send(e.retvalue)
                except Exception:
                    LOG.exception(_('in looping call'))
                    done.send_exception(*sys.exc_info())
                    return
                else:
                    done.send(True)

            self.done = done

            greenthread.spawn(_inner)
            return self.done

        def stop(self):
            self._running = False

        def wait(self):
            return self.done.wait()
```

`nova.utils.LoopingCall`的作用就是实现前面提到的定时调用函数的功能.

将函数对象作为`LoopingCall`的第一个构造参数传入构造一个`LoopingCall`对象, 然后调用其`start()`方法后
调用其`wait()`方法, 就可以实现定时执行函数的功能.

`start`方法的`interval`参数为函数两次执行期间的时间间隔, 单位为秒. 前面提到的`@periodic_task`修饰符
可以设置一个参数`ticks_between_runs`, 是与其配合使用的, 指经过几次ticks才执行函数. 比如, 上下文为
`@periodic_task(ticks_between_runs=2)`并且`interval=5`的话, 被修饰的函数将每 (2+1)*5=15 seconds 执行一次

如果被修饰的函数里`raise nova.utils.LoopingCallDone`, 可以让LoopingCall的定时任务close gracefully.


## Examples

下面两个例子演示了如何使用`nova.manager`中的`@periodic_task`修饰符, 或直接使用`nova.utils.LoopingCall`
在nova中实现定时任务.



### periodic_task.py

[[raw file|http://netease.stanzgy.org/~stanzgy/script/periodic_task/periodic_task.py]]

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
            super(TestManager, self).__init__(*args, **kwargs)

        @manager.periodic_task
        def foooooo(self, context=None):
            self.count += 1
            print "Just you know why"
            #print time.ctime()

        @manager.periodic_task
        def baaaaar(self, context=None):
            self.count += 1
            print "Panda"
            #print time.ctime()

        @manager.periodic_task(ticks_between_runs=3)
        def panda(self, context=None):
            print "Panda. Mae teleheshka."


    test_manager = TestManager()
    periodic = utils.LoopingCall(test_manager.periodic_tasks, context=None)
    periodic.start(interval=0.8753)
    periodic.wait()
```

Output:

    stanzgy % python periodic_task.py
    Just you know why
    Panda
    Just you know why
    Panda
    Just you know why
    Panda
    Just you know why
    Panda
    Panda. Mae teleheshka.
    Just you know why
    Panda
    Just you know why
    Panda
    Just you know why
    Panda
    Just you know why
    Panda
    Panda. Mae teleheshka.
    ...


### periodic_func.py

[[raw file|http://netease.stanzgy.org/~stanzgy/script/periodic_task/periodic_func.py]]

```python
    #!/usr/bin/env python2

    from nova import utils
    import inspect
    import eventlet

    global count
    count=0

    def panda(tagline):
        global count
        count += 1
        print "#", count, "Panda.", tagline
        if count >= 9:
            raise utils.LoopingCallDone

    periodic = utils.LoopingCall(panda, "Mae teleheshka.")
    periodic.start(interval=0.8753)
    periodic.wait()
```

Output:

    stanzgy % python periodic_func.py
    # 1 Panda. Mae teleheshka.
    # 2 Panda. Mae teleheshka.
    # 3 Panda. Mae teleheshka.
    # 4 Panda. Mae teleheshka.
    # 5 Panda. Mae teleheshka.
    # 6 Panda. Mae teleheshka.
    # 7 Panda. Mae teleheshka.
    # 8 Panda. Mae teleheshka.
    # 9 Panda. Mae teleheshka.
