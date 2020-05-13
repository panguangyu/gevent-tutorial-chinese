# Gevent 中文入门教程

## Core 核心

### Greenlets

gevent中使用的主要模式是 Greenlet，这是作为C扩展模块提供给 Python 的一个轻量级协同程序。所有 greenlet 都在主程序的 OS 进程中运行，但它们是协同调度的。

在任何给定的时间内，只有一个greenlet在运行。

这不同于由 multiprocessing 或 threading 库提供的并行结构，它们这些库可以自旋进程和POSIX线程，由操作系统调度，并且真正并行的。

### 异步和同步执行

并发性的核心思想是，可以将较大的任务分解为多个子任务的集合，这些子任务计划同时或异步运行，而不是一次或同步运行。两个子任务之间的切换称为上下文切换。

gevent中的上下文切换是通过 yielding 来完成的。在本例中，我们有两个上下文，它们通过调用 gevent.sleep(0) 相互让步。

```Python
import gevent

def foo():
    print('Running in foo')
    gevent.sleep(0)
    print('Explicit context switch to foo again')

def bar():
    print('Explicit context to bar')
    gevent.sleep(0)
    print('Implicit context switch back to bar')

gevent.joinall([
    gevent.spawn(foo),
    gevent.spawn(bar),
])
```

```
Running in foo
Explicit context to bar
Explicit context switch to foo again
Implicit context switch back to bar
```

当我们将 gevent 用于网络和 IO 绑定函数时，它的真正威力就来了，这些函数可以协同调度。Gevent 负责处理所有细节，以确保网络库尽可能隐式地让步出它们的 greenlet 上下文。我再怎么强调这是一个多么有力的成语也不为过。但也许可以举个例子来说明。

在这种情况下，select() 函数通常是一个阻塞调用，它轮询各种文件描述符。

```Python
import time
import gevent
from gevent import select

start = time.time()
tic = lambda: 'at %1.1f seconds' % (time.time() - start)

def gr1():
    # Busy waits for a second, but we don't want to stick around...
    print('Started Polling: %s' % tic())
    select.select([], [], [], 2)
    print('Ended Polling: %s' % tic())

def gr2():
    # Busy waits for a second, but we don't want to stick around...
    print('Started Polling: %s' % tic())
    select.select([], [], [], 2)
    print('Ended Polling: %s' % tic())

def gr3():
    print("Hey lets do some stuff while the greenlets poll, %s" % tic())
    gevent.sleep(1)

gevent.joinall([
    gevent.spawn(gr1),
    gevent.spawn(gr2),
    gevent.spawn(gr3),
])
```

```
Started Polling: at 0.0 seconds
Started Polling: at 0.0 seconds
Hey lets do some stuff while the greenlets poll, at 0.0 seconds
Ended Polling: at 2.0 seconds
Ended Polling: at 2.0 seconds
```
