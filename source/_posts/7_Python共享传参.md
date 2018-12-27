---
title: Python共享传参
tags: Python
comments: on
---

----

## **函数的参数作为引用**

> Python 唯一支持的参数传递是 **共享传参** ，也就是常说的引用传参。   
> *函数内部的形参是实参的别名*


```python
def f(a, b):
    a += b
    return a
```

```python
# 数字，不变
x, y = 1, 2
f(x, y)
print(x, y)

# 列表可变类型，变了
x, y = [1, 2], [3, 4]
f(x, y)
print(x, y)

# 元组不可变类型，不变
x, y = (1, 2), (3, 4)
f(x, y)
print(x, y)
```
	# Out
    1 2
    [1, 2, 3, 4] [3, 4]
    (1, 2) (3, 4)
    

### 函数参数的默认值

*避免使用可变类型作为参数的默认值*


```python
class HauntedBus:
    
    def __init__(self, passengers=[]):
        self.passengers = passengers
        
    def pick(self, name):
        self.passengers.append(name)
        
    def drop(self, name):
        self.passengers.remove(name)
```


```python
# 给定参数没有问题
bus1 = HauntedBus(['Alice', 'Bob'])
print(bus1.passengers)

bus1.pick('Canny')
bus1.drop('Alice')
bus1.passengers
```
	# Out
    ['Alice', 'Bob']
    ['Bob', 'Canny']




```python
# 使用参数的默认值，即 []
bus2 = HauntedBus()
bus2.pick('David')
bus2.passengers
```



	# Out
    ['David']




```python
# 同样使用默认值，问题就来了
bus3 = HauntedBus()
bus3.passengers
```



	# Out
    ['David']




```python
bus3.pick('Bruce')
bus2.passengers
```



	# Out
    ['David', 'Bruce']




```python
bus3.passengers is bus2.passengers
```



	# Out
    True




```python
HauntedBus.__init__.__defaults__
```



	# Out
    (['David', 'Bruce'],)




```python
HauntedBus.__init__.__defaults__[0] is bus2.passengers
```



	# Out
    True



`bus3` 按道理使用的是默认值 `[]`，但是得到的确实 `bus2` 的 `passengers` 值，而且它们都是同一个引用。    
**默认值在定义函数时计算**，因此默认值变成了函数对象的属性。    
如果默认值是可变对象，而且修改了它的值，那么后续的函数调用都会受到影响。

#### `__init__` 方法应该这样处理默认值，防御可变参数


```python
class XXX:
    
    def __init__(self, passengers=None):
        if passengers is None:
            self.passengers = []
        else:
            self.passengers = list(passengers)   # 注意不是直接赋值，而是副本赋值，关于是深浅拷贝视情况而定
            
    ...
```

**除非确实是想修改通过参数传入的对象，否则在类中直接把参数赋值给实例变量之前一定要想清楚，因为这样会为参数对象创建别名。**

<br>

[md效果不好，原先是 jupyter notebook 版本](https://nbviewer.jupyter.org/github/sxnhys/mypython/blob/master/FluentPython/8_ObjectReferences_Mutability_Recycling/3_function_parameters_as_references.ipynb)