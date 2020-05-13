# gevent-tutorial-chinese

### Core 核心

## Greenlets

gevent中使用的主要模式是Greenlet，这是作为C扩展模块提供给Python的一个轻量级协同程序。所有greenlet都在主程序的OS进程中运行，但它们是协同调度的。

在任何给定的时间内，只有一个greenlet在运行。

这不同于由 multiprocessing 或 threading 库提供的任何真正的并行结构，这些库可以自旋进程和POSIX线程，它们由操作系统调度，并且真正并行的。
