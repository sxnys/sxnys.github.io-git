---
title: Python的复制（拷贝）问题
tags: 
- Python
- Fluent Python
comments: on
---

----
> *深浅复制的讨论是基于可变类型的*
----

## **浅复制**

 **复制最外层容器，副本中的元素是源容器中元素的引用**      
 **列表**浅复制包括 ——   
 1. 构造方法
 2. `[:]` 切片赋值
 3. 自身的 `copy` 方法
 4. `copy` 模块的 `copy` 方法


```python
l1 = [1, 2, [3, 4], (5, 6, 7), 8]

# 构造方法
l2 = list(l1)

# 切片赋值
l3 = l1[:]

# 列表浅复制方法
l4 = l1.copy()

# copy模块的浅复制方法
import copy
l5 = copy.copy(l1)
```


```python
# 外层容器已经不是同一个对象了
print(l1 is l2, 
      l1 is l3, 
      l1 is l4, 
      l1 is l5)

# 内层容器还是同一个对象
print(l1[2] is l2[2], 
     l1[2] is l3[2], 
     l1[2] is l4[2], 
     l1[2] is l5[2], )

print(l1[3] is l2[3], 
     l1[3] is l3[3], 
     l1[3] is l4[3], 
     l1[3] is l5[3], )
```
	# Out
    False False False False
    True True True True
    True True True True
    

毫无疑问，结果展现出了 **浅复制** 的特点   

接下来针对内部容器进行一些修改，不出意外的话应该是改一个必然会影响其他的，因为内部容器还是相同对象。
但是 ...  *（以 `l1` 和 `l2` 为例）*


```python
# l1 修改内部的非容器序列不会影响 l2
l1.remove(8)
l1.append(9)
print('l1: {}'.format(l1), 
      'l2: {}'.format(l2), sep='\n')

print()

# l1 修改内部的列表肯定会影响 l2，l2也会改变
l1[2] += [33, 44]
print('l1: {}'.format(l1), 
      'l2: {}'.format(l2), sep='\n')
```
	# Out
    l1: [1, 2, [3, 4], (5, 6, 7), 9]
    l2: [1, 2, [3, 4], (5, 6, 7), 8]
    
    l1: [1, 2, [3, 4, 33, 44], (5, 6, 7), 9]
    l2: [1, 2, [3, 4, 33, 44], (5, 6, 7), 8]
    

结果依然符合预期，接下来对内部元组进行*“修改”*（ps: 元组不可变）


```python
l1[3] += (55, 66)
print('l1: {}'.format(l1), 
      'l2: {}'.format(l2), sep='\n')
```
	# Out
    l1: [1, 2, [3, 4, 33, 44], (5, 6, 7, 55, 66), 9]
    l2: [1, 2, [3, 4, 33, 44], (5, 6, 7), 8]
    

结果似乎不是想的那样，`l1` 改了内部元组没有影响到 `l2` 中对应的元组。   

首先，这里的修改是 `+=`，这类运算符有其特殊性。    
- 对于列表，`+=` 是 **就地修改**；对于元组，`+=` 会重新创建一个元组，然后绑定到变量名上（这里就是 `l1[3]` ）    
`+=` 是容器对象的 `__iadd__` 实现的，本质含义就是 **就地修改**    
- 列表实现了 `__iadd__` ，但是元组是 **不可变** 的，所以尽管 `+=` 对于元组照样能用，只不过不是就地修改，而是创建一个新的元组对象再赋值

---

## **深复制**

**副本不共享内部对象的引用**    

使用 `copy` 模块的 `deepcopy` 方法实现深复制    

#### 深浅复制对比  *（上下公交车）*


```python
class Bus:
    
    def __init__(self, passengers=None):
        if passengers is None:
            self.passengers = []
        else:
            self.passengers = list(passengers)
            
    def pick(self, name):
        self.passengers.append(name)
        
    def drop(self, name):
        self.passengers.remove(name)
        
import copy
bus1 = Bus(['Alice', 'Bob', 'Canny', 'David'])
bus2 = copy.copy(bus1)
bus3 = copy.deepcopy(bus1)
```


```python
# 三个不同的对象
id(bus1), id(bus2), id(bus3)
```



	# Out
    (1887896688120, 1887896688176, 1887897229296)




```python
bus1.drop('Alice')
```


```python
# bus2 是 bus1 的浅复制，共享 passengers属性
bus2.passengers
```



	# Out
    ['Bob', 'Canny', 'David']




```python
# bus3 是 bus1 的深复制，passengers属性指向另一个列表
bus3.passengers
```



	# Out
    ['Alice', 'Bob', 'Canny', 'David']




```python
id(bus1.passengers), id(bus2.passengers), id(bus3.passengers)
```



	# Out
    (1887897173640, 1887897173640, 1887897170888)



> `deepcopy`会记住已经复制的对象，优雅地处理**循环引用**


```python
a = [1, 2]
b = [a, 3]
a.append(b)
a
```




    [1, 2, [[...], 3]]




```python
c = copy.deepcopy(a)
c
```




    [1, 2, [[...], 3]]

--- 

## **不可变类型的复制**

**看似奇怪但是细想有道理的现象是，Python对不可变类型复制的实现细节和可变类型不同**    

> 包括 tuple、str、bytes、frozenset 等的不可变类型，它们的构造方法、`[:]` 赋值、copy、deepcopy，最后得到的都是同一个对象的引用，不是创建的副本。


```python
t1 = (1, 2, 3)
t2 = tuple(t1)
t3 = t1[:]
t4 = copy.copy(t1)
t5 = copy.deepcopy(t1)

t2 is t1, t3 is t1, t4 is t1, t5 is t1
```



	# Out
    (True, True, True, True)



所以在之前对于包含元组的列表进行深拷贝，里面的元组同样指向同一对象


```python
ll1 = [1, (2, 3)]
ll2 = copy.deepcopy(ll1)
ll1[1] is ll2[1]
```



	# Out
    True



文档中也指出这一行为 —— **`If the argument is a tuple, the return value is the same object.`**


```python
help(tuple)
```
	# Out
    Help on class tuple in module builtins:
    
    class tuple(object)
     |  tuple() -> empty tuple
     |  tuple(iterable) -> tuple initialized from iterable's items
     |  
     |  If the argument is a tuple, the return value is the same object.
     |  
     |  Methods defined here:
     |  
     |  __add__(self, value, /)
     |      Return self+value.
     |  
     |  __contains__(self, key, /)
     |      Return key in self.
     |  
     |  __eq__(self, value, /)
     |      Return self==value.
     |  
     |  __ge__(self, value, /)
     |      Return self>=value.
     |  
     |  __getattribute__(self, name, /)
     |      Return getattr(self, name).
     |  
     |  __getitem__(self, key, /)
     |      Return self[key].
     |  
     |  __getnewargs__(...)
     |  
     |  __gt__(self, value, /)
     |      Return self>value.
     |  
     |  __hash__(self, /)
     |      Return hash(self).
     |  
     |  __iter__(self, /)
     |      Implement iter(self).
     |  
     |  __le__(self, value, /)
     |      Return self<=value.
     |  
     |  __len__(self, /)
     |      Return len(self).
     |  
     |  __lt__(self, value, /)
     |      Return self<value.
     |  
     |  __mul__(self, value, /)
     |      Return self*value.n
     |  
     |  __ne__(self, value, /)
     |      Return self!=value.
     |  
     |  __new__(*args, **kwargs) from builtins.type
     |      Create and return a new object.  See help(type) for accurate signature.
     |  
     |  __repr__(self, /)
     |      Return repr(self).
     |  
     |  __rmul__(self, value, /)
     |      Return self*value.
     |  
     |  count(...)
     |      T.count(value) -> integer -- return number of occurrences of value
     |  
     |  index(...)
     |      T.index(value, [start, [stop]]) -> integer -- return first index of value.
     |      Raises ValueError if the value is not present.
    
    

考虑一下为什么Python对于不可变类型的复制是这样的。不可变类型由于其 **不可变性**，所以不会出现由改变某一对象而引发的各种意外，因此针对不可变类型的复制完全可以是指向同一个对象，这样能够节省内存，提升解释器的速度。类似的现象还有共享字符串字面量的 **驻留**。<br>
如果非要创建一个内容完全一样，但是不想指向同一对象的元组，也不想重新敲一遍赋值的内容，就是想由 `t1` 创建一个对象不同的 `t2`，那就这样吧 ...


```python
t1 = (1, 2)
t2 = t1 + ()

print(t1, t2)

t1 is t2
```
	# Out
    (1, 2) (1, 2)
    False

<br>

[md效果不好，原先是 jupyter notebook 版本](https://nbviewer.jupyter.org/github/sxnhys/mypython/blob/master/FluentPython/8_ObjectReferences_Mutability_Recycling/2_copy.ipynb)