## 前言
```
Concurrency is about dealing with lots of things at once.
Parallelism is about doing lots of things at once.
Not the same, but related.
One is about structure, one is about execution.
Concurrency provides a way to structure a solution to solve a problem that may (but not
necessarily) be parallelizable.
                                                            ———— Rob Pike (Co-inventor of the Go language)
```

## 一个使用线程的例子   
```python
import threading
import itertools
import time
import sys

LUCKY = 3

class Signal:
    go = True

def spin(msg, signal):
    write = sys.stdout.write
    flush = sys.stdout.flush
    for char in itertools.cycle('|/-\\'):
        status = char + ' ' + msg
        write(status)
        flush()
        write('\x08' * len(status))
        time.sleep(0.1)
        if not signal.go:
            break
    write(' '*len(status) + '\x08'*len(status))

def slow_function():
    time.sleep(3)
    return LUCKY

def supervisor():
    signal = Signal()
    spinner = threading.Thread(target=spin, args=('thinking!', signal))
    print('spinner object:', spinner)
    spinner.start()
    result = slow_function()
    signal.go = False
    spinner.join()
    return result

def main():
    result = supervisor()
    print('Answer:', result)

if __name__ == '__main__':
    main()
```
- 使用退格符(\x08)把光标移回来
- `time.sleep()`会阻塞主线程，释放GIL，从属线程以动画的形式显示旋转指针


## 使用协程改写上面的程序
```python
import asyncio
import itertools
import sys

LUCKY = 3

@asyncio.coroutine
def spin(msg):
    write = sys.stdout.write
    flush = sys.stdout.flush
    for char in itertools.cycle('|/-\\'):
        status = char + ' ' + msg
        write(status)
        flush()
        write('\x08' * len(status))
        try:
            yield from asyncio.sleep(0.1)
        except asyncio.CancelledError:
            break
    write(' '*len(status) + '\x08'*len(status))

@asyncio.coroutine
def slow_function():
    yield from asyncio.sleep(3)
    return LUCKY

@asyncio.coroutine
def supervisor():
    spinner = asyncio.async(spin('thinking!'))
    print('spinner object:', spinner)
    result = yield from slow_function()
    spinner.cancel()
    return result

def main():
    loop = asyncio.get_event_loop()
    result = loop.run_until_complete(supervisor())
    loop.close()
    print('Answer:', result)

if __name__ == '__main__':
    main()
```
- 打算交给`asyncio`处理的协程要使用`@asyncio.coroutine`装饰。使用`@asyncio.coroutine`装饰器不是强制要求,但是强烈建议这么做,因为这样能在一众普通的函数中把协程凸显出来, 也有助于调试:如果还没从中产出值,协程就被垃圾回收了(意味着有操作未完成,因此有可能是个缺陷),那就可以发出警告。这个装饰器不会预激协程
- 使用`yield from asyncio.sleep(.1)`代替`time.sleep(.1)`, 这样的休眠不会阻塞事件循环
- 如果`spin()`函数苏醒后抛出`asyncio.CancelledError`异常, 其原因是发出了取消请求,因此退出循环
- `asyncio.async(...)`函数排定`spin`协程的运行时间,使用一个`Task`对象包装`spin`协程,并立即返回
- `slow_function()`函数是协程, 在用休眠`假装`进行`I/O`操作时, 使用`yield from`继续执行事件循环
- `yield from asyncio.sleep(3)`表达式把控制权交给主循环, 在休眠结束后恢复这个协程(这里是不是可以理解成把控制权交给主循环，然后主循环调用`asyncio.sleep()`， 结束后控制权交给`supervisor()`?) 
-  除非想阻塞主线程, 从而冻结事件循环或整个应用, 否则不要在`asyncio`协程中使用`time.sleep(...)`。如果协程需要在一段时间内什么也不做, 应该使用`yield from asyncio.sleep(DELAY)`


## 对比线程和协程两个版本的实现
- `asyncio.async`创建的`Task`对象用于驱动协程, Thread对象用于调用可调用的对象。
- 获取的`Task`对象已经排定了运行时间(例如,由`asyncio.async()`函数排定); `Thread`实例则必须调用`start()`方法, 明确告知让它运行
- 在线程版`supervisor()`函数中, `slow_function()`函数是普通的函数, 直接由线程调
用。在异步版`supervisor()`函数中, `slow_function()`函数是协程, 由`yield from`驱动。
- 没有`API`能从外部终止线程, 因为线程随时可能被中断, 导致系统处于无效状态。如果想终止任务,可以使用`Task.cancel()`实例方法, 在协程内部抛出`CancelledError`异常。协程可以在暂停的`yield`处捕获这个异常,处理终止请求。
- 协程自身就会同步, 因为在任意时刻只有一个协程运行。想交出控制权时,可以使用 `yield`或`yield from`把控制权交还调度程序。这就是能够安全地取消协程的原因:按照定义,协程只能在暂停的`yield`处取消, 因此可以处理`CancelledError`异常,执行清理操作。

## `asyncio.Future`
`asyncio.Future`类与`concurrent.futures.Future`类的接口基本一致,不过实现方式不同,不可以互换  

在`asyncio`包中, `BaseEventLoop.create_task(...)`方法接收一个协程, 排定它的运行时间, 然后返回一个`asyncio.Task`实例, 它也是`asyncio.Future`类的实例, 因为`Task`是Future 的子类,用于包装协程  

与`concurrent.futures.Future`类似, `asyncio.Future`类也提供了`done()`、`add_done_callback(...)`和`result()`等方法。`asyncio.Future`类的`result()`方法没有参数, 因此不能指定超时时间。此外,如果调用`result()`方法时`Future`还没运行完毕, 那么`result()`方法不会阻塞去等待结果, 而是抛出`asyncio.InvalidStateError`异常。

使用`yield from`处理`asyncio.Future`, 等待`Future`实例运行完毕这一步无需我们关心, 而且不会阻塞事件循环, 因为在`asyncio`包中, `yield from`的作用是把控制权还给事件循环。注意,使用`yield from`处理`Future`实例与使用`add_done_callback`方法处理协程的作用一样:延迟的操作结束后,事件循环不会触发回调对象, 而是设置它们的返回值; 而`yield from`表达式则在暂停的协程中生成返回值 , 恢复执行协程。

`asyncio.Future`类的目的是与`yield from`一起使用,所以通常不需要使用以下方法。无需调用my_future.add_done_callback(...),因为可以直接把想在期物运行结束后执行的操作放在协程中`yield from my_future`表达式的后面。这是协程的一大优势:协程是可以暂停和恢复的函数。无需调用`my_future.result()`, 因为 yield from 从期物中产出的值就是结果(例如,result = yield from my_future)。

当然,有时也需要使用`done()`、`add_done_callback(...)`和`result()`方法。但是一般情况下,asyncio.Future 对象由 yield from 驱动,而不是靠调用这些方法驱动。

## Yielding from Futures, Tasks, and Coroutines
在`asyncio`包中, `futures`和协程关系紧密, 因为可以使用`yield from`从`asyncio.Future`对象中产出结果。(即如果`foo()`是协程函数(调用后返回协程对象),抑或是返回`Future`或`Task`实例的普通函数, 那么就可以写成`res = yield from foo()`)

为了执行上述操作, 必须排定协程的运行时间, 然后使用`asyncio.Task`对象包装协程。对协程来说, 获取`Task`对象有两种主要方式:
- 使用`asyncio.async(coro_or_future, *, loop=None)`这个函数统一了协程和期物:第一个参数可以是二者中的任何一个。如果是 `Future`或`Task`对象, 那就原封不动地返回。如果是协程, 那么`async`函数会调用`loop.create_task(...)`方法创建`Task`对象。`loop=`关键字参数是可选的,用于传入事件循环; 如果没有传入, 那么`async`函数会通过调用`asyncio.get_event_loop()`函数获取循环对象。
- `BaseEventLoop.create_task(coro)`这个方法排定协程的执行时间, 返回一个`asyncio.Task`对象。如果在自定义的`BaseEventLoop`子类上调用, 返回的对象可能是外部库(如 Tornado)中与 Task 类兼容的某个类的实例。

`asyncio`包中有多个函数会自动(内部使用的是`asyncio.async()`函数)把参数指定的协程包装在`asyncio.Task`对象中,例如`BaseEventLoop.run_until_complete(...)`方法。

## 避免阻塞型调用
有两种方法能避免阻塞型调用中止整个应用程序的进程:
- 在单独的线程中运行各个阻塞型操作
- 把每个阻塞型操作转换成非阻塞的异步调用使用

多个线程是可以的, 但是各个操作系统线程(Python使用的是这种线程)消耗的内存达兆字节(具体的量取决于操作系统种类)。如果要处理几千个连接, 而每个连接都使用一个线程的话, 我们负担不起  

为了降低内存的消耗,通常使用回调来实现异步调用。这是一种低层概念,类似于所有并发机制中最古老、最原始的那种——硬件中断。使用回调时, 我们不等待响应, 而是注册一个函数, 在发生某件事时调用。这样, 所有调用都是非阻塞的。  

当然,只有异步应用程序底层的事件循环能依靠基础设置的中断、线程、轮询和后台进程等,确保多个并发请求能取得进展并最终完成,这样才能使用回调。事件循环获得响应后,会回过头来调用我们指定的回调。不过, 如果做法正确, 事件循环和应用代码共用的主线程绝不会阻塞

为了尽量提高性能, save_flag函数应该执行异步操作,可是 asyncio 包目前没有提供异步文件系统 API(Node 有)。如果这是应用的瓶颈,可以使用`loop.run_in_executor`方法(https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.BaseEventLoop.run_in_executor),在线程池中运行 save_flag函数。  

## 使用`Executor`对象, 防止阻塞事件循环 
之前的下载国旗的例子中，`save_flag()`访问本地文件系统是会阻塞客户代码与`asyncio`事件循环所共用的唯一线程，因此在保存文件时，整个应用程序都会冻结。解决这一问题的方法是使用事件循环对象`run_in_executor()`。  

`asyncio`的事件循环在背后维护着一个`ThreadPoolExecutor`对象, 我们可以调用`run_in_executor`方法, 把可调用的对象发给它执行  

## 从回调到`future`和协程
javascript中存在的`回调地狱`现象
```javascript
api_call_1(request1, function(response1){
    var requests2 = step1(response1);
    api_call_2(request2, function(response2){
        var request3 = step2(response2);
        api_call_3(request3, function(response3){
            step3(response3);
        });
    });
});
```
python中的回调地狱，链式回调
```python
def stage1(response1):
    requets2 = step1(response1)
    api_call_2(request2, stage2)

def stage2(response2):
    request3 = step2(response2)
    api_call_3(request3, stage3)

def stage(response3):
    step3(response3)
    api_call_1(request1, stage1)
```
这样操作会存在一个问题：每个函数做一部分工作, 设置下一个回调, 然后返回, 让事件循环继续运行。这样,所有本地的上下文都会丢失。执行下一个回调时(例如 stage2),就无法获取 request2 的值。如果需要那个值,那就必须依靠闭包, 或者把它存储在外部数据结构中, 以便在处理过程的不同阶段使用。  

在这个问题上,协程能发挥很大的作用。在协程中,如果要连续执行 3 个异步操作,只需使用`yield` 3 次,让事件循环继续运行。准备好结果后,调用`send()`方法, 激活协程。对事件循环来说,这种做法与调用回调类似。但是对使用协程式异步`API`的用户来说, 情况就大为不同了:3 次操作都在同一个函数定义体中,像是顺序代码,能在处理过程中使用局部变量保留整个任务的上下文
```python
@asyncio.coroutine
def three_stages(request1):
    response1 = yield from api_call_1(request1)
    request2 = step1(response1)
    response2 = yield from api_call_2(request2)
    request3 = step2(response2)
    response3 = yield from api_call_3(request3)
    step3(response3)

loop.create_task(three_stages(request1))
```

在基于回调的`API`中, 可能要为每个异步调用注册两个回调,一个用于处理操作成功时返回的结果,另一个用于处理错误。一旦涉及错误处理,回调地狱的危害程度就会迅速增大。

## 写在最后
- 这一章有用`asyncio`和`aiohttp`实现的两个小项目，一个是国旗下载，另一个是实现服务器，都有一点代码量，推荐移步到[github](https://github.com/fluentpython/example-code)上阅读
- 其次是，由于新增加的关键字`async/await`导致我不想看到书上的旧写法，以及`aiohttp`的一些方法和函数好像也有变动，最关键的是我觉得这本书在这章写得不好，至少对我这个对协程掌握不好的人很不友好。
- 后续会专门出一篇关于协程的文章，敬请关注。