---
title: Python数据模型
tags: 
- Python
- Fluent Python
comments: on
---

----
> 理解好Python的数据模型才是真正理解了Python这门语言
----

> Python数据模型其实就是 **对象模型**，**数据模型是对Python框架的描述，规范了Python自身构建模块的接口**，数据模型体现了Python的设计思想，理解好就能写出 **Pythonic** 代码。<br>     
> 类似于经常用到的 **`len(something)`** 背后调用的是 **`something.__len__`** 方法，**`obj[key]`** 背后调用的是 **`obj.__getitem__`** 方法等等，这些方法就是构建Python数据模型的关键，它们称为 **特殊方法**（魔术方法或双下方法）。

----

## **Pythonic 纸牌**

*Fluent Python 中对于 Python 特殊方法的代码示例*

```
import collections
from random import choice

# namedtuple用来构建只有少数属性但是没有方法的对象
# 访问其中的属性，可以像元组那样使用索引，也可以像一般对象那样使用点操作符(.)
Card = collections.namedtuple('Card', ['rank', 'suit'])

class FrenchDeck:
    ranks = [str(i) for i in range(2, 11)] + list('JQKA')
    suits = 'spades diamonds clubs hearts'.split()

    def __init__(self):
        self._cards = [Card(rank, suit) for suit in self.suits for rank in self.ranks]

    def __len__(self):
        return len(self._cards)

    # 可以利用索引查找元素，而且使得它的对象可迭代
    def __getitem__(self, pos):
        return self._cards[pos]

    def __repr__(self):
        return 

if __name__ == '__main__':
    deck = FrenchDeck()
    print(len(deck))

    # 打印第一张、最后一张、最上面三张、花色是A的扑克牌
    print(deck[0], deck[-1], deck[:3], deck[12::13], sep='\n')

    # 随机选择一个元素
    print(choice(deck))

    # 打印每一张牌，打印数字，打印花色
    for card in deck:
        print(card, card[0], card.suit)
    
    # 扑克牌排序
    suit_values = dict(spades = 3, hearts = 2, diamonds = 1, clubs = 0)
    def spades_high(card):
        rank_value = FrenchDeck.ranks.index(card.rank)
        return len(suit_values) * rank_value + suit_values[card.suit]
    for card in sorted(deck, key=spades_high):
        print(card)
```
可以看到仅仅这些特殊方法就让对象的操作十分简便，特别是定义了 **`__getitem__`** 方法就可以实现对象的 **迭代** 和 **切片** 操作，这个示例不仅体现了Python面向对象的风格，也是体现出Python的核心语言特(如迭代和切片)

----

## **自定义二维向量**
*关于运算符的特殊方法*
```
﻿from math import hypot 	 # 两个数平方和开方

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
这里通过 **`__add__`** 和 **`__mul__`** 方法分别实现了自定义二维向量的对于 + 和 * 算数运算符的使用，同时 **`__abs__`** 实现了向量的模，**`__bool__`** 实现判断向量是否为零向量。

----

## **特殊方法一览**

1. **与运算符无关的特殊方法**
```
﻿字符串/字节序列表示形式 ： __repr__、__str__、__format__、__bytes__
数值转换 ： __abs__、__bool__、__complex__、__int__、__float__、__hash__、__index__
集合模拟 ： __len__、__getitem__、__setitem__、__delitem__、__contains__
迭代枚举 ： __iter__、__reversed__、__next__
可调用模拟 ： __call__
上下文管理 ： __enter__、__exit__
实例创建和销毁 ： __new__、__init__、__del__
属性管理 ： __getattr__、__getattribute__、__setattr__、__delattr__、__dir__
属性描述符 ： __get__、__set__、__delete__
类相关的服务 ： __prepare__、__instancecheck__、__subclasscheck__
```
2. **与运算符有关的特殊方法**
```
一元运算符 ： __neg__  - 、__pos__  + 、__abs__  abs()
比较运算符 ： __lt__  < 、__le__  <= 、__eq__  == 、__ne__  != 、__gt__  > 、__ge__  >=
算术运算符 ： __add__  + 、__sub__  - 、__mul__  * 、__truediv__  / 、__floordiv__  // 、__mod__  % 、__divmod__  divmod() 、__pow__  ** 、__round__  round()
反向算术运算符 ： __radd__、__rsub__、__rmul__、__rtruediv__、__rfloordiv__、__rmod__、__rdivmod__、__rpow__
增量赋值算术运算符 ： __iadd__、__isub__、__imul__、__itruediv__、__ifloordiv__、__imod__、__ipow__
位运算符 ： __invert__  ~ 、__lshift__  << 、__rshift__  >> 、__and__  & 、__or__  \： 、__xor__  ^
反向位运算符 ： __rlshift__、__rrshift__、__rand__、__ror__、__rxor__
增量赋值位运算符 ： __ilshift__、__irshift__、__iand__、__ior__、__ixor__
```
----

## **小结**
数据模型是Python风格的体现，特殊方法又是数据模型构建的关键，通过实现特殊方法，自定义的数据类型可以像内置类型一样随心所欲。特殊方法这么多，当然是遇则熟然则疏，没有必要去记住所有，当需要用到的时候自然就认识它了。