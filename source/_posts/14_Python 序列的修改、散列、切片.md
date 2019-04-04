---
title: Python 序列的修改、散列、切片
tags:
- Python
- Fluent Python
comments: on
---



*在 “符号Python风格对象” 中的二维向量类型的基础，扩展n维向量，并实现基本的序列协议、切片等*

> **基本的序列协议： `__len__`, `__getitem__`**

## 1、多维Vector类

> 序列类型的构造方法最好接受可迭代对象作为参数（内置序列类型均是如此）


```python
from array import array
import reprlib
import math

class Vector:
    typecode = 'd'
    
    def __init__(self, components):
        self._components = array(self.typecode, components)
        
    def __iter__(self):
        return iter(self._components)
    
    def __repr__(self):
        components = reprlib.repr(self._components)   # array的字符串表现形式如 array([1, 2, ...])
        components = components[components.find('['):-1]
        return 'Vector({})'.format(components)
    
    def __str__(self):
        return str(tuple(self))
    
    def __bytes__(self):
        return bytes([ord(self.typecode)]) + bytes(self._components)
    
    def __eq__(self, other):
        return tuple(self) == tuple(other)
    
    def __abs__(self):
        return math.sqrt(sum(x ** 2 for x in self))
    
    def __bool__(self):
        return bool(abs(self))
    
    @classmethod
    def frombytes(cls, octets):
        typecode = chr(octets[0])
        memv = memoryview(octets[1:]).cast(typecode)
        return cls(memv)
```

- `reprlib.repr()` 方法可以获取有限长度的表现形式


```python
v1 = Vector([1, 2, 3])
print(v1)
```

    (1.0, 2.0, 3.0)



```python
Vector(range(10))
```




    Vector([0.0, 1.0, 2.0, 3.0, 4.0, ...])




```python
Vector.frombytes(bytes(v1))
```




    Vector([1.0, 2.0, 3.0])



## 2、协议和鸭子类型

> Python 中创建功能完善的序列类型无需使用继承，只需实现符合**序列协议**的方法，行为像序列那么它就是序列，即鸭子类型。   
> OOP 中，协议是非正式的接口，只有在文档中定义，在代码中不定义。    
> Python 的序列协议只需要 `__len__` 和 `__getitem__` 两个方法。

## 3、可切片的序列 Vector类

简单地，只需在 Vector 类中实现 `__len__` 和 `__getitem__` 两个方法，委托给对象中的序列属性。   
```
def __len__(self):
    return len(self._components)

def __getitem__(self, index):
    return self._components[index]
```

但是这样做有欠缺，得到的切片不是Vector类型，而是数组。    
内置的序列类型，**切片得到的都是各自类型的新实例**，所以这里不能简单地委托给数组切片。

### 3.1 切片原理


```python
class MySeq:
    def __getitem__(self, index):
        return index
```


```python
s = MySeq()
```


```python
s[1]
```




    1




```python
s[1:4]
```




    slice(1, 4, None)




```python
s[1:4:2]
```




    slice(1, 4, 2)



多维索引


```python
s[1:4:2, 9]
```




    (slice(1, 4, 2), 9)




```python
s[1:4:2, 7:9]
```




    (slice(1, 4, 2), slice(7, 9, None))



**`slice`**


```python
dir(slice)
```




    ['__class__',
     '__delattr__',
     '__dir__',
     '__doc__',
     '__eq__',
     '__format__',
     '__ge__',
     '__getattribute__',
     '__gt__',
     '__hash__',
     '__init__',
     '__init_subclass__',
     '__le__',
     '__lt__',
     '__ne__',
     '__new__',
     '__reduce__',
     '__reduce_ex__',
     '__repr__',
     '__setattr__',
     '__sizeof__',
     '__str__',
     '__subclasshook__',
     'indices',
     'start',
     'step',
     'stop']



`slice` 是内置的类型，其中 `indices` 方法，对于长度为 len 的序列，用于处理缺失索引、负数索引、长度超过目标序列的切片。   
> 如果没有底层序列类型作为依靠，使用 `indices` 方法能节省大量时间


```python
help(slice.indices)
```

    Help on method_descriptor:
    
    indices(...)
        S.indices(len) -> (start, stop, stride)
        
        Assuming a sequence of length len, calculate the start and stop
        indices, and the stride length of the extended slice described by
        S. Out of bounds indices are clipped in a manner consistent with the
        handling of normal slices.


​    


```python
# 'ABCDE'[:10:2] <==> 'ABCDE'[0:5:2] 
slice(None, 10, 2).indices(5)
```




    (0, 5, 2)




```python
# 'ABCDE'[-3:] <==> 'ABCDE'[2:5:1]
slice(-3, None, None).indices(5)
```




    (2, 5, 1)



### 3.2 处理切片的 `__getitem__` 方法


```python
import numbers

def __getitem__(self, index):
    cls = type(self)
    # 切片索引，创建一个新 Vector 实例
    if isinstance(index, slice):
        return cls(self._components[index])
    # 单个索引，只取一个分量
    elif isinstance(index, numbers.Integral):
        return self._components[index]
    # 不支持多维索引
    else:
        msg = '{cls.__name__} indices must be integers'
        raise TypeError(msg.format(cls=cls))
```

***完整代码和测试最后给出***

## 4、动态存取属性

目的：通过特定的名称访问向量的分量，如 `v.x` 代表 `v[0]`    
- `@property` 装饰器可以绑定新特性，但是要实现多个属性较麻烦。    
- `__getattr__` 特殊方法，接收属性名称的字符串形式，当属性查找失败，便会调用改方法。**仅当对象没有指定名称的属性时，才会被调用，是一种后备机制**（备胎？）


```python
# Vector 类中新增类属性 shortcut_names 和特殊方法 __getattr__

shortcut_names = 'xyzt'   # 四个字母分别代表序列的前四个元素

def __getattr__(self, name):
    cls = type(self)
    if len(name) == 1:
        pos = cls.shortcut_names.find(name)
        if 0 <= pos < len(self._components):
            return self._components[pos]
    msg = '{.__name__!r} object has no attribute {!r}'
    raise AttributeError(msg.format(cls, name))
```

如果对 xyzt 中的一个属性赋值，会导致对象拥有该属性，导致矛盾，所以应对单小写字母属性赋值需要抛出异常


```python
# Vector 类中新增特殊方法 __setattr__

def __setattr__(self, name, value):
    cls = type(self)
    
    if len(name) == 1:
        if name in cls.shortcut_names:
            error = 'readonly attribute {attr_name!r}'
        elif name.islower():
            error = "cant't set attributes 'a' to 'z' in {cls_name!r}"
        else:
            error = ''
        if error:
            msg = error.format(attr_name=name, cls_name=cls.__name__)
            raise AttributeError(msg)
    super().__setattr__(name, value)
```

> `super()` 用于动态访问超类的方法，把子类方法的某些任务委托给超类中的适当方法（多重继承导致 `super()` 的诞生）

**为了防止对象行为不一致，如果实现了 `__getattr__` 方法，也要实现 `__setattr__` 方法**

***完整代码和测试最后给出***

## 5、散列和快速等值

可散列需要实现 `__eq__` 和 `__hash__` 方法，前者已经实现，后者返回所有分量的散列值异或，使用 `reduce()` 方法。    
PS：`reduce()` 方法最好提供第三个参数，即归约的初始值，这样序列为空不会出现异常，而是返回该初始值；异或可以使用 `operator` 模块中的 `xor` 方法，避免使用 `lambda` 表达式，`operator` 模块以函数的形式提供了 Python 中的全部中缀运算符


```python
# Vector 类中新增 __hash__ 方法

from functools import reduce
import operator

# 映射归约
def __hash__(self):
    hashes = (hash(x) for x in self._components)  # hashes = map(hash, self._components)  惰性，按需产出值
    return reduce(operator.xor, hashes, 0)
```

已实现的 `__eq__` 方法是将任务委托给 `tuple`，这在 Vector 分量很多时效率很低，所以重写一个效率较高的等值方法


```python
def __eq__(self, other):
    if len(self) != len(other):
        return False
    for a, b in zip(self, other):
        if a != b:
            return False
    return True

# 或者可以用 all 方法进行归约
def __eq__(self, other):
    return len(self) == len(other) and all(a == b for a, b in zip(self, other))
```

***完整代码和测试最后给出***

## 6、格式化

自定义格式代码 'h'（避免重用内置类型支持的格式代码），重写 `__format__` 方法。与二维的极坐标类似，该自定义格式返回对象的超球体坐标（`<r, angle1, angle2, ...>`）


```python
# Vector 类中新增 __format__ 方法和辅助方法 angle 和 angles

# 超球体某个角坐标
def angle(self, n):
    r = abs(self)
    a = math.atan2(r, self[n-1])
    if n == len(self) - 1 and self[-1] < 0:
        return math.pi * 2 - a
    return a

def angles(self):
    return (self.angle(n) for n in range(1, len(self)))

def __format__(self, fmt_spec=''):
    if fmt_spec.endswith('h'):
        fmt_spec = fmt_spec[:-1]
        coords = itertools.chain([abs(self)], self.angles())
        outer_fmt = '<{}>'
    else:
        coords = self
        outer_fmt = '({})'
    components = (format(c, fmt_spec) for c in coords)
    return outer_fmt.format(', '.join(components))
```

***完整代码和测试最后给出***

# 完整代码及测试


```python
from array import array
import reprlib
import functools
import operator
import math
import numbers
import itertools

class Vector:
    typecode = 'd'
    shortcut_names = 'xyzt'

    def __init__(self, components):
        self._components = array(self.typecode, components)

    def __iter__(self):
        return iter(self._components)

    def __repr__(self):
        components = reprlib.repr(self._components)
        components = components[components.find('['):-1]
        return 'Vector({})'.format(components)

    def __str__(self):
        return str(tuple(self))

    def __bytes__(self):
        return bytes([ord(self.typecode)]) + bytes(self._components)

    def __eq__(self, other):
        return len(self) == len(other) and all(a == b for a, b in zip(self, other))

    def __hash__(self):
        hashes = (hash(x) for x in self)
        return functools.reduce(operator.xor, hashes, 0)

    def __abs__(self):
        return math.sqrt(sum(x * x for x in self))

    def __bool__(self):
        return bool(abs(self))

    def __len__(self):
        return len(self._components)

    def __getitem__(self, index):
        cls = type(self)
        if isinstance(index, slice):
            return cls(self._components[index])
        elif isinstance(index, numbers.Integral):
            return self._components[index]
        else:
            msg = '{.__name__} indices must be integers'
            raise TypeError(msg.format(cls))

    def __getattr__(self, name):
        cls = type(self)
        if len(name) == 1:
            pos = cls.shortcut_names.find(name)
            if 0 <= pos < len(self._components):
                return self._components[pos]
        msg = '{.__name__!r} objects has no attribute {!r}'
        raise AttributeError(msg.format(cls, name))

    def __setattr__(self, name, value):
        cls = type(self)
        if len(name) == 1:
            if name in cls.shortcut_names:
                error = 'readonly attribute {attr_name!r}'
            elif name.islower():
                error = "can't set attributes 'a' to 'z' in {cls_name!r}"
            else:
                error = ''
            if error:
                msg = error.format(cls_name=cls.__name__, attr_name=name)
                raise AttributeError(msg)
        super().__setattr__(name, value)

    def angle(self, n):
        r = abs(self)
        a = math.atan2(r, self[n-1])
        if n == len(self) - 1 and self[-1] < 0:
            return math.pi * 2 - a
        return a

    def angles(self):
        return (self.angle(n) for n in range(1, len(self)))

    def __format__(self, fmt_spec=''):
        if fmt_spec.endswith('h'):
            fmt_spec = fmt_spec[:-1]
            coords = itertools.chain([abs(self)], self.angles())
            outer_fmt = '<{}>'
        else:
            coords = self
            outer_fmt = '({})'
        components = (format(c, fmt_spec) for c in coords)
        return outer_fmt.format(', '.join(components))

    @classmethod
    def frombytes(cls, octets):
        typecode = chr(octets[0])
        memv = memoryview(octets[1:]).cast(typecode)
        return cls(memv)
```

### 1. 实例化测试


```python
Vector([3.1, 4.2])
```




    Vector([3.1, 4.2])




```python
Vector([3, 4, 5])
```




    Vector([3.0, 4.0, 5.0])




```python
Vector(range(10))
```




    Vector([0.0, 1.0, 2.0, 3.0, 4.0, ...])



### 2. 二维向量测试


```python
v1 = Vector([3, 4])
x, y = v1
x, y
```




    (3.0, 4.0)




```python
v1
```




    Vector([3.0, 4.0])




```python
v1_clone = eval(repr(v1))
v1 == v1_clone
```




    True




```python
print(v1)
```

    (3.0, 4.0)



```python
octets = bytes(v1)
octets
```




    b'd\x00\x00\x00\x00\x00\x00\x08@\x00\x00\x00\x00\x00\x00\x10@'




```python
abs(v1)
```




    5.0




```python
bool(v1), bool(Vector([0, 0]))
```




    (True, False)



### 3. 三维向量测试


```python
v1 = Vector([3, 4, 5])
x, y, z = v1
x, y, z
```




    (3.0, 4.0, 5.0)




```python
v1
```




    Vector([3.0, 4.0, 5.0])




```python
v1_clone = eval(repr(v1))
v1 == v1_clone
```




    True




```python
print(v1)
```

    (3.0, 4.0, 5.0)



```python
abs(v1)
```




    7.0710678118654755




```python
bool(v1), bool(Vector([0, 0, 0]))
```




    (True, False)



### 4. 多维向量测试


```python
v7 = Vector(range(7))
v7
```




    Vector([0.0, 1.0, 2.0, 3.0, 4.0, ...])



### 5. 字节测试和字节构造方法测试


```python
v1 = Vector([3, 4, 5])
v1_clone = Vector.frombytes(bytes(v1))
v1_clone
```




    Vector([3.0, 4.0, 5.0])




```python
v1 == v1_clone
```




    True



### 6. 序列测试（长度，索引，切片）


```python
v1 = Vector([3, 4, 5])
len(v1)
```




    3




```python
v1[0], v1[len(v1)-1], v1[-1]
```




    (3.0, 5.0, 5.0)




```python
v7 = Vector(range(7))
v7[-1]
```




    6.0




```python
v7[1:4]
```




    Vector([1.0, 2.0, 3.0])




```python
v7[-1:]
```




    Vector([6.0])




```python
v7[1, 2]
```


    ---------------------------------------------------------------------------
    
    TypeError                                 Traceback (most recent call last)
    
    <ipython-input-47-54e115c7e17b> in <module>()
    ----> 1 v7[1, 2]


    <ipython-input-22-677e1f3d7f64> in __getitem__(self, index)
         52         else:
         53             msg = '{.__name__} indices must be integers'
    ---> 54             raise TypeError(msg.format(cls))
         55 
         56     def __getattr__(self, name):


    TypeError: Vector indices must be integers


### 7. 动态属性测试


```python
v7 = Vector(range(10))
v7.x,  v7.y,  v7.z,  v7.t
```




    (0.0, 1.0, 2.0, 3.0)




```python
v7.k
```


    ---------------------------------------------------------------------------
    
    AttributeError                            Traceback (most recent call last)
    
    <ipython-input-49-70ed3a6fcb8f> in <module>()
    ----> 1 v7.k


    <ipython-input-22-677e1f3d7f64> in __getattr__(self, name)
         61                 return self._components[pos]
         62         msg = '{.__name__!r} objects has no attribute {!r}'
    ---> 63         raise AttributeError(msg.format(cls, name))
         64 
         65     def __setattr__(self, name, value):


    AttributeError: 'Vector' objects has no attribute 'k'



```python
v7.x = 2
```


    ---------------------------------------------------------------------------
    
    AttributeError                            Traceback (most recent call last)
    
    <ipython-input-50-bd2e4daceec7> in <module>()
    ----> 1 v7.x = 2


    <ipython-input-22-677e1f3d7f64> in __setattr__(self, name, value)
         74             if error:
         75                 msg = error.format(cls_name=cls.__name__, attr_name=name)
    ---> 76                 raise AttributeError(msg)
         77         super().__setattr__(name, value)
         78 


    AttributeError: readonly attribute 'x'



```python
v7.k = 2
```


    ---------------------------------------------------------------------------
    
    AttributeError                            Traceback (most recent call last)
    
    <ipython-input-51-b010ed8b3ae4> in <module>()
    ----> 1 v7.k = 2


    <ipython-input-22-677e1f3d7f64> in __setattr__(self, name, value)
         74             if error:
         75                 msg = error.format(cls_name=cls.__name__, attr_name=name)
    ---> 76                 raise AttributeError(msg)
         77         super().__setattr__(name, value)
         78 


    AttributeError: can't set attributes 'a' to 'z' in 'Vector'


### 8. 散列测试


```python
v1 = Vector([3, 4])
v2 = Vector([3.1, 4.2])
v3 = Vector([3, 4, 5])
v6 = Vector(range(6))
hash(v1), hash(v3), hash(v6)
```




    (7, 2, 1)




```python
# 32 位和 64 位机器上对 v2 的哈希值不一样
import sys
hash(v2) == (384307168202284039 if sys.maxsize > 2 ** 32 else 357915986)
```




    True



### 9. 格式化测试


```python
v1 = Vector([3, 4])
format(v1)
```




    '(3.0, 4.0)'




```python
format(v1, '.2f')
```




    '(3.00, 4.00)'




```python
format(v1, '.3e')
```




    '(3.000e+00, 4.000e+00)'




```python
v3 = Vector([3, 4, 5])
v7 = Vector(range(7))
format(v3), format(v7)
```




    ('(3.0, 4.0, 5.0)', '(0.0, 1.0, 2.0, 3.0, 4.0, 5.0, 6.0)')




```python
format(v1, 'h')
```




    '<5.0, 1.0303768265243125>'




```python
format(v1, '.3eh')
```




    '<5.000e+00, 1.030e+00>'




```python
format(v1, '0.5fh')
```




    '<5.00000, 1.03038>'




```python
format(v7, '.3eh')
```




    '<9.539e+00, 1.571e+00, 1.466e+00, 1.364e+00, 1.266e+00, 1.174e+00, 1.088e+00>'




```python
format(v7, '0.5fh')
```




    '<9.53939, 1.57080, 1.46635, 1.36413, 1.26610, 1.17375, 1.08802>'




```python
format(Vector([-1, -1, -1, -1]), 'h')
```




    '<2.0, 2.0344439357957027, 2.0344439357957027, 4.2487413713838835>'




```python
format(Vector([0, 1, 0, 0]), '0.5fh')
```




    '<1.00000, 1.57080, 0.78540, 1.57080>'



<br>

[md效果不好，原先是 jupyter notebook 版本](https://nbviewer.jupyter.org/github/sxnys/mypython/blob/master/FluentPython/10_Sequence_Hacking_Hashing_Slicing/1_Sequence_Hacking_Hashing_Slicing.ipynb)

