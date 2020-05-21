# Gevent 中文入门教程

## 核心

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
    # 可以理解成一个 IO 阻塞的操作，阻塞了2秒，这时 Greenlet 会自动切换到 gr2() 上下文执行 
    print('Ended Polling: %s' % tic())

def gr2():
    # Busy waits for a second, but we don't want to stick around...
    print('Started Polling: %s' % tic())
    select.select([], [], [], 2)
    print('Ended Polling: %s' % tic())

def gr3():
    print("Hey lets do some stuff while the greenlets poll, %s" % tic())
    gevent.sleep(1)
    # 让当前 Greenlet 休眠1秒，上面 gr1() gr2() 阻塞操作完成后，再切换到 gr1() gr2() 的上下文执行

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

另一个比较综合的例子定义了一个不确定的任务函数 (它的输出不能保证对相同的输入给出相同的结果) 。在这种情况下，运行该函数的副作用是任务暂停执行的时间是随机的。

```Python
import gevent
import random

def task(pid):
    """
    Some non-deterministic task
    """
    gevent.sleep(random.randint(0,2)*0.001)
    print('Task %s done' % pid)

def synchronous():
    for i in range(1,10):
        task(i)

def asynchronous():
    threads = [gevent.spawn(task, i) for i in xrange(10)]
    gevent.joinall(threads)

print('Synchronous:')
synchronous()

print('Asynchronous:')
asynchronous()
```

```
Synchronous:
Task 1 done
Task 2 done
Task 3 done
Task 4 done
Task 5 done
Task 6 done
Task 7 done
Task 8 done
Task 9 done
Asynchronous:
Task 1 done
Task 5 done
Task 6 done
Task 2 done
Task 4 done
Task 7 done
Task 8 done
Task 9 done
Task 0 done
Task 3 done
```

在同步情况下，所有任务都是按顺序运行的，这会导致在执行每个任务时主程序阻塞(即暂停主程序的执行)。

程序的重要部分是 gevent.spawn，它将给定的函数封装在一个 Greenlet 线程中。初始化的 greenlet 列表存储在传递给 gevent 的数组线程中。gevent.joinall 函数，它会阻塞当前程序，来运行所有给定的 greenlet。只有当所有 greenlet 终止时，执行才会继续进行。

需要注意的是，异步情况下的执行顺序本质上是随机的，异步情况下的总执行时间比同步情况下少得多。实际上，同步的例子完成的最大时间是每个任务停顿0.002秒，导致整个队列停顿0.02秒。在异步情况下，最大运行时间大约为0.002秒，因为没有一个任务会阻塞其他任务的执行。

在更常见的用例中，异步地从服务器获取数据，fetch() 的运行时在请求之间会有所不同，这取决于请求时远程服务器上的负载。

```Python
import gevent.monkey
gevent.monkey.patch_socket()
# 把内置的阻塞的 socket替换成非阻塞的socket

import gevent
import urllib2
import simplejson as json

def fetch(pid):
    response = urllib2.urlopen('http://json-time.appspot.com/time.json')
    result = response.read()
    json_result = json.loads(result)
    datetime = json_result['datetime']

    print('Process %s: %s' % (pid, datetime))
    return json_result['datetime']

def synchronous():
    for i in range(1,10):
        fetch(i)

def asynchronous():
    threads = []
    for i in range(1,10):
        threads.append(gevent.spawn(fetch, i))
    gevent.joinall(threads)

print('Synchronous:')
synchronous()

print('Asynchronous:')
asynchronous()
```

### 确定性

如前所述，greenlet 是确定的。给定相同的 greenlet 配置和相同的输入集，它们总是产生相同的输出。例如，让我们将一个任务分散到一个多进程（multiprocessing）池中，并将其结果与一个gevent池的结果进行比较。

```Python
import time

def echo(i):
    time.sleep(0.001)
    return i

# Non Deterministic Process Pool

from multiprocessing.pool import Pool

p = Pool(10)
run1 = [a for a in p.imap_unordered(echo, xrange(10))]          # imap_unordered 返回一个顺序随机的 iterable 对象
run2 = [a for a in p.imap_unordered(echo, xrange(10))]
run3 = [a for a in p.imap_unordered(echo, xrange(10))]
run4 = [a for a in p.imap_unordered(echo, xrange(10))]

print(run1 == run2 == run3 == run4)

# Deterministic Gevent Pool

from gevent.pool import Pool

p = Pool(10)
run1 = [a for a in p.imap_unordered(echo, xrange(10))]      # [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
run2 = [a for a in p.imap_unordered(echo, xrange(10))]      # [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
run3 = [a for a in p.imap_unordered(echo, xrange(10))]      # [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
run4 = [a for a in p.imap_unordered(echo, xrange(10))]      # [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]

print(run1 == run2 == run3 == run4)
```

```
False
True
```

尽管 gevent 通常是确定的，但当您开始与外部服务(如 socket 和文件)进行交互时，非确定性的来源可能会潜入您的程序。因此，即使 green 线程是确定性并发的一种形式，它们仍然会遇到POSIX线程和进程遇到的一些相同的问题。

与并发有关的长期问题称为竞争条件。简而言之，当两个并发线程/进程依赖于某些共享资源但还试图修改该值时，就会发生竞争状态。这将导致资源的值变得依赖于执行顺序。这是一个问题，一般来说，应该尽量避免竞态条件，因为它们会导致全局的不确定程序行为。

最好的方法是在任何时候都避免所有全局状态。全局状态和导入时间的副作用总是会回来咬你一口

### 生成 Greenlets

gevent提供了一些关于Greenlet初始化的包装器。一些最常见的模式是：

```Python
import gevent
from gevent import Greenlet

def foo(message, n):
    """
    Each thread will be passed the message, and n arguments
    in its initialization.
    """
    gevent.sleep(n)
    print(message)

# Initialize a new Greenlet instance running the named function
# foo
thread1 = Greenlet.spawn(foo, "Hello", 1)

# Wrapper for creating and running a new Greenlet from the named
# function foo, with the passed arguments
thread2 = gevent.spawn(foo, "I live!", 2)

# Lambda expressions
thread3 = gevent.spawn(lambda x: (x+1), 2)

threads = [thread1, thread2, thread3]

# Block until all threads complete.
gevent.joinall(threads)
```

```
Hello
I live!
```

除了使用基本的Greenlet类，您还可以子类化 Greenlet 类并覆盖 _run 方法。

```Python
import gevent
from gevent import Greenlet

class MyGreenlet(Greenlet):

    def __init__(self, message, n):
        Greenlet.__init__(self)
        self.message = message
        self.n = n

    def _run(self):
        print(self.message)
        gevent.sleep(self.n)

g = MyGreenlet("Hi there!", 3)
g.start()
g.join()
```

```
Hi there!
```

### Greenlets 状态

与代码的其他部分一样，greenlet可能以各种方式失败。greenlet可能无法抛出异常、无法停止或消耗太多系统资源。

greenlet 的内部状态通常是一个与时间相关的参数。在greenlets上有许多标志，它们允许您监视线程的状态：

started -- 布尔值，指示Greenlet是否已启动

ready() -- 布尔值，指示Greenlet是否已停止

successful() -- 布尔值，指示Greenlet是否已停止且没有抛出异常

value -- Greenlet返回的值

exception -- 异常，在greenlet中抛出的未捕获异常实例

```Python
import gevent

def win():
    return 'You win!'

def fail():
    raise Exception('You fail at failing.')

winner = gevent.spawn(win)
loser = gevent.spawn(fail)

print(winner.started) # True
print(loser.started)  # True

# Exceptions raised in the Greenlet, stay inside the Greenlet.
try:
    gevent.joinall([winner, loser])
except Exception as e:
    print('This will never be reached')

print(winner.value) # 'You win!'
print(loser.value)  # None

print(winner.ready()) # True
print(loser.ready())  # True

print(winner.successful()) # True
print(loser.successful())  # False

# The exception raised in fail, will not propagate outside the
# greenlet. A stack trace will be printed to stdout but it
# will not unwind the stack of the parent.

print(loser.exception)

# It is possible though to raise the exception again outside
# raise loser.exception
# or with
# loser.get()
```

```
True
True
You win!
None
True
True
True
False
You fail at failing.
```

### 程序关闭

当主程序接收到SIGQUIT时，无法生成（yield）的 greenlet 可能会使程序的执行时间比预期的更长。这将导致所谓的“僵尸进程”，需要从 Python 解释器外部杀死这些进程。

一种常见的模式是监听主程序上的SIGQUIT事件并在退出前调用 gevent.shutdown 。

```Python
import gevent
import signal

def run_forever():
    gevent.sleep(1000)

if __name__ == '__main__':
    gevent.signal(signal.SIGQUIT, gevent.kill)
    thread = gevent.spawn(run_forever)
    thread.join()
```

### 超时

超时是对代码块或Greenlet的运行时的约束。

```Python
import gevent
from gevent import Timeout

seconds = 10

timeout = Timeout(seconds)
timeout.start()

def wait():
    gevent.sleep(10)

try:
    gevent.spawn(wait).join()
except Timeout:
    print('Could not complete')
```

在with语句中，它们还可以与上下文管理器一起使用。

```Python
import gevent
from gevent import Timeout

time_to_wait = 5 # seconds

class TooLong(Exception):
    pass

with Timeout(time_to_wait, TooLong):
    gevent.sleep(10)
```

此外，gevent 还为各种 Greenlet 和数据结构相关的调用提供超时参数。例如：

```Python
import gevent
from gevent import Timeout

def wait():
    gevent.sleep(2)

timer = Timeout(1).start()
thread1 = gevent.spawn(wait)

try:
    thread1.join(timeout=timer)
except Timeout:
    print('Thread 1 timed out')

# --

timer = Timeout.start_new(1)
thread2 = gevent.spawn(wait)

try:
    thread2.get(timeout=timer)
except Timeout:
    print('Thread 2 timed out')

# --

try:
    gevent.with_timeout(1, wait)
except Timeout:
    print('Thread 3 timed out')
```

```
Thread 1 timed out
Thread 2 timed out
Thread 3 timed out
```

### 猴子补丁

我们来到了Gevent的黑暗角落。到目前为止，我一直避免提到monkey patching，以尝试和激发强大的协同模式，但是现在是讨论monkey patching的黑魔法的时候了。
如果您注意到上面我们调用了命令 monkey.patch_socket()，这是一个纯粹用于修改标准库套接字库(socket)的副作用命令。

```Python
import socket
print(socket.socket)

print("After monkey patch")
from gevent import monkey
monkey.patch_socket()
print(socket.socket)

import select
print(select.select)
monkey.patch_select()
print("After monkey patch")
print(select.select)
```

```
class 'socket.socket'
After monkey patch
class 'gevent.socket.socket'

built-in function select
After monkey patch
function select at 0x1924de8
```

Python 允许在运行时修改大多数对象，包括模块、类甚至函数。这通常是一个令人震惊的坏主意，因为它创建了一个“隐式副作用”，如果出现问题，通常非常难以调试，然而在极端情况下，库需要改变Python本身的基本行为，可以使用monkey补丁。在这种情况下，gevent能够修补标准库中的大多数阻塞系统调用，包括 socket、ssl、threading 和 select 模块中的调用。

例如，Redis-python 的绑定通常使用常规tcp socket与Redis-server实例通信。只需调用gevent.monkey.patch_all()，我们可以让redis绑定协同调度请求，并与gevent堆栈的其他部分一起工作。

这让我们可以在不编写任何代码的情况下集成通常无法与gevent一起工作的库。（尽管猴子补丁仍然是邪恶的，但在这种情况下，它是“有用的邪恶”。）

## 数据结构

### 事件

事件是greenlet之间异步通信的一种形式。

```Python
import gevent
from gevent.event import Event

'''
Illustrates the use of events
'''


evt = Event()

def setter():
    '''After 3 seconds, wake all threads waiting on the value of evt'''
    print('A: Hey wait for me, I have to do something')
    gevent.sleep(3)
    print("Ok, I'm done")
    evt.set()   # 运行到evt.set()会将flag设置为True，然后另外两个被阻塞的waitter的evt.wait()方法在看到flag已经为True之后不再继续阻塞运行并且结束。


def waiter():
    '''After 3 seconds the get call will unblock'''
    print("I'll wait for you")
    evt.wait()  # blocking
    print("It's about time")

def main():
    gevent.joinall([
        gevent.spawn(setter),
        gevent.spawn(waiter),
        gevent.spawn(waiter),
        gevent.spawn(waiter),
        gevent.spawn(waiter),
        gevent.spawn(waiter)
    ])

if __name__ == '__main__': main()
```

事件对象的扩展是 AsyncResult，它允许您在唤醒调用时发送一个值。这有时被称为future或deferred，因为它有对 future 值的引用，可以在任意的时间设置该值。

```Python
import gevent
from gevent.event import AsyncResult
a = AsyncResult()

def setter():
    """
    After 3 seconds set the result of a.
    """
    gevent.sleep(3)
    a.set('Hello!')

def waiter():
    """
    After 3 seconds the get call will unblock after the setter
    puts a value into the AsyncResult.
    """
    print(a.get())

gevent.joinall([
    gevent.spawn(setter),
    gevent.spawn(waiter),
])
```

### 队列

队列是按顺序排列的数据集，它们具有通常的 put / get 操作，但可以在Greenlets上安全操作的方式编写。

例如，如果一个Greenlet从队列中获取一个项目(item)，则同一项目(item)不会被同时执行的另一个Greenlet获取。

```Python
import gevent
from gevent.queue import Queue

tasks = Queue()

def worker(n):
    while not tasks.empty():
        task = tasks.get()
        print('Worker %s got task %s' % (n, task))
        gevent.sleep(0)

    print('Quitting time!')

def boss():
    for i in xrange(1,25):
        tasks.put_nowait(i)

gevent.spawn(boss).join()

gevent.joinall([
    gevent.spawn(worker, 'steve'),
    gevent.spawn(worker, 'john'),
    gevent.spawn(worker, 'nancy'),
])
```

```
Worker steve got task 1
Worker john got task 2
Worker nancy got task 3
Worker steve got task 4
Worker john got task 5
Worker nancy got task 6
Worker steve got task 7
Worker john got task 8
Worker nancy got task 9
Worker steve got task 10
Worker john got task 11
Worker nancy got task 12
Worker steve got task 13
Worker john got task 14
Worker nancy got task 15
Worker steve got task 16
Worker john got task 17
Worker nancy got task 18
Worker steve got task 19
Worker john got task 20
Worker nancy got task 21
Worker steve got task 22
Worker john got task 23
Worker nancy got task 24
Quitting time!
Quitting time!
Quitting time!
```

根据需要，队列还可以在put或get上阻塞。

每个put和get操作都有一个非阻塞的对应操作，put_nowait 和 get_nowait 不会阻塞。如果操作是不可能的，会引发 gevent.queue.Empty 或 gevent.queue.Full

在这个例子中，我们让boss同时和workers运行，并且对队列有一个限制，防止它包含三个以上的元素。这个限制意味着put操作将阻塞，直到队列上有空间为止。相反，如果队列上没有要获取的元素，get操作就会阻塞，它还会接受一个超时参数，如果在超时的时间范围内找不到工作(work)，则允许队列以异常 gevent.queue.Empty 退出。

```Python
import gevent
from gevent.queue import Queue, Empty

tasks = Queue(maxsize=3)

def worker(name):
    try:
        while True:
            task = tasks.get(timeout=1) # decrements queue size by 1
            print('Worker %s got task %s' % (name, task))
            gevent.sleep(0)
    except Empty:
        print('Quitting time!')

def boss():
    """
    Boss will wait to hand out work until a individual worker is
    free since the maxsize of the task queue is 3.
    """

    for i in xrange(1,10):
        tasks.put(i)                            # 输入1，2，3，到4时，队列达到最大值，put方法阻塞，gevent 切换到下一个协程worker（steve）
    print('Assigned all work in iteration 1')

    for i in xrange(10,20):
        tasks.put(i)
    print('Assigned all work in iteration 2')

gevent.joinall([
    gevent.spawn(boss),
    gevent.spawn(worker, 'steve'),
    gevent.spawn(worker, 'john'),
    gevent.spawn(worker, 'bob'),
])
```

```
Worker steve got task 1
Worker john got task 2
Worker bob got task 3
Worker steve got task 4
Worker john got task 5
Worker bob got task 6
Assigned all work in iteration 1
Worker steve got task 7
Worker john got task 8
Worker bob got task 9
Worker steve got task 10
Worker john got task 11
Worker bob got task 12
Worker steve got task 13
Worker john got task 14
Worker bob got task 15
Worker steve got task 16
Worker john got task 17
Worker bob got task 18
Assigned all work in iteration 2
Worker steve got task 19
Quitting time!
Quitting time!
Quitting time!
```

### 分组和池

组是运行中的greenlet的集合，这些greenlet作为组一起管理和调度。它还兼做并行调度程序，借鉴 Python multiprocessing 库。

```Python
import gevent
from gevent.pool import Group

def talk(msg):
    for i in xrange(3):
        print(msg)

g1 = gevent.spawn(talk, 'bar')
g2 = gevent.spawn(talk, 'foo')         
g3 = gevent.spawn(talk, 'fizz')  

group = Group()
group.add(g1)                           
group.join()                      # 修改了官方的例子，这里join只会让当前线程等待g1，但g2和g3已经被启动，会被继续安排执行
```

```
bar
bar
bar
foo
foo
foo
fizz
fizz
fizz
```

这对于管理异步任务组非常有用。

如上所述，Group还提供了一个API，用于将作业分派给分组的greenlet并以各种方式收集它们的结果。

```Python
import gevent
from gevent import getcurrent
from gevent.pool import Group

group = Group()

def hello_from(n):
    print('Size of group %s' % len(group))
    print('Hello from Greenlet %s' % id(getcurrent()))

group.map(hello_from, xrange(3))


def intensive(n):
    gevent.sleep(3 - n)
    return 'task', n

print('Ordered')

ogroup = Group()
for i in ogroup.imap(intensive, xrange(3)):
    print(i)

print('Unordered')

igroup = Group()
for i in igroup.imap_unordered(intensive, xrange(3)):
    print(i)
```

```
Size of group 3
Hello from Greenlet 4340152592
Size of group 3
Hello from Greenlet 4340928912
Size of group 3
Hello from Greenlet 4340928592
Ordered
('task', 0)
('task', 1)
('task', 2)
Unordered
('task', 2)
('task', 1)
('task', 0)
```

池是一种结构，用于处理需要限制并发的动态数量的greenlets。在需要并行执行许多网络或IO绑定任务的情况下，这通常是可取的。


```Python
import gevent
from gevent.pool import Pool

pool = Pool(2)

def hello_from(n):
    print('Size of pool %s' % len(pool))

pool.map(hello_from, xrange(3))
```

```
Size of pool 2
Size of pool 2
Size of pool 1
```

通常在构建gevent驱动的服务时，会将整个服务围绕一个池结构进行中心处理。一个例子可能是在各种套接字（socket）上轮询的类。

```Python
from gevent.pool import Pool

class SocketPool(object):

    def __init__(self):
        self.pool = Pool(1000)
        self.pool.start()

    def listen(self, socket):
        while True:
            socket.recv()

    def add_handler(self, socket):
        if self.pool.full():
            raise Exception("At maximum pool size")
        else:
            self.pool.spawn(self.listen, socket)

    def shutdown(self):
        self.pool.kill()
```

### 锁和信号量

信号量是一种低级同步原语，它允许greenlet协调和限制并发访问或执行。信号量公开两种方法，获取和释放信号量被获取和释放的次数之差称为信号量的界限。如果信号量范围达到0，它就会阻塞，直到另一个greenlet释放它的捕获。

```Python
from gevent import sleep
from gevent.pool import Pool
from gevent.coros import BoundedSemaphore

sem = BoundedSemaphore(2)

def worker1(n):
    sem.acquire()
    print('Worker %i acquired semaphore' % n)
    sleep(0)
    sem.release()
    print('Worker %i released semaphore' % n)

def worker2(n):
    with sem:
        print('Worker %i acquired semaphore' % n)
        sleep(0)
    print('Worker %i released semaphore' % n)

pool = Pool()
pool.map(worker1, xrange(0,2))
pool.map(worker2, xrange(3,6))
```

```
Worker 0 acquired semaphore
Worker 1 acquired semaphore
Worker 0 released semaphore
Worker 1 released semaphore
Worker 3 acquired semaphore
Worker 4 acquired semaphore
Worker 3 released semaphore
Worker 4 released semaphore
Worker 5 acquired semaphore
Worker 5 released semaphore
```

界限为1的信号量称为锁。它提供对一个greenlet的独占执行。它们通常用于确保资源只在程序上下文中使用一次。

翻译持续更新中 ...
