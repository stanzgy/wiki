## 小议Python3的原生协程机制

在最近发布的 Python 3.5 版本中，官方正式引入了 `async`/`await`关键字、在
`asyncio` [1] 标准库中实现了IO多路复用、原生协程（coroutine）与
事件循环（event loop），让人耳目一新，本文也尝试对 Python 3.5 新增加的原生协程
机制与`asyncio`标准库相关的内容做一个小结。

IO多路复用与协程的引入，可以极大的提高高负载下程序的IO性能表现。几年前，M$在
C# 中便已经通过引入 `async`/`await` 关键字实现了一套异步IO机制，成为业界模范，
Python现在引入相同的编程范式也算博众家之长。

事实上，在官方实现原生协程机制前，在目前比较流行的 Python 2.X 版本中，由于
GIL [2] 的存在，Python 程序的多线程/多进程性能非常不理想，使得协程成为 Python
并发编程的最佳模型，大量的 Python 项目都开始通过使用第三方库实现的协程编写程序
（如 eventlet / gevent ）、特别是网络编程相关的 Python 项目。不过限于 Python 2
语言实现的局限，协程的实现比较原始，众多第三方库的实现并不统一，并且通常都需要
使用一些特殊的编程技巧（monkey patch / green 标准库等手段）才能实现非阻塞IO等特
性来真正提高性能。

相比之下，直接官方提供标准库原生支持"async io"与协程的 Python 3.5 编写程序无疑
更加方便。


### async/await

官方在 PEP-492 [3] 中定义了 `async`/`await` 关键字来使用协程。

声明一个协程非常简单，通过 `async` 实现：

    async def foo():
        pass

在普通的函数声明前加上`async`关键字，这个函数就变成了一个协程。

需要注意的是，在 `async def` 定义的协程内，不能含有 `yield` 或 `yield from`
表达式，否则会报 SyntaxError 异常。

`await`表示等待另一个协程执行完成返回，必须在协程内才能使用。

下面是一个官方文档中提供的协程简单示例，非常直观的表示了 `async`/`await`、协程
与事件循环的执行过程：

    import asyncio

    async def compute(x, y):
        print("Compute %s + %s ..." % (x, y))
        await asyncio.sleep(1.0)
        return x + y

    async def print_sum(x, y):
        result = await compute(x, y)
        print("%s + %s = %s" % (x, y, result))

    loop = asyncio.get_event_loop()
    loop.run_until_complete(print_sum(1, 2))
    loop.close()

![mitmproxy](../asset/py3_coro_1.png)

从图中可以看到，`asyncio` 标准库自带的事件循环负责所有协程的调度。在首先执行
print_sum 协程时，内部使用`await`等待另外一个协程compute的返回结果，于是本地
协程被挂起去执行协程compute。而协程compute执行过程中调用了
`await asyncio.sleep(1.0)`，于是本地协程挂起1秒、将程序控制权交回事件循环，1秒
后再恢复协程compute的执行直到整个stack执行结束。

而在大量协程并发执行的过程中，除了在协程中主动使用 `await`，当本地协程发生
IO等待、调用 `asyncio.sleep()`等方法时，程序的控制权也会在不同的协程间切换，从
而在 GIL 的限制下实现最大程度的并发执行，不会由于等待IO等原因导致程序阻塞，达到
较高的性能表现。

值得一提的是，`asyncio` 和类似 `libev` 一样的众多第三方类库实现的事件循环一样，
在不同平台上会使用不同的轮询机制，比如在Linux平台上使用 poll/epoll、BSD平台上
使用 kqueue、NT内核上会直接使用Proactor模式的完成端口。


### yield from

如果有一直关注 Python 3 原生协程实现的同学，应该会知道其实它是靠生成器
（Generator）实现的。`yield from` 则是在 PEP-380 [4] 中新增加的生成器定义
关键字，与原生协程的实现密不可分。

原理上 `yield from` 基本等价于 `await`，只是 `await` 针对协程的实现做了更多具体
的处理与约定。

`yield from` 表达式的语义定义有很长的一串，要彻底搞明白它的实现，需要先学习在
PEP-342 [5] 中描述的增强型生成器（Enhanced Generators），但这里我们可以简单的把
它看作一个生成器语法糖：

    for v in g:
        yield v

加上在生成器内

    return value

等价于

    raise StopIteration(value)

这样实现以后，像这样的表达式

    y = f(x)

我们就可以用 `yield from` 语法将 f(x) 改造成协程实现了

    y = yield from g(x)

g 是 f 的生成器，只需要保证两者最终返回值一致，我们并不关心生成器 g 的中间
状态。

如果要详细了解这里的实现，建议还是阅读 PEP-380 与 PEP-342 原文，里面有专门的
章节专门描述这个问题。


### context manager

PEP-492 中让人眼前一亮的一点是定义了协程的上下文管理器（context manager），新增
了 `async with` 语法，让我们可以将一个上下文作为协程处理，在进入（enter）和退出
（exit）一个 BLOCK 时做协程调度操作：

    async with EXPR as VAR:
        BLOCK

与普通的 context manager 相比，async 版本的上下文管理器在内部方法前加了字母 "a"

    class AsyncContextManager:
        async def __aenter__(self):
            await log('entering context')

        async def __aexit__(self, exc_type, exc, tb):
            await log('exiting context')

一种典型的应用就是进入临界区

    class Lock:
        async def __aenter__(self):
            await self.lock.lock()

        async def __aexit__(self, exc_type, exc, tb):
            await self.lock.release()


    async with Lock():
        ...

类似的语法还有 `async for`

    async for TARGET in ITER:
        BLOCK

这里不再赘述。


### promise/future

`promise` 是一种最近在 nodejs 流行起来另外一种异步编程范式，在不同的地方可能也
被称作 `future` / `deferred`，但一般都指的是同一种类似的东西。`promise` 并没有
一个非常官方的标准，我了解的比较知名的`promise`标准规范有 `Promises A+` [6]。
Python 从 Python 3.4 开始也提供了 `asyncio.Future` 实现类似的功能。

`promise` 的核心思想是为一个异步操作定义操作成功和失败的不通情况下的的回调
函数。在 nodejs 这类缺少官方 `await` 语法支持的语言中，能有效减轻
`callback hell` 问题，让代码更简洁，减少 raw callback 写法导致的缩进太多的
问题，并能方便的实现链式操作。

而在 Python 中，语言语法本身提供的功能已经足够丰富，没有 `callback hell` 这类
问题，`Future` 则可以更专注的让我们可以将各种异步操作以一种顺序的、更接近人类
逻辑思维与自然语言方式描述出来：

    import asyncio

    @asyncio.coroutine
    def slow_operation(future):
        yield from asyncio.sleep(1)
        future.set_result('Future is done!')

    loop = asyncio.get_event_loop()
    future = asyncio.Future()
    asyncio.ensure_future(slow_operation(future))
    loop.run_until_complete(future)
    print(future.result())
    loop.close()


### 小结

在现在流行的编程语言中，异步、协程、事件循环被越来越多的关注与使用，Python 中的
asyncio.coroutine、Ruby 中的 Fiber、Node 中的 Promise、甚至像 Scala 这样的语言
中也有了 Future，更不用说 Golang 这种在这条路上一头走到底的编程语言。
`await`/`Future` 这样的异步编程方式被越来越多的普及与使用，
以后可能会像 if、for 一样成为编程语言不可缺少的一部分，而协程这种轻量级线程在
某些情境下可能也会越来越多的代替目前的多线程/多进程成为主流的并发编程方式。

美中不足的是，秉承 Guido 一向以来只挖坑不填坑的习惯，Python 3.5 实现新的
`async`/`await` 关键字后，并没有给出具体的现有第三方类库如何向新的 native 协程
实现迁移的方案，更不用说一些流行的 Python 2 第三方类库目前连 Python 3 都不
支持。距离我们真正能方便流畅地使用体验它可能还需要一段时间。


### References

[1]: https://docs.python.org/3/library/asyncio.html
[2]: https://wiki.python.org/moin/GlobalInterpreterLock
[3]: https://www.python.org/dev/peps/pep-0492
[4]: https://www.python.org/dev/peps/pep-0380
[5]: https://www.python.org/dev/peps/pep-0342
[6]: https://github.com/promises-aplus/promises-spec
[7]: https://lwn.net/Articles/643786
