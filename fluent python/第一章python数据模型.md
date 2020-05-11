## namedtuple
第二个参数传入一个字符串, namedtuple将创建一个Point类，且这个类有x和y这两个属性
```python
import collections
Point = collections.namedtuple('Point', 'x y')
O = Point(0, 0)
print(O)
```

## 特殊方法一览


## `__repr__` 和 `__str__`的区别
- `__repr__` 返回的字符串应该准确、无歧义并且尽可能的表达出特定对象的特征，如`Point(1, 2)`, 即包含特定属性值
- `__str__`是在`print()`以及`str()`的时候使用
- 当一个类没有`__str__`函数时，将调用`__repr__`


## 自定义类的布尔值
默认情况下我们自己定义的类的实例总被认为是`True`的，除非这个类对`__bool__`或者`__len__`函数有自己的实现

## `len()`为什么不是一个普通方法
`len()`之所以不是一个普通方法是为了让python自带的数据结构可以走后门(CPython直接从一个C结构体读取对象的长度)

