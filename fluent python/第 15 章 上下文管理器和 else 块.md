## 前言
with 语句会设置一个临时的上下文,交给上下文管理器对象控制,并且负责清理上下文。这么做能避免错误并减少样板代码,因此 API 更安全,而且更易于使用。除了自动关闭文件之外,with 块还有很多用途。

## `else`

### `for/else`
仅当`for`循环运行完毕时(即`for`循环没有被`break`语句中止)才运行`else`块。
```python
for x in range(3):
    print(x)
    if x > 100:
        break
else:
    print('x never greater than 100')
>>> 0
>>> 1
>>> 2
>>> x never greater than 100

for x in range(5):
    print(x)
    if x > 2:
        break
else:
    print('I am a dog')
>>> 0
>>> 1
>>> 2
>>> 3
```

### `while/else`
仅当`while`循环因为条件为假值而退出时(即`while`循环没有被`break`语句中止)才运行`else`块。
```python
i = 0
while i < 10:
    i += 1
    print(i)
    if i > 2:
        break
else:
    print('I am a dog')
>>> 1
>>> 2
>>> 3
```

### `try/else`
仅当try 块中没有异常抛出时才运行 else 块。[官方文档](https://docs.python.org/3/reference/compound_stmts.html)还指出`else`子句抛出的异常不会由前面的`except`子句处理。
```python
try:
    x = 1 / 0
except ZeroDivisionError:
    print('ZeroDivisionError')
else:
    print('I am a dog')

>>> ZeroDivisionError:
```

在所有情况下,如果异常或者`return`、`break`或`continue`语句导致控制权跳到了复合语句的主块之外, `else`子句也会被跳过。

ps:本书作者认为then能替代这样语境下的else, 但是我们的`Guido`极其不喜欢加入新关键字


### `EAFP`
取得原谅比获得许可容易(`easier to ask for forgiveness than permission`)。这是一种常见的 Python 编程风格,先假定存在有效的键或属性,如果假定不成立,那么捕获异常。这种风格简单明快,特点是代码中有很多 try 和 except 语句。与其他很多语言一样(如 C 语言),这种风格的对立面是`LBYL`风格。

### `LBYL`
三思而后行(`look before you leap`)。这种编程风格在调用函数或查找属性或键之前显式测试前提条件。与 `EAFP`风格相反,这种风格的特点是代码中有很多`if`语句。在多线程环境中, `LBYL`风格可能会在“检查”和“行事”的空当引入条件竞争。


## 上下文管理器和with块
上下文管理器对象存在的目的是管理`with`语句,就像迭代器的存在是为了管理`for`句一样  

`with`语句的目的是简化`try/finally`模式。这种模式用于保证一段代码运行完毕后执行某项操作,即便那段代码由于异常、`return`语句或`sys.exit()`调用而中止,也会执行指定的操作。`finally`子句中的代码通常用于释放重要的资源,或者还原临时变更的状态。

上下文管理器协议包含`__enter__`和`__exit__`两个方法。`with`语句开始运行时,会在上下文管理器对象上调用`__enter__`方法。`with`语句运行结束后,会在上下文管理器对象上调用`__exit__`方法,以此扮演`finally`子句的角色。  

```python
with open('filename.txt') as fp:
    pass
```

执行`with`后面的表达式得到的结果是`上下文管理器对象`, 而把值绑定到目标变量上(`as`子句)是在上下文管理器对象上调用`__enter__`方法的结果(即先生成一个上下文管理对象，然后调用`__enter__()`生成一个目标对象(在上面那个例子是`TextIOWrapper`), 返回绑定到`fp`上)  

`open()`数返回`TextIOWrapper`类的实例,而该实例的`__enter__`方法返回`self`。不过,`__enter__`方法除了返回上下文管理器之外,还可能返回其他对象。  

不管控制流程以哪种方式退出`with`块,都会在`上下文管理器对象`上调用`__exit__`方法,`而不是`在`__enter__`方法返回的对象上调用。  

`with`语句的`as`子句是可选的, 有些上下文管理器会返回`None`  


## 一个例子
```python
class LookingGlass:
    def __enter__(self):
        import sys
        self.original_write = sys.stdout.write
        sys.stdout.write = self.reverse_write
        return 'I LOVE YOU'
    
    def reverse_write(self, text):
        self.original_write(text[::-1])
    
    def __exit__(self, exc_type, exc_value, traceback):
        import sys
        sys.stdout.write = self.original_write
        if exc_type is ZeroDivisionError:
            print('Please DO NOT divide by zero!')
            return True

with LookingGlass() as recv:
    print(recv)
    print('test output')
>>> UOY EVOL I
>>> tuptuo tset

print('test output')
>>> test output
```
- 上下文管理器是`LookingGlass`类的实例;Python 在上下文管理器上调用`__enter__`方法,把返回结果绑定到变量`recv`上。
- 由于更改了标准输出`sys.stdout.write`, 导致输出为反的
- 执行完毕后退出作用域调用`__exit__`将之前的标准输出重新绑定回来 
- 如果`__exit__`方法返回`None`, 或者`True`之外的值, `with`块中的任何异常都会向上冒泡。

### `__exit__`的三个参数
- `exc_type`: 异常类, 如`ZeroDivisionError`
- `exc_value`: 异常类实例，有时候会有额外的信息保存在异常类实例里
- `traceback`: `traceback`对象

>在`try/finally`语句的`finally`块中调用`sys.exc_info()`(https://docs.python.org/3/library/sys.html#sys.exc_info), 得到的就是`__exit__`接收的这三个参数。鉴于`with`语句是为了取代大多数`try/finally`语句,而且通常需要调用`sys.exc_info()`来判断做什么清理操作,这种行为是合理的。


### 使用上下文管理器的场景
- sqlite3模块用于管理事务
- 在`threading`模块中用于维护锁、条件和信号
- 为`Decimal`对象的算术运算设置环境
- 为了测试临时给对象打补丁,参见 unittest.mock.patch 函数的[文档](https://docs.python.org/3/library/unittest.mock.html#patch)

## `contextlib`模块
自己定义上下文管理器类之前,先看一下 Python 标准库文档中的[29.6 contextlib — Utilities for with-statement contexts](https://docs.python.org/3/library/contextlib.html)

### `closing`
如果对象提供了`close()`方法，但没有实现`__enter__`和`__exit__`协议，那么可以使用这个`函数`构建上下文管理器

### `suppress()`
构建临时忽略指定异常的上下文管理器

### `@contextmanager`
>This function is a decorator that can be used to define a factory function for with statement context managers, without needing to create a class or separate __enter__() and __exit__() methods. 
 The function being decorated must return a generator-iterator when called. This iterator must yield exactly one value, which will be bound to the targets in the with statement’s as clause, if any.
这个装饰器把简单的生成器函数变成上下文管理器,这样就不用创建类去实现管理器协议了

来自文档的一个抽象的例子
```python
from contextlib import contextmanager

@contextmanager
def managed_resource(*args, **kwds):
    # Code to acquire resource, e.g.:
    resource = acquire_resource(*args, **kwds)
    try:
        yield resource
    finally:
        # Code to release resource, e.g.:
        release_resource(resource)

>>> with managed_resource(timeout=3600) as resource:
...     # Resource is released at the end of this block,
...     # even if code in the block raises an exception

```
在使用`@contextmanager`装饰的生成器中, `yield`语句的作用是把函数的定义体分成两部分: `yield`语句前面的所有代码在`with`块开始时(即解释器调用`__enter__`方法时)执行, `yield`语句后面的代码在`with`块结束时(即调用`__exit__`方法时)执行, `yield`产生的结果返回绑定给`as`后面的变量  

使用`@contextmanager`重新实现之前的`LookingGlass`
```python
from contextlib import contextmanager

@contextmanager
def looking_glass():
    import sys
    stdout = sys.stdout.write

    def reverse_write(text):
        stdout(text[::-1])
    
    sys.stdout.write = reverse_write
    yield 'I LOVE YOU'
    sys.stdout.write = stdout

with looking_glass() as recv:
    print(recv)
    print('test output')
>>> UOY EVOL I
>>> tuptuo tset

print('test output')
>>> test output
```
注意上面这个例子，如果在`with`块中抛出了异常, `Python`解释器会将其捕获,然后在`looking_glass`函数的`yield`表达式里再次抛出。但是, 那里没有处理错误的代码, 因此`looking_glass`函数会中止, 永远无法恢复成原来的`sys.stdout.write`方法,导致系统处于无效状态

修改后:
```python
from contextlib import contextmanager

@contextmanager
def looking_glass():
    import sys
    stdout = sys.stdout.write

    def reverse_write(text):
        stdout(text[::-1])
    
    sys.stdout.write = reverse_write
    try:
        yield 'I LOVE YOU'
    except ZeroDivisionError:
        msg = 'error'
    finally:
        sys.stdout.write = stdout
        if msg:
            print(error)

with looking_glass() as recv:
    print(recv)
    print('test output')
>>> UOY EVOL I
>>> tuptuo tset

print('test output')
>>> test output
```
ps: 这么写我还不如直接`try/except/finally`

前面说过,为了告诉解释器异常已经处理了, `__exit__`方法会返回 True,此时解释器会压制异常。如果`__exit__`方法没有显式返回一个值,那么解释器得到的是`None`, 然后向上冒泡异常。使用`@contextmanager`装饰器时,默认的行为是相反的:装饰器提供的`__exit__`方法假定发给生成器的所有异常都得到处理了, 因此应该压制异常. 如果
不想让`@contextmanager`压制异常, 必须在被装饰的函数中显式重新抛出异常。  

使用`@contextmanager`装饰器时,要把`yield`语句放在`try/finally`语句中(或者放在`with`语句中),这是无法避免的,因为我们永远不知道上下文管理器的用户会在`with`块中做什么  


### `ContextDecorator`
这是个基类,用于定义基于类的上下文管理器。这样实现的上下文管理器可以当成装饰器来使用
```python
from contextlib import ContextDecorator

class mycontext(ContextDecorator):
    def __enter__(self):
        print('Starting')
        return self

    def __exit__(self, *exc):
        print('Finishing')
        return False

@mycontext()
def function():
    print('The bit in the middle')

function()
>>> Starting
>>> The bit in the middle
>>> Finishing

with mycontext():
    print('The bit in the middle')

>>> Starting
>>> The bit in the middle
>>> Finishing
```

### `ExitStack`
这个上下文管理器能进入多个上下文管理器。`with`块结束时, `ExitStack`按照后进先出的顺序调用栈中各个上下文管理器的`__exit__`方法。如果事先不知道 with 块要进入多少个上下文管理器,可以使用这个类。例如,同时打开任意一个文件列表中的所有文件。


## 使用一个上下文管理器实现文件的同时原地读与写
作者`Martijn Pieters`, [博客地址](https://www.zopatista.com/python/2013/11/26/inplace-file-rewriting/)  
使用场景
```python
import csv

with inplace(csvfilename, 'r', newline='') as (infh, outfh):
    reader = csv.reader(infh)
    writer = csv.writer(outfh)

    for row in reader:
        row += ['new', 'coloumn']
        writer.writerow(row)
```


`inplace()`的实现
```python
from contextlib import contextmanager
import io
import os


@contextmanager
def inplace(filename, mode='r', buffering=-1, encoding=None, errors=None,
            newline=None, backup_extension=None):
    """Allow for a file to be replaced with new content.

    yields a tuple of (readable, writable) file objects, where writable
    replaces readable.

    If an exception occurs, the old file is restored, removing the
    written data.

    mode should *not* use 'w', 'a' or '+'; only read-only-modes are supported.

    """

    # move existing file to backup, create new file with same permissions
    # borrowed extensively from the fileinput module
    if set(mode).intersection('wa+'):
        raise ValueError('Only read-only file modes can be used')

    backupfilename = filename + (backup_extension or os.extsep + 'bak')
    try:
        os.unlink(backupfilename)
    except os.error:
        pass
    os.rename(filename, backupfilename)
    readable = io.open(backupfilename, mode, buffering=buffering,
                       encoding=encoding, errors=errors, newline=newline)
    try:
        perm = os.fstat(readable.fileno()).st_mode
    except OSError:
        writable = open(filename, 'w' + mode.replace('r', ''),
                        buffering=buffering, encoding=encoding, errors=errors,
                        newline=newline)
    else:
        os_mode = os.O_CREAT | os.O_WRONLY | os.O_TRUNC
        if hasattr(os, 'O_BINARY'):
            os_mode |= os.O_BINARY
        fd = os.open(filename, os_mode, perm)
        writable = io.open(fd, "w" + mode.replace('r', ''), buffering=buffering,
                           encoding=encoding, errors=errors, newline=newline)
        try:
            if hasattr(os, 'chmod'):
                os.chmod(filename, perm)
        except OSError:
            pass
    try:
        yield readable, writable
    except Exception:
        # move backup back
        try:
            os.unlink(filename)
        except os.error:
            pass
        os.rename(backupfilename, filename)
        raise
    finally:
        readable.close()
        writable.close()
        try:
            os.unlink(backupfilename)
        except os.error:
            pass
```

