# python eventlet并发原理分析


## motivation

114.113.199.11服务器上nova服务中基于python eventlet实现的定时任务(periodic_task)和
心跳任务(report_state)都是eventlet的一个greenthread实例.

目前服务器上出现了nova定时任务中某些任务执行时间过长而导致心跳任务不能准时运行的问题.

如果eventlet是一个完全意义上的类似线程/进程的并发库的话, 不应该出现这个问题, 需要研究
eventlet的并发实现, 了解它的并发实现原理, 避免以后出现类似的问题.


## 分析

经过阅读eventlet源代码, 可以知道eventlet主要依赖另外2个python package:

- greenlet
- python-epoll (或其他类似的异步IO库, 如poll/select等)


主要做了3个工作:

- 封装greenlet
- 封装epoll
- 改写python标准库中相关的module, 以便支持epoll


### epoll

epoll是linux实现的一个基于事件的异步IO库, 在之前类似的异步IO库poll上改进而来.

下面两个例子会演示如何用epoll将阻塞的IO操作用epoll改写为异步非阻塞. (取自官方文档)

#### blocking IO

```python
    import socket

    EOL1 = b'\n\n'
    EOL2 = b'\n\r\n'
    response  = b'HTTP/1.0 200 OK\r\nDate: Mon, 1 Jan 1996 01:01:01 GMT\r\n'
    response += b'Content-Type: text/plain\r\nContent-Length: 13\r\n\r\n'
    response += b'Hello, world!'

    serversocket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    serversocket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    serversocket.bind(('0.0.0.0', 8080))
    serversocket.listen(1)

    try:
        while True:
            connectiontoclient, address = serversocket.accept()
            request = b''
            while EOL1 not in request and EOL2 not in request:
                request += connectiontoclient.recv(1024)
            print('-'*40 + '\n' + request.decode()[:-2])
            connectiontoclient.send(response)
            connectiontoclient.close()
    finally:
        serversocket.close()
```

这个例子实现了一个简单的监听在8080端口的web服务器. 通过一个死循环不停的接收来自8080端口
的连接, 并返回结果.

需要注意的是程序会在

    connectiontoclient, address = serversocket.accept()

这一行block住, 直到获取到新的连接, 程序才会继续往下运行.

同时, 这个程序同一个时间内只能处理一个连接, 如果有很多用户同时访问8080端口, 必须要按先后
顺序依次处理这些连接, 前面一个连接成功返回后, 才会处理后面的连接.

下面的例子将用epoll将这个简单的web服务器改写为异步的方式


#### non-blocking IO by using epoll

```python
import socket, select

EOL1 = b'\n\n'
EOL2 = b'\n\r\n'
response  = b'HTTP/1.0 200 OK\r\nDate: Mon, 1 Jan 1996 01:01:01 GMT\r\n'
response += b'Content-Type: text/plain\r\nContent-Length: 13\r\n\r\n'
response += b'Hello, world!'

serversocket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
serversocket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
serversocket.bind(('0.0.0.0', 8080))
serversocket.listen(1)
serversocket.setblocking(0)

epoll = select.epoll()
epoll.register(serversocket.fileno(), select.EPOLLIN)

try:
    connections = {}; requests = {}; responses = {}
    while True:
        events = epoll.poll(1)
        for fileno, event in events:
            if fileno == serversocket.fileno():
                connection, address = serversocket.accept()
                connection.setblocking(0)
                epoll.register(connection.fileno(), select.EPOLLIN)
                connections[connection.fileno()] = connection
                requests[connection.fileno()] = b''
                responses[connection.fileno()] = response
            elif event & select.EPOLLIN:
                requests[fileno] += connections[fileno].recv(1024)
                if EOL1 in requests[fileno] or EOL2 in requests[fileno]:
                    epoll.modify(fileno, select.EPOLLOUT)
                    print('-'*40 + '\n' + requests[fileno].decode()[:-2])
            elif event & select.EPOLLOUT:
                byteswritten = connections[fileno].send(responses[fileno])
                responses[fileno] = responses[fileno][byteswritten:]
                if len(responses[fileno]) == 0:
                    epoll.modify(fileno, 0)
                    connections[fileno].shutdown(socket.SHUT_RDWR)
            elif event & select.EPOLLHUP:
                epoll.unregister(fileno)
                connections[fileno].close()
                del connections[fileno]
finally:
    epoll.unregister(serversocket.fileno())
    epoll.close()
    serversocket.close()
```

可以看到, 例子中首先使用`serversocket.setblocking(0)`将socket设为异步的模式, 然后
用`select.epoll()`新建了一个epoll, 接着用`epoll.register(serversocket.fileno(), select.EPOLLIN)`
将该socket上的IO输入事件(`select.EPOLLIN`)注册到epoll里. 这样做了以后, 就可以将
上面例子中会在`socket.accept()`这步阻塞的Main Loop改写为基于异步IO事件的epoll循环了.

    events = epoll.poll(1)

简单的说, 如果有很多用户同时连接到8080端口, 这个程序会同时accept()所有的socket连接,
然后通过这行代码将发生IO事件socket放到events中, 并在后面循环中处理. 没有发生IO事件的
socket不会在loop中做处理. 这样使用epoll就实现了一个简单的并发web服务器.

注意, 这里提到的并发, 和我们通常所理解线程/进程的并发并不太一样, 更准确的说, 是 **IO多路复用** .


### greenlet

greentlet是python中实现我们所谓的"Coroutine(协程)"的一个基础库.

看了下面的例子就明白了.

```python
    from greenlet import greenlet

    def test1():
        print 12
        gr2.switch()
        print 34

    def test2():
        print 56
        gr1.switch()
        print 78

    gr1 = greenlet(test1)
    gr2 = greenlet(test2)
    gr1.switch()
```

输出

    12
    56
    34

程序先分别为两个函数定义了2个greenlet: gr1和gr2.

`gr1.switch()`显式切换到gr1上执行, gr1中输出"12"后`gr2.switch()`显式切换到gr2上执行
输出56, 又`gr1.switch()`显式切换到gr1上, 输出34. test1()执行结束, gr1 die. 于是
test2()里的78不会输出.

可以发现greenlet仅仅是实现了一个最简单的"coroutine", 而eventlet中的greenthread是在
greenlet的基础上封装了一些更high-level的功能, 比如greenlet的调度等.


### eventlet.green

从epoll的运行机制可以看出, 要使用异步IO, 必须要将相关IO操作改写成non-blocking的方式.
但是我们用`eventlet.spawn()`的函数, 并没有针对epoll做任何改写, 那eventlet是怎么实现
异步IO的呢?

这也是eventlet这个package最凶残的地方, 它自己重写了python标准库中IO相关的操作, 将它们
改写成支持epoll的模式, 放在eventlet.green中.

比如说, `socket.accept()`被改成了这样

    def accept(self):
        if self.act_non_blocking:
            return self.fd.accept()
        fd = self.fd
        while True:
            res = socket_accept(fd)
            if res is not None:
                client, addr = res
                set_nonblocking(client)
                return type(self)(client), addr
            trampoline(fd, read=True, timeout=self.gettimeout(),
                           timeout_exc=socket.timeout("timed out"))

然后在eventlet.spawn()的时候, 通过
一些高阶魔法和"huge hack", 将这些改写过得模块"patch"到spawn出的greenthread上, 从而
实现epoll的IO多路复用, 相当凶残.


### eventlet并发机制分析

前面说了这么多, 这里可以分析一下eventlet的并发机制了.

eventlet的结构如下图所示

     _______________________________________
    | python process                        |
    |   _________________________________   |
    |  | python thread                   |  |
    |  |   _____   ___________________   |  |
    |  |  | hub | | pool              |  |  |
    |  |  |_____| |   _____________   |  |  |
    |  |          |  | greenthread |  |  |  |
    |  |          |  |_____________|  |  |  |
    |  |          |   _____________   |  |  |
    |  |          |  | greenthread |  |  |  |
    |  |          |  |_____________|  |  |  |
    |  |          |   _____________   |  |  |
    |  |          |  | greenthread |  |  |  |
    |  |          |  |_____________|  |  |  |
    |  |          |                   |  |  |
    |  |          |        ...        |  |  |
    |  |          |___________________|  |  |
    |  |                                 |  |
    |  |_________________________________|  |
    |                                       |
    |   _________________________________   |
    |  | python thread                   |  |
    |  |_________________________________|  |
    |   _________________________________   |
    |  | python thread                   |  |
    |  |_________________________________|  |
    |                                       |
    |                 ...                   |
    |_______________________________________|

![eventlet arch](http://netease.stanzgy.org/~stanzgy/pics/gollum/eventlet_arch.png)


其中的hub和greenthread分别对应eventlet.hubs.hub和eventlet.greenthread, 本质都是
一个greenlet的实例.

hub中封装前面提到的epoll, epoll的事件循环是由`hub.run()`这个方法里实现. 每当用户调用
eventlet.spawn(), 就会在当前python线程的pool里产生一个新的greenthread. 由于greenthread
里的IO相关的python标准库被改写成non-blocking的模式(参考上面的`socket.accept()`).

每当greenthread里做IO相关的操作时, 最终都会返回到hub中的epoll循环, 然后根据epoll中的
IO事件, 调用响应的函数. 具体如下面所示.

`greenthread.sleep()`, 实际上也是将CPU控制权交给hub, 然后由hub调度下一个需要运行的
greenthread.

```python
    # in eventlet.hubs.poll.Hub

    def wait(self, seconds=None):
        readers = self.listeners[READ]
        writers = self.listeners[WRITE]

        if not readers and not writers:
            if seconds:
                sleep(seconds)
            return
        try:
            presult = self.poll.poll(int(seconds * self.WAIT_MULTIPLIER))
        except select.error, e:
            if get_errno(e) == errno.EINTR:
                return
            raise
        SYSTEM_EXCEPTIONS = self.SYSTEM_EXCEPTIONS

        for fileno, event in presult:
            try:
                if event & READ_MASK:
                    readers.get(fileno, noop).cb(fileno)
                if event & WRITE_MASK:
                    writers.get(fileno, noop).cb(fileno)
                if event & select.POLLNVAL:
                    self.remove_descriptor(fileno)
                    continue
                if event & EXC_MASK:
                    readers.get(fileno, noop).cb(fileno)
                    writers.get(fileno, noop).cb(fileno)
            except SYSTEM_EXCEPTIONS:
                raise
            except:
                self.squelch_exception(fileno, sys.exc_info())
                clear_sys_exc_info()
```


## 总结

eventlet实现的并发和我们理解的通常意义上类似线程/进程的并发是不同的, eventlet实现的"并发"
更准确的讲, 是 **IO多路复用** . 只有在被`eventlet.spawn()`的函数中存在可以 **支持异步IO**
相关的操作, 比如说读写socket/named pipe等时, 才能不用对被调用的函数做任何修改而实现
所谓的"并发".

如果被`eventlet.spawn()`的函数中存在大量的CPU计算或者读写普通文件, eventlet是无法对其
实现并发操作的. 如果想要在这样的greenthread间实现类似"并发"运行的效果, 需要手动的在函数
中插入`greenthread.sleep()`.

