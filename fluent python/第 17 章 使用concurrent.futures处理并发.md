## 网络下载的三种风格
为了高效处理网络`I/O`, 需要使用并发, 因为网络有很高的延迟, 所以为了不浪费`CPU`周期去等待, 最好在收到网络响应之前做些其他的事。

### 按顺序下载
```python
import os
import time
import sys
import requests

COUNTRIES = 'CN IN US ID BR PK NG BD RU JP MX PH VN ET EG DE IR TR CD FR'.split()
BASE_URL = 'http://flupy.org/data/flags'
BASE_DIR = os.path.abspath(os.path.dirname(__file__))
FLODER = os.path.join(BASE_DIR, 'downloads')

def mk_dir():
    if os.path.exists(FLODER):
        pass
    else:
        os.makedirs(FLODER)

def save(img, filename):
    path = os.path.join(FLODER, filename)
    with open(path, 'wb') as fp:
        fp.write(img)
    

def get_flag(cc):
    url = '{}/{cc}/{cc}.gif'.format(BASE_URL, cc=cc.lower())
    rsp = requests.get(url)
    return rsp.content

def show(text):
    print(text, end=' ')
    sys.stdout.flush()

def download_many(countries):
    for c in countries:
        image = get_flag(c)
        show(c)
        save(image, c.lower()+'.gif')
    return len(countries)

def main():
    st = time.time()
    mk_dir()
    count =download_many(COUNTRIES)
    ed = time.time()
    msg = '\n{} flags download in {:.2f}s'
    print(msg.format(count, ed-st))

if __name__ == '__main__':
    main()
```
- `sys.stdout.flush()`的作用是强制刷新`stdout`缓冲区，将其输出, 这样就可以在屏幕上实时显示输出信息，在Python中得这么做,因为正常情况下, 遇到换行才会刷新`stdout`缓冲。(可以尝试注释后运行看区别)

### 使用`concurrent.futures`模块下载
The main features of the `concurrent.futures` package are the `ThreadPoolExecutor` and `ProcessPoolExecutor` classes, which implement an interface that allows you to submit callables for execution in different threads or processes, respectively. The classes
manage an internal pool of worker threads or processes, and a queue of tasks to be
executed. 
```python
import os
import time
import sys
import requests

from concurrent import futures

COUNTRIES = 'CN IN US ID BR PK NG BD RU JP MX PH VN ET EG DE IR TR CD FR'.split()
BASE_URL = 'http://flupy.org/data/flags'
BASE_DIR = os.path.abspath(os.path.dirname(__file__))
FLODER = os.path.join(BASE_DIR, 'downloads')
MAX_WORKERS = 20

def mk_dir():
    if os.path.exists(FLODER):
        pass
    else:
        os.makedirs(FLODER)

def save(img, filename): 
    path = os.path.join(FLODER, filename)
    with open(path, 'wb') as fp:
        fp.write(img)

def show(text):
    print(text, end=' ')
    sys.stdout.flush()
    

def download(country):
    url = '{}/{country}/{country}.gif'.format(BASE_URL, country=country.lower())
    rsp = requests.get(url)
    save(rsp.content, country.lower()+'.gif')
    show(country)
    return country

def main():
    mk_dir()
    workers = min(MAX_WORKERS, len(COUNTRIES))

    st = time.time()
    with futures.ThreadPoolExecutor(workers) as executor:
        res = executor.map(download, COUNTRIES)
    ed = time.time()

    msg = '\n{} flags download in {:.2f}s'
    print(msg.format(len(list(res)), ed-st))

if __name__ == '__main__':
    main()
```
- `executor.__exit__`方法会调用`executor.shutdown(wait=True)`方法, 它会在所有线程都执行完毕前`阻塞`。
- `map`方法的作用与内置的`map`函数类似, 不同的是，`download()`函数会在多个线程中被`并发`调用; `map()`方法返回一个生成器, 因此可以迭代, 获取各个函数返回的值
- map()返回值是一个迭代器, 迭代器的__next__ 方法调用各个`Future`的`result()`方法,因此我们得到的是各个`Future`的结果,而非`Future`本身。 (这里是翻译的问题，但是可以明确不管它具体是生成器还是迭代器，都有`__next__`方法，返回的是`Futures.result`)


## `Futures`
ps: 这里不好理解，找来了英文原版对照阅读可能会容易理解些  
Futures are essential components in the internals of `concurrent.futures` and of `asyncio`, but as users of these libraries we sometimes don’t see them  

As of Python 3.4, there are two classes named Future in the standard library: concurrent.futures.Future and asyncio.Future . They serve the same purpose: an instance of either Future class represents a deferred computation that may or may not have completed. This is similar to the Deferred class in Twisted, the Future class in Tornado, and Promise objects in various JavaScript libraries.  

Futures encapsulate pending operations so that they can be put in queues, their state of completion can be queried, and their results (or exceptions) can be retrieved when available.

An important thing to know about futures in general is that you and I should not create them: they are meant to be instantiated exclusively by the concurrency framework, be it `concurrent.futures` or `asyncio`. It’s easy to understand why: a `Future` represents something that will eventually happen, and the only way to be sure that something will happen is to schedule its execution. Therefore, `concurrent.futures.Future` instances are created only as the result of scheduling something for execution with a `concurrent.futures.Executor` subclass. For example, the Executor.submit() method takes a callable, schedules it to run, and returns a `future`.

Client code is not supposed to change the state of a future: the concurrency framework changes the state of a future when the computation it represents is done, and we can’t control when that happens.

Both types of Future have a .done() method that is nonblocking and returns a Boolean
that tells you whether the callable linked to that future has executed or not. Instead of asking whether a future is done, client code usually asks to be notified. That’s why both `Future` classes have an `.add_done_callback()` method: you give it a callable, and the callable will be invoked with the future as the single argument when the future is done.

There is also a .result() method, which works the same in both classes when the future is done: it returns the result of the callable, or re-raises whatever exception might have been thrown when the callable was executed. However, when the future is not done, the behavior of the result method is very different between the two flavors of Future . In a concurrency.futures.Future instance, invoking `f.result()` will block the caller’s thread until the result is ready. An optional timeout argument can be passed, and if the future is not done in the specified time, a TimeoutError exception is raised. In “asyncio.Future: Nonblocking by Design” on page 545, we’ll see that the `asyncio.Future.result` method does not support timeout, and the preferred way to get the result of futures in that library is to use yield from —which doesn’t work with `concurrency.futures.Future` instances.

对比如下(我删减了部分)  
- 从`Python3.4`起, 标准库中有两个名为`Future`的类:`concurrent.futures.Future`和`asyncio.Future`。  
- 两个`Future`类的实例都表示可能已经完成或者尚未完成的延迟计算，它们背后的思想是封装待完成的操作, 使得可以放入队列, 完成的状态可以查询,得到结果(或抛出异常)后可以获取结果(或异常)。  
- 与`Twisted`引擎中的`Deferred`类、`Tornado`框架中的`Future`类, 以及多个`JavaScript`库中的`Promise`对象类似。
- 通常情况下自己不应该创建`futures`,而只能由并发框架(`concurrent.futures`或`asyncio`)实例化。原因很简单:`futures`表示终将发生的事情,而确定某件事会发生的唯一方式是执行的时间已经排定。因此,只有排定把某件事交给`concurrent.futures.Executor`子类处理时,才会创建`concurrent.futures.Future`实例。例如, `Executor.submit()`方法的参数是一个可调用的对象,调用这个方法后会为传入的可调用对象排期, 并返回一个`Future`。
- 这两种`Future`都有`.done()`方法, 这个方法不阻塞, 返回值是布尔值, 指明`future`链接的可调用对象是否已经执行。客户端代码通常不会询问`Future`是否运行结束,而是会等待通知。因此, 两个`Future`类都有`.add_done_callback()`方法:这个方法只有一个参数, 类型是可调用的对象(`callable`), 运行结束后会调用指定的可调用对象。
- 此外,还有`.result()`方法。在`future`运行结束后调用的话,这个方法在两个`Future`类中的作用相同:返回可调用对象的结果,或者重新抛出执行可调用的对象时抛出的异常。可是,如果`future`没有运行结束, result 方法在两个`Future` 类中的行为相差很大。对`concurrency.futures.Future`实例来说, 调用`f.result()`方法会阻塞调用方所在的线程,直到有结果可返回。此时,`result`方法可以接收可选的`timeout`参数, 如果在指定的时间内`future`没有运行完毕,会抛出 `TimeoutError`异常。`asyncio.Future.result`方法不支持设定超时时间,在那个库中获取`future`的结果最好使用`yield from`结构。不过,对`concurrency.futures.Future`实例不能这么做。

ps: 我觉得16章后面(使用了那个抽象例子，但又没有说清楚)以及17章这里都写得不够好，不拿出具体的例子就开始长篇大论`Future`的一些方法和表现，让人看了觉得有点莫名其妙。可能是这本书不是面向萌新的，要求有一定的理解了，我在看到这本书的时候才第一次接触到`Future`。但是`Future`的使用方法和使用`threading`进行编程时类似，可以类比  


```python
import os
import time
import sys
import requests

from concurrent import futures

COUNTRIES = 'CN IN US ID BR PK NG BD RU JP MX PH VN ET EG DE IR TR CD FR'.split()
BASE_URL = 'http://flupy.org/data/flags'
BASE_DIR = os.path.abspath(os.path.dirname(__file__))
FLODER = os.path.join(BASE_DIR, 'downloads')
MAX_WORKERS = 20

def mk_dir():
    if os.path.exists(FLODER):
        pass
    else:
        os.makedirs(FLODER)

def save(img, filename): 
    path = os.path.join(FLODER, filename)
    with open(path, 'wb') as fp:
        fp.write(img)

def show(text):
    print(text, end=' ')
    sys.stdout.flush()
    

def download(country):
    url = '{}/{country}/{country}.gif'.format(BASE_URL, country=country.lower())
    rsp = requests.get(url)
    save(rsp.content, country.lower()+'.gif')
    show(country)
    return country

def main():
    mk_dir()
    workers = min(MAX_WORKERS, len(COUNTRIES))

    st = time.time()


    with futures.ThreadPoolExecutor(workers) as executor:
        todo = list()
        for country in COUNTRIES:
            future = executor.submit(download, country)
            todo.append(future)
        
        for future in futures.as_completed(todo):
            res = future.result()
            # use res do something

    ed = time.time()

    msg = '\n{} flags download in {:.2f}s'
    print(msg.format(len(list(res)), ed-st))

if __name__ == '__main__':
    main()
```
- 在这个示例中调用`future.result()`方法绝不会阻塞, 因为`future`由`as_completed`函数产出(), (即已经执行完成了？)
- `as_completed`函数只返回已经运行结束的`future`

## 阻塞型`I/O`和`GIL`
`CPython`解释器本身就不是线程安全的, 因此有全局解释器锁(`GIL`),一次只允许使用一个线程执行Python字节码。因此, 一个Python进程通常不能同时使用多个CPU核心。 (这是`CPython`解释器的局限, 与Python语言本身无关。`Jython`和`IronPython`没有这种限制。不过, 目前最快的`Python`解释器`PyPy`也有`GIL`。  

标准库中所有执行阻塞型 I/O 操作的函数,在等待操作系统返回结果时都会释放`GIL`。这意味着在Python语言这个层次上可以使用多线程,而`I/O`密集型Python程序能从中受益:一个Python线程等待网络响应时, 阻塞型`I/O`函数会释放`GIL`, 再运行一个线程。  

Python 标准库中的所有阻塞型 I/O 函数都会释放 GIL,允许其他线程运行。`time.sleep()`函数也会释放`GIL`。因此, 尽管有，`GIL`Python 线程还是能在`I/O`密集型应用中发挥作用  


## 使用`concurrent.futures`模块绕开`GIL`
`concurrent.futures`模块的[文档](https://docs.python.org/3/library/concurrent.futures.html)副标题是“Launching parallel
tasks”(执行并行任务)。这个模块实现的是真正的并行计算,因为它使用`ProcessPoolExecutor`类把工作分配给多个`Python`进程处理。因此,如果需要做CPU密集型处理, 使用这个模块能绕开GIL, 利用所有可用的CPU核心。  

`ProcessPoolExecutor`和`ThreadPoolExecutor`类都实现了通用的`Executor`接口,因此使用`concurrent.futures`模块能特别轻松地把基于线程的方案转成基于进程的方案。

直接将
```python
def main():
    mk_dir()
    workers = min(MAX_WORKERS, len(COUNTRIES))

    st = time.time()
    with futures.ThreadPoolExecutor(workers) as executor:
        res = executor.map(download, COUNTRIES)
    ed = time.time()

    msg = '\n{} flags download in {:.2f}s'
    print(msg.format(len(list(res)), ed-st))
```
改为
```python
def main():
    mk_dir()
    workers = min(MAX_WORKERS, len(COUNTRIES))

    st = time.time()
    with futures.ProcessPoolExecutor(workers) as executor:
        res = executor.map(download, COUNTRIES)
    ed = time.time()

    msg = '\n{} flags download in {:.2f}s'
    print(msg.format(len(list(res)), ed-st))
```

ProcessPoolExecutor 的价值体现在 CPU 密集型作业上  

## 测试`Executor.map`方法
```python
from time import sleep, strftime
from concurrent import futures

def display(*args):
    print(strftime('[%H:%M:%S]'), end=' ')
    print(*args)

def loiter(n):
    msg = '{}loiter({}): doing nothing for {}s...'
    display(msg.format('\t'*n, n, n))
    sleep(n)
    msg = '{}loiter({}): done.'
    display(msg.format('\t'*n, n))
    return n * 10

def main():
    executor = futures.ThreadPoolExecutor(max_workers=3)
    results = executor.map(loiter, range(5))
    display('results:', results)
    display('Waiting for individual results:')
    for i, result in enumerate(results):
        display('result {}: {}'.format(i, result))

main()
>>> [08:13:52] loiter(0): doing nothing for 0s...
    [08:13:52] loiter(0): done.
    [08:13:52]      loiter(1): doing nothing for 1s...
    [08:13:52]              loiter(2): doing nothing for 2s...
    [08:13:52] results: <generator object Executor.map.<locals>.result_iterator at 0x7f831942bf10>
    [08:13:52]                      loiter(3): doing nothing for 3s...
    [08:13:52] Waiting for individual results:
    [08:13:52] result 0: 0
    [08:13:53]      loiter(1): done.
    [08:13:53]                              loiter(4): doing nothing for 4s...
    [08:13:53] result 1: 10
    [08:13:54]              loiter(2): done.
    [08:13:54] result 2: 20
    [08:13:55]                      loiter(3): done.
    [08:13:55] result 3: 30
    [08:13:57]                              loiter(4): done.
    [08:13:57] result 4: 40
```
- 把五个任务提交给`executor`(因为只有3个线程,所以只有3个任务会立即开始: `loiter(0)`、`loiter(1)`和`loiter(2))`;这是`非阻塞调用`。
- `for`循环中的`enumerate()`函数会隐式调用`next(results)`, 这个函数又会在(内部)表示第一个任务(loiter(0))的`future`上调用 `result()`方法。result 方法会阻塞, 直到`future`运行结束, 因此这个循环每次迭代时都要等待下一个结果做好准备(注意输出)
- `Executor.map`函数易于使用, 有一个特性:这个函数返回结果的顺序与调用开始的顺序一致。如果第一个调用生成结果用时10秒, 而其他调用只用 1 秒, 代码会阻塞 10 秒, 获取map方法返回的生成器产出的第一个结果。在此之后,获取后续结果时不会阻塞, 因为后续的调用已经结束

这章还讲了在使用`futures`时的异常处理，大同小异  

ps: 这一章我感觉啥都没有学会, 去看[文档](https://docs.python.org/3/library/concurrent.futures.html)吧，盆友  


