## 异步关键词（async/await）

在计算机被发明之初，没有人会预料到它会发展的如此之迅猛、庞大，所以那时有许多现在看来很愚蠢的设计。而在网络领域，C10k 就是这样一个由于过去受限于时代的设计导致的问题。C10k 是指“在同时连接到服务器的客户端数量超过 10000 个的环境中，即便硬件性能足够， 依然无法正常提供服务”。为了解决这个问题，协程这个自单处理机系统时代就存在的老古董在时代的发展中又上了舞台，只不过这次是跟着同步 IO 多路复用和异步 IO 表演的。

在 Python 这种高度屏蔽系统底层实现的语言里，同步 IO 多路复用和异步 IO 的使用方式是一样的，所以下文所提到的异步 IO，不专指 IOCP 这类异步 IO，也指 select/epoll 这类同步 IO 多路复用的方式。

协程分有栈协程和无栈协程，Python 使用的便是基于生成器的无栈协程。无栈协程相对于有栈协程来说，优势在于无需切换调用栈，性能更佳；劣势在于无栈协程有传染性，一旦你想通过无栈协程为程序提速，就不得不把每个 IO 操作都变成基于无栈协程的异步操作。

Python 官方异步库是 asyncio，虽然被许多人写文批评，但毕竟是几个核心开发成员一起做出来的，所以目前为止还是影响力最大的异步库。虽然也有其他的异步库，例如 trio，同样可以使用 `async/await` 关键词，但影响力相对较小。

### `await`

经历过 Python 早期异步语法的人可能知道，一开始 Python 使用 `yield from` 用于等待一个协程完成，这与它底层的生成器设计保持了一致，但后来增加了 `async/await` 语法用来清晰协程和普通生成器的界限。

能够使用 `await obj` 语法去获取执行结果的对象就被称为可等待对象。`await ...` 会阻塞当前的协程不能继续进行，并且把执行权重新交给事件循环，由事件循环继续寻找、执行下一个可以继续执行的协程，而当 `await ...` 语句的结果被返回，当前协程就变成了一个可以继续执行的协程，事件循环会自己选择在一个某一个时刻执行它。

### Coroutine

在具体讲述 asyncio 的设计之前，要明确一点 `Coroutine` 对象是 Python 层面的概念，无论是 asyncio 还是 trio 都支持这一对象。`Coroutine` 对象只能通过调用 `async def` 方式定义的函数来创建。下面是一个最简单的例子，在这个例子里，通过调用 `async def` 定义的函数，创建了一个 `Coroutine` 对象。并且你会发现，`print` 语句根本没有执行——控制台不会有任何输出。

```python
async def i_want_to_sleep(second):
    print("I'm going to sleep.")


coroutine = i_want_to_sleep(10)
```

因为需要一个事件循环（Loop）让 `Coroutine` 跑起来，协程里的语句才会运行。而创建事件循环、管理各个协程，正是 asyncio/trio 所做的事情。下面是一个基于 asyncio 的简单例子，通过 `asyncio.run` 这个快捷函数启动了一个事件循环运行协程、等待协程完成、最后销毁事件循环。值得注意的是，这里使用了 `asyncio.sleep` 而不是传统的 `time.sleep`。因为 `time.sleep` 是作用于整个线程之上的，而一个事件循环上所有的协程都只运行在这一个线程里（免去了线程的切换这也是协程更快的原因），一旦你有什么操作挂起整个线程，所有协程都无法进行了。

```python
import asyncio


async def i_want_to_sleep(second):
    print("I'm going to sleep.")
    await asyncio.sleep(second)
    print("I woke up.")


coroutine = i_want_to_sleep(10)

asyncio.run(coroutine)
```

### Future/Task

接下来看看 asyncio 当中的两个对象 `Future` 和 `Task`。这两个对象与 `Coroutine` 对象一样，也是可以等待的对象，但有着它们自己的用途。

`Future` 对象顾名思义，它创建于当下，实际返回结果在未来。它也是一个可等待的对象，其作用在于将回调式的代码转变为以 `await` 关键词编写的顺序代码。通过 `future.set_result(result)` 可以设置 `future` 对象的结果，使得 `await future` 能够停止等待并返回一个结果。下面是一个伪代码。

```python
import asyncio


async def convert(other_task):
    loop = asyncio.get_event_loop()
    future = loop.create_future()

    other_task.add_callback(future.set_result)

    return await future
```

由 `Future` 的作用可知，在平常的业务代码里是很难用到它的。因为回调转 `async/await` 这种事情一般都由更为基础的库来做。

在业务代码里，使用的更多是另一个可等待对象 `Task`，它继承了 `Future` 除了 `set_result` 和 `set_exception` 之外的所有 API。传入一个可等待对象给 `loop.create_task` 就可以创建一个 `Task`。对于 `Task` 对象而言，无论你是否调用了 `await` 去等待它，它都会在事件循环里执行，这与 `Coroutine` 这个可等待对象的行为不一样。所以 `Task` 可以用于并发执行多个协程，如果不关心它们的结果，创建 `Task` 就行了，剩下的就是协程本身和事件循环的事情。但如果你关心它的执行情况，也有许多的 API 可以用于管理它。

### 兼容同步代码

Python 有庞大的同步库生态，纵然无栈协程有传染性，也不应该放弃它们。这里要提到两个 API：`loop.run_in_executor`、`asyncio.to_thread`。

通过 `loop.run_in_executor` 和 `concurrent.futures` 里的线程池或进程池配合，可以将原来的阻塞程序丢进线程池或进程池里执行，使得事件循环本身不被阻塞。这里的阻塞程序指同步的网络 IO 操作、文件的 IO 操作或大量的 CPU 运算。以下是一段样例代码，通过把阻塞的 IO 操作丢到线程池里、大量的 CPU 运算丢到进程池里来修改阻塞程序到非阻塞程序。

```python
import asyncio
import concurrent.futures

def blocking_io():
    # File operations (such as logging) can block the
    # event loop: run them in a thread pool.
    with open('/dev/urandom', 'rb') as f:
        return f.read(100)

def cpu_bound():
    # CPU-bound operations will block the event loop:
    # in general it is preferable to run them in a
    # process pool.
    return sum(i * i for i in range(10 ** 7))

async def main():
    loop = asyncio.get_running_loop()

    ## Options:

    # 1. Run in the default loop's executor:
    result = await loop.run_in_executor(
        None, blocking_io)
    print('default thread pool', result)

    # 2. Run in a custom thread pool:
    with concurrent.futures.ThreadPoolExecutor() as pool:
        result = await loop.run_in_executor(
            pool, blocking_io)
        print('custom thread pool', result)

    # 3. Run in a custom process pool:
    with concurrent.futures.ProcessPoolExecutor() as pool:
        result = await loop.run_in_executor(
            pool, cpu_bound)
        print('custom process pool', result)

asyncio.run(main())
```

`asyncio.to_thread` 则是 Python3.9 才出现的新 API，它其实是对 `loop.run_in_executor` 和 `concurrent.futures.ThreadPoolExecutor` 的二次封装。没有本质上的区别，都是把阻塞 IO 任务丢到线程池里执行保证事件循环所在的线程不会被阻塞。

```python
import asyncio

def blocking_io():
    print(f"start blocking_io at {time.strftime('%X')}")
    # Note that time.sleep() can be replaced with any blocking
    # IO-bound operation, such as file operations.
    time.sleep(1)
    print(f"blocking_io complete at {time.strftime('%X')}")

async def main():
    print(f"started main at {time.strftime('%X')}")

    await asyncio.gather(
        asyncio.to_thread(blocking_io),
        asyncio.sleep(1))

    print(f"finished main at {time.strftime('%X')}")

asyncio.run(main())
```
