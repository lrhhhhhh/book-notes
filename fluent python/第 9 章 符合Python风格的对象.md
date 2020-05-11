## 前言
得益于Python 数据模型,自定义类型的行为可以像内置类型那样自然。实现如此自然的行为,靠的不是继承,而是鸭子类型`(duck typing)`

## 对象的表示形式
python提供了两种对象表示形式
- `repr()`以便于开发者理解的方式返回对象的字符串表示形式
- `str()`以便于用户理解的方式返回对象的字符串表现形式

这一章的精华就在下面的例子里:
```python
from array import array
from math import sqrt

class Vector2d:
    __slots__ = ('__x', '__y')

    typecode = 'd'

    def __init__(self, x:float, y:float):
        self.__x = float(x)
        self.__y = float(y)

    @property
    def x(self):
        return self.__x

    @property
    def y(self):
        return self.__y

    def __iter__(self):
        return (_ for _ in (self.x, self.y))

    def __repr__(self):
        return 'Vector2d({x}, {y})'.format(x=self.x, y=self.y)
    
    def __str__(self):
        return '({x}, {y})'.format(x=self.x, y=self.y)
    
    def __eq__(self, rhs):
        return self.x==rhs.x and self.y==rhs.y

    def __bytes__(self):
        return (bytes([ord(self.typecode)]) + bytes(array(self.typecode, self))) 

    def __abs__(self):
        return sqrt(self.x**2 + self.y**2)
    
    def __bool__(self):
        return self.x != 0 and self.y != 0
    
    def __hash__(self):
        return hash(self.x) ^ hash(self.y)

    @classmethod
    def frombytes(cls, octets):
        typecode = chr(octets[0])
        memv = memoryview(octets[1:]).cast(typecode)
        return cls(*memv)


v1 = Vector2d(1, 2)
v2 = Vector2d(1, 2)
print(v1 == v2)
>>>True

print(v1)
>>>(1.0, 2.0)

print(abs(Vector2d(3, 4)))
>>>5.0

v0 = Vector2d(0, 0)
print(bool(v0))
>>>False

x, y = v0
print(x, y)
>>>0.0 0.0

print(bytes(v0))
>>>b'd\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'

s = {v0, v1}
print(s)
>>>{Vector2d(0.0, 0.0), Vector2d(1.0, 2.0)}

```
- `typdecode='d'`约定`Vector2d`的`x`, `y`为`double`类型  
- 定义`__iter__()`，变成可迭代对象，继而支持拆包操作  
- `math.hypot(x, y)`返回`sqrt(x*x+y*y)`  
- `array(typecode, self)`的实现需要先实现`__iter__()`
- `frombytes()`费解
- 实现`__hash__()`以支持散列(`__eq__()`已经实现) 
- 用装饰器`@property`实现属性的只读
 
## `@classmethod`和`@staticmethod`的区别
`@staticmethod`和`@classmethod`都可以直接类名.方法名()来调用
`@classmethod`持有cls参数，可以来调用类的属性，类的方法，实例化对象等
```python
class Demo:
    @classmethod
    def klassmeth(*args):
        return args
    
    @staticmethod
    def statmeth(*args):
        return args

print(Demo.klassmeth())
>>>(<class '__main__.Demo'>,)
print(Demo.statmeth())
>>>()
```
可以看到第一个函数返回了`cls`的内容

## 格式化显示

## 私有属性和受保护的属性
python没有真正意义上的私有属性，只是约定俗成

## 使用`__slots__`类属性节省空间
默认情况下,Python在各个实例中使用字典`__dict__`存储实例属性。为了使用底层的散列表提升访问速度,字典会消耗大量内存。如果要处理数百万个属性不多的实例,通过`__slots__`类属性,能节省大量内存,方法是让解释器在元组中存储实例属性,而不用字典
```python
class Vector2d:
    __slots__ = ('__x', '__y')

    typecode = 'd'

    def __init__(self, x:float, y:float):
        self.__x = float(x)
        self.__y = float(y)
```

在类中定义`__slots__`属性之后,实例不能再有`__slots__`中所列名称之外的其他属性。这只是一个副作用,不是`__slots__`存在的真正原因。不要使用`__slots__`属性禁止类的用户新增实例属性。`__slots__`是用于优化的,不是为了约束程序员。  

此外不要把`__dict__`添加到`__slots__`  

还有一个实例属性可能需要注意,即`__weakref__`属性,为了让对象支持弱引用,必须有这个属性。用户定义的类中默认就有`__weakref__`属性。可是,如果类中定义了`__slots__`属性,而且想把实例作为弱引用的目标,那么要把`__weakref__`添加到`__slots__`中。

## 覆盖类属性
