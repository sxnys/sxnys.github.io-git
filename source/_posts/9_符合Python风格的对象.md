---
title: 符合Python风格的对象
tags:
- Python
- Fluent Python
comments: on
---

----

基于 Python 的数据模型，自定义类型可以实现和内置类型一样自然的行为，实际上靠的是 **鸭子类型**。
> 鸭子类型：按照预定行为实现对象所需的方法

## **0、最初的Vector类**


```python
from math import hypot 	 # 两个数平方和开方

class Vector():
    def __init__(self, x=0, y=0):
        self.x = x
        self.y = y

    def __repr__(self):     # 字符串表示形式
        return 'Vector(%r, %r)' % (self.x, self.y)

    def __abs__(self):
        return hypot(self.x, self.y)

    def __bool__(self):
        return bool(abs(self))
        # OR return bool(self.x or self.y)

    def __add__(self, other):
        return Vector(self.x + other.x, self.y + other.y)

    def __mul__(self, scalar):
        return Vector(self.x * scalar, self.y * scalar)
```

----

## **1、对象表现形式**

Python 提供了两种获取对象的**字符串表现形式**的标准方式：
> `repr()` ：便于开发者理解的方式返回对象字符串表现形式，由 `__repr__` 特殊方法提供支持     
> `str()` ：便于用户理解的方式返回对象字符串表现形式，由 `__str__` 特殊方法提供支持

为了给对象提供其他的表现形式，还涉及到另外两个特殊方法：
> `__bytes__` ：与 `__str__` 类似；`bytes()` 调用它获取对象的 **字节序列表现形式**   
> `__format__` ：被内置的 `format()` 函数和 `str.format()` 方法调用，使用特殊的格式代码显示对象的字符串表现形式

! PS:
> Python3 中，`__repr__`、`__str__`、`__format__` 都必须返回 **Unicode 字符串** (`str` 类型)；`__bytes__` 返回 **字节序列** (`bytes` 类型)

----

## **2、Vector v0**


```python
from array import array
import math

class Vector2d:
    typecode = 'd'
    
    # 尽早转换为float，尽早捕获错误，防止传入非法参数
    def __init__(self, x, y):
        self.x = float(x)
        self.y = float(y)
        
    # 实现可迭代，用于拆包
    def __iter__(self):
        return (i for i in (self.x, self.y))    # OR yield self.x; yield self.y
    
    # !r 获取各个分量的表现形式
    def __repr__(self):
        class_name = type(self).__name__     # OR self.__class__.__name__
        return '{}({!r}, {!r})'.format(class_name, *self)
    
    # 因为可迭代，可以根据实例得到元组
    def __str__(self):
        return str(tuple(self))
    
    def __bytes__(self):
        return bytes([ord(self.typecode)]) + bytes(array(self.typecode, self))   # 可迭代，可以根据实例得到数组
        
    def __eq__(self, other):
        return tuple(self) == tuple(other)
    
    def __abs__(self):
        return math.hypot(*self)
    
    def __bool__(self):
        return bool(abs(self))
    
    @classmethod   # 定义类方法：从字节序列转换成该类的实例
    def frombytes(cls, octets):
        typecode = chr(octets[0])
        memv = memoryview(octets[1:]).cast(typecode)    # 传入 octets 字节序列创建一个内存视图，使用 typecode 转换
        return cls(*memv)                          # 拆包内存视图得到一对参数传入构造方法
```


```python
v1 = Vector2d(3, 4)
```


```python
# 无需调用读值方法，直接通过属性访问 Vertor 实例的分量
print(v1.x, v1.y)
```
	# Out
    3.0 4.0
    


```python
# 因为可迭代，可以拆包成变量元组
x, y = v1
x, y
```



	# Out
    (3.0, 4.0)




```python
# repr 字符串表现形式
v1
```



	# Out
    Vector2d(3.0, 4.0)




```python
# 使用 eval 表明 repr 函数调用实例得到的是对构造方法的准确表述
v1_clone = eval(repr(v1))
v1 == v1_clone
```



	# Out
    True




```python
# str 字符串表现形式
print(v1)
```
	# Out
    (3.0, 4.0)
    


```python
# 二进制表现形式
bytes(v1)
```



	# Out
    b'd\x00\x00\x00\x00\x00\x00\x08@\x00\x00\x00\x00\x00\x00\x10@'




```python
# Vertor模值
abs(v1)
```



	# Out
    5.0




```python
# 布尔值
bool(v1), bool(Vector2d(0, 0))
```



	# Out
    (True, False)




```python
# 字节序列转换成实例
v2 = Vector2d.frombytes(bytes(v1))
v2
```



	# Out
    Vector2d(3.0, 4.0)



----

## **3、`classmethod` 和 `staticmethod`**

两者都是装饰器
> `classmethod` ：定义操作类的方法（类方法），而不是操作实例；类方法的第一个参数是类本身；最常见的用途是定义备选构造方法（Vector v0 中的`frombytes` 方法）    
> `staticmethod` ：静态方法，其实就是普通函数，只不过在类的定义体中定义，而不在模块层定义


```python
class Demo:
    @classmethod
    def klassmeth(*args):
        return args
    
    @staticmethod
    def statmeth(*args):
        return args
```


```python
# 类方法的第一个参数始终是类本身
Demo.klassmeth(), Demo.klassmeth('spam')
```



	# Out
    ((__main__.Demo,), (__main__.Demo, 'spam'))




```python
# 静态方法的行为和普通函数相似
Demo.statmeth(), Demo.statmeth('spam')
```


	
	# Out
    ((), ('spam',))



----

## **4、格式化显示**

- 内置的 `format()` 函数和 `str.format()` 方法把各个类型的格式化方式委托给相应的 `__format__(format_spec)` 方法。    
- `format_spec` —— 格式说明符，使用 **格式规范微语言**（Format Specification Mini-Language）表示法：
> `format(obj, format_spec)` 的第二个参数      
> `str.format()` 方法的格式字符串中，`{}` 里冒号之后的部分    

它和 % 格式化方法 % 后面的部分基本一样，但是有所不同（如对齐方式）       
详细的格式化符号及用法见 [官方文档](https://docs.python.org/3/library/string.html#formatstrings "Format String Syntax") 和 [fat39 的博客](https://www.cnblogs.com/fat39/p/7159881.html "Python 格式化输出")


```python
brl = 1 / 2.43
brl
```



	# Out
    0.4115226337448559




```python
format(brl, '0.4f')   # 0可以省略
```



	# Out
    '0.4115'




```python
'1 BRL = {rate:^15.2e} USD'.format(rate=brl)
```



	# Out
    '1 BRL =    4.12e-01     USD'



上例中的 `0.4f` 和 `^15.2e`（取小数点后两位，科学计数法表示，居中并占15个位置单位）就是 `format_spec`

> 类似于 `{0.attr:5.3e}` 这样的格式字符串包括两个部分：     
> - 冒号前的是 **字段名**，可以是数字表示，也可以是有具体含义的名称，如果它是某个类的实例并且要使用它的某个属性，可以使用 `.` 来获取
> - 冒号后的就是 **格式说明符** (`format_spec`)

#### **4 种十进制转十六进制的写法**


```python
# 1. hex 内置函数
hex(100)
```



	# Out
    '0x64'




```python
# 2. % 格式化
'%x' % 100
```



	# Out
    '64'




```python
# 3. format 函数
format(100, 'x')
```



	# Out
    '64'




```python
# 4. str.format() 方法
'{:x}'.format(100)
```



	# Out
    '64'



二进制、八进制、十进制、十六进制 之间都可以用上述类似的方法相互转换。格式规范微语言为这些内置类型提供了专用的表示代码（如十六进制的 x）

> 格式规范微语言是可扩展的，每个类可以自行决定如何解释 `format_spec` 参数      
> 如 `datetime` 模块中的类，它们的 `__format__` 方法使用的格式代码和 `strftime()` 函数一样


```python
from datetime import datetime
now = datetime.now()
```


```python
format(now, '%Y-%m-%d  %H:%M:%S')
```



	# Out
    '2018-12-28  20:52:25'




```python
"It's now {:%I:%M %p}".format(now) 
```



	# Out
    "It's now 08:52 PM"




```python
now.strftime('Today is %D')
```


		
	# Out
    'Today is 12/28/18'



> 如果类没有定义 `__format__` 方法，从 `object` 继承的方法会返回 `str(my_object)`，但是如果传入格式说明符就会抛出 `TypeError`


```python
format(v1)
```



	# Out
    '(3.0, 4.0)'




```python
format(v1, '.3f')
```

	# Out
    ---------------------------------------------------------------------------

    TypeError                                 Traceback (most recent call last)

    <ipython-input-87-9f7764dc3848> in <module>()
    ----> 1 format(v1, '.3f')
    

    TypeError: unsupported format string passed to Vector2d.__format__


> 自定义的格式代码选择字母时应该避免其他类型用过的     
> - 整数使用的代码：`'bcdoXn'`
> - 浮点数使用的代码：`'eEfFgGn%'`
> - 字符串使用的代码：`'s'`

在现有的 Vector 类的基础上实现自定义格式化，包括两点：
1. 一般的格式化，分别进行向量两个分量的格式化，返回形式 `(fmt_x, fmt_y)`
2. 格式化为极坐标，选用代码 `'p'`，返回形式 `<r, θ>`（ `r` 是模长，即已经实现的 `__abs__` 方法；`θ` 是弧度）


```python
# 计算弧度
def angle(self):
    return math.atan2(self.y, self.x)

def __format__(self, fmt_spec=''):
    # 格式化为极坐标
    if fmt_spec.endwith('p'):
        fmt_spec = fmt_spec[:-1]
        coords = (abs(self), self.angle())
        outer_fmt = '<{}, {}>'
    else:
        coords = self
        outer_fmt = '({}, {})'
    components = (format(c, fmt_spec) for c in coords)
    return outer_fmt.format(*components)
```

***关于自定义格式化的完整代码和测试，和之后的内容一起给出（不然代码会重复冗长）***

----

## **5、可散列的Vector**

> 类的实例可散列，需要满足以下三个条件：   
> 1. 实例不可变
> 2. 实现 `__hash__` 方法
> 3. 实现 `__eq__` 方法

** 针对 Vector 类实现以上三个条件 **

#### **1. 实例不可变**

使用 **私有属性** 和 **`property`** 装饰器，将向量的两个分量设为**只读属性**，来保证实例“不可变”（因为属性不是真正的私有，所以不是真正的不可变）


```python
class Vector2d:
    def __init__(self, x, y):
        self.__x = float(x)
        self.__y = float(y)
        
    @property
    def x(self):
        return self.__x
    
    @property
    def y(self):
        return self.__y
    
    def __iter__(self):
        return (i for i in (self.x, self.y))
```

- 两个前导下划线标记属性为私有
- `@property` 装饰器把读值方法标记为属性（方法名可以当作属性来使用，它们就是公共属性，就像 `__iter__` 方法里面用的那样）
- 之所以要实例不可变，是因为要实现 `__hash__` 方法和 `__eq__` 方法，保证相等的对象具有相同的散列值

#### **2. 实现 `__hash__` 方法**

`__hash__` 方法应该返回一个整数，根据[官方的文档](https://docs.python.org/3/reference/datamodel.html)，建议返回**各分量散列值的异或值**


```python
def __hash__(self):
    return hash(self.x) ^ hash(self.y)
```

#### **3. 实现 `__eq__` 方法**

之前已经实现      

<br>

***关于可散列 Vector 的完整代码和测试，和之后的内容一起给出（不然代码会重复冗长）***

----

## **6、“私有”属性和“受保护”的属性**

> 首先说明一点，Python 中没有像 Java 中那样的类似于 `private` 的限定修饰符，所以没有真正的实现私有和受保护，而是一种防护措施，更像是一种越定，就好比在 Java 中“私有”是法律规定，而在 Python 中“私有” 是道德规范。

> - 私有属性：防止某个属性在被直接使用或者是创建子类时被覆盖。Python 中使用两个前导下划线（尾部没有或者最多有一个下划线）命名的属性就是私有属性。
> - 名称改写：被命名为私有的属性，会被内部改写成 `_class__attr` 的形式，被存入实例的 `__dict__` 属性中。


```python
class A:
    def __init__(self, x):
        self.__x = x
        
    @property
    def x(self):
        return self.__x
```


```python
a.x = 2
```

	# Out
    ---------------------------------------------------------------------------

    AttributeError                            Traceback (most recent call last)

    <ipython-input-10-4930f24ffcf0> in <module>()
    ----> 1 a.x = 2
    

    AttributeError: can't set attribute



```python
a = A(1)
a.__dict__
```



	# Out
    {'_A__x': 1}




```python
a._A__x = 2
```

所以实际上私有属性的名称改写仅仅是一种防护措施，避免无意地覆盖了我们所想的变量，但是如果是要故意通过改写后的名称来覆盖属性也没有被禁止。    
一种比较好的处理属性的方案就是用 **私有属性** + **@property** 装饰器，不去使用改名后的名称，而是使用我们真正想命名的名称。

> 对于前导单下划线的属性，Python 不会做任何处理，但却是一个不成文的约定，约定它是一个“私有的”或者是“受保护的”属性，避免意外覆盖属性。但是需要注意的是，不能用 `from module import *` 导入

----

## **7、`__slots__` 类属性**

***慎用***

> - `__slots__` 类属性，在处理百万个属性不多的实例时，能够节省大量内存，它让解释器在元组中存储实例属性来代替字典（`__dict__`）；   
> - `__slots__` 类属性的值设为一个字符串构成的可迭代对象，一般是元组，其中每个元素表示各个实例属性；    
> - 在类中定义了 `__slots__` 属性后，实例不能再有 `__slots__` 中所列名称之外的其他属性；但是不要滥用它来禁止类的用户新增实例属性；    
> - 如果把 `'__dict__'` 加入到 `__slots__` 中，实例会在元组中保存各个实例的属性，也会支持动态创建属性，存在 `__dict__` 中；
> - `__weakref__` 实例属性为了让对象支持弱引用，自定义类中默认就有，但是如果定义了 `__slots__` 属性，而且想支持弱引用，必须把 `'__weakref__'` 加入 `__slots__` 中。

----

## **8、覆盖类属性**

> Python 独特的特性：**类属性可用于为实例属性提供默认值**

就像 Vector 类中的 `typecode` 属性，定义为了类属性，并且可以通过 `self.typecode` 来访问。

> - 如果实例本身并没有这个属性，就会使用类的同名属性，否则会用实例自己的属性
> - 为不存在的 **实例属性** 赋值，会新建 **实例属性**，并且同名 **类属性** 不受影响    
> - 要修改 **类属性** 的值，必须直接通过类获取它然后修改，不能通过实例修改

> 一个更符合 Python 风格的覆盖类属性的方法：**创建一个子类，只用于定制类的数据属性** （e.g. Django 基于子类的视图）

----

## **- 最终的Vector类代码以及测试 -**


```python
from array import array
import math

class Vector2d:
    typecode = 'd'
    
    def __init__(self, x, y):
        self.__x = float(x)
        self.__y = float(y)
        
    @property
    def x(self):
        return self.__x
    
    @property
    def y(self):
        return self.__y
    
    def __iter__(self):
        return (i for i in (self.x, self.y))
    
    def __repr__(self):
        class_name = type(self).__name__
        return '{}({!r}, {!r})'.format(class_name, *self)
    
    def __str__(self):
        return str(tuple(self))
    
    def __bytes__(self):
        return bytes([ord(self.typecode)]) + bytes(array(self.typecode, self))
    
    def __eq__(self, other):
        return tuple(self) == tuple(other)
    
    def __hash__(self):
        return hash(self.x) ^ hash(self.y)
    
    def __abs__(self):
        return math.hypot(*self)
    
    def __bool__(self):
        return bool(abs(self))
    
    def angle(self):
        return math.atan2(self.y, self.x)
    
    def __format__(self, fmt_spec):
        if fmt_spec.endswith('p'):
            fmt_spec = fmt_spec[:-1]
            coords = (abs(self), self.angle())
            outer_fmt = '<{}, {}>'
        else:
            coords = self
            outer_fmt = '({}, {})'
        components = (format(c, fmt_spec) for c in coords)
        return outer_fmt.format(*components)
    
    @classmethod
    def frombytes(cls, octets):
        typecode = chr(octets[0])
        memv = memoryview(octets[1:]).cast(typecode)
        return cls(*memv)
```

#### 格式化测试


```python
v1 = Vector2d(3, 4)
format(v1)
```



	# Out
    '(3.0, 4.0)'




```python
format(v1, '.2f')
```



	# Out
    '(3.00, 4.00)'




```python
format(v1, '.3e')
```



	# Out
    '(3.000e+00, 4.000e+00)'




```python
format(v1, 'p')
```



	# Out
    '<5.0, 0.9272952180016122>'




```python
format(v1, '.3ep')
```



	# Out
    '<5.000e+00, 9.273e-01>'




```python
format(v1, '0.5fp')
```



	# Out
    '<5.00000, 0.92730>'



#### hash 测试


```python
v2 = Vector2d(5, 12)
```


```python
hash(v1), hash(v2)
```



	# Out
    (7, 9)




```python
len(set([v1, v2]))
```



	# Out
    2




```python
d = {}
d[v1] = 'Python'
d[Vector2d(3.0, 4.0)]
```



	# Out
    'Python'

<br>

[md效果不好，原先是 jupyter notebook 版本](https://nbviewer.jupyter.org/github/sxnys/mypython/blob/master/FluentPython/9_Pythonic_Object/1_pythonic_object.ipynb)