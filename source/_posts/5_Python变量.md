---
title: Python变量
tags: Python
comments: on
---

----

## **变量**

> **Python中，变量只是一个标注（标签），而不是一个盒子（box）。**    

Python中的变量类似于C++、Java等中的引用变量。**Important!** 这里谈到的变量是**容器序列**（list、tuple、dict、set ...），扁平序列（str、bytes ...）有所不同。

翻译一下就是，这里有一个装着*变量* 的盒子，你可以给这个盒子起名为*a*、*py*、*sxn*、...，想起多少个起多少个（只要你能够记住它们都代表这个盒子，不要有人问的时候自己都迷糊了），但是真正确定这个盒子的是他的 **标识（id）**，比如XX市xx小区xx楼xx这样的位置，这里就是这个盒子在内存地址。

Python中的变量类似于C++、Java等中的引用变量，所以所有参数传递都是 **引用传参**（**共享传参**）。

> **摒弃这样一个错误观念：变量是存储数据的盒子（众多隐藏 bug的罪魁祸首）；接受这样一个正确观念：变量是存储数据的盒子 的标注（标签），不唯一**

<br>


**最简单的 *Example* **


```python
a = [1, 2, 3]
b = a
b.append(4)
print(a, b)
a.append(5)
print(a, b)
```
	# Out
    [1, 2, 3, 4] [1, 2, 3, 4]
    [1, 2, 3, 4, 5] [1, 2, 3, 4, 5]
    

对于赋值语句，正确的理解是**把变量分配给对象**（即左边分配给右边），因为对象在赋值前就创建了，赋值语句始终先读右边。


```python
class myAssignment:
    def __init__(self):
        print('myAssignment id is {}'.format(id(self)))
        
x = myAssignment()
y = myAssignment() * 10    # 报错，但是右边的myAssignment对象已经创建，而y没有创建
```
	# Out
    myAssignment id is 1930710009840
    myAssignment id is 1930710009336
    


    ---------------------------------------------------------------------------

    TypeError                                 Traceback (most recent call last)

    <ipython-input-2-389f3865e312> in <module>()
          4 
          5 x = myAssignment()
    ----> 6 y = myAssignment() * 10    # 报错，但是右边的myAssignment对象已经创建，而y没有创建
    

    TypeError: unsupported operand type(s) for *: 'myAssignment' and 'int'



```python
print(dir())   # 可以看到y没有被创建
```
	# Out
    ['In', 'Out', '_', '_3', '__', '___', '__builtin__', '__builtins__', '__doc__', '__loader__', '__name__', '__package__', '__spec__', '_dh', '_i', '_i1', '_i2', '_i3', '_i4', '_ih', '_ii', '_iii', '_oh', 'a', 'b', 'exit', 'get_ipython', 'myAssignment', 'quit', 'x']
    
---
## **相等性**

1. 之前提到给变量盒子起很多个名字，或者说贴很多个标注，它们仅仅都是 **别名**
2. 如果有另一个盒子，里面的数据和之前的盒子里的一模一样，当然这就是“冒充”了

所以从两个方面比较两个变量：
1. 等价，即内存地址一样 （`is` 运算符和 `id` 函数，`is` 比较的就是对象的标识，`id()` 返回标识的整数表示）
2. 值相等（ ``==`` 运算符）


```python
sxn = {'name': 'Xiaonan Sang', 'age': 24}
bruce_sang = sxn
print(bruce_sang is sxn)
print(id(sxn), id(bruce_sang))
# 等价则值一定相等，就不code了
```
	# Out
    True
    1930710672512 1930710672512
    


```python
fake_sxn = {'name': 'Xiaonan Sang', 'age': 24}
print(fake_sxn == sxn)    # 值相等，tips: ==由__eq__方法实现
print(fake_sxn is sxn)    # 但是不等价
```
	# Out
    True
    False
    

*In summary*，`==` 运算符比较两个对象的值（保存的数据）；`is` 比较对象的标识。   

通常情况下可能会更关心的是值相等，但是在变量和单例值之间比较时应使用`is`，最常用的就是检查变量绑定的值是否为 `None`。   
`x is None`    
`x is not None`    

由于 `is` 不能重载，所以它比 `==` 速度快很多。

### 扁平序列的特殊性

> 扁平序列，即 str、bytes、array.array 等单一类型的序列，它们和容器序列不同，它们保存的不是引用，而是**在连续的内存中保存数据本身**。


```python
a = 'abc'
b = 'abc'
print(a is b)

a = '#'
b = '#'
print(a is b)

a = '#@'
b = '#@'
print(a is b)
```
	# Out
    True
    True
    False
    

看上面的结果会感到很奇怪，一般字符的组合会在同一个连续内存块保存，但是两组特殊字符的组合会另开空间存。   

> 这其实是Python**共享字符串字面量**，称为 **驻留**，是一种优化措施，同样适用于小的整数，防止重复创建一些常用的字符串或者数字；关于驻留条件的实现细节没有文档说明，不要依赖 **驻留**，只需知道这个奇怪现象的存在。

---

## **元组的相对不可变性**

和其他容器类型一样，元组保存的也是对象的引用。   

但是，如果其中某个引用的元素是可变的，即便元组本身不可变，该元素依然可变。   

换句话说，元组的不可变性是指**保存的引用**不可变，与引用的对象无关。


```python
t1 = (1, 2, [3, 4])
t2 = (1, 2, [3, 4])
print(t1 == t2)
print(id(t1[-1]))

t1[-1].append(5)
print(t1, id(t1[-1]), t1 == t2)   # t1[-1]就地修改了，但是标识没有变，也就是说 t1 没有变
```
	# Out
    True
    1930710647624
    (1, 2, [3, 4, 5]) 1930710647624 False
    

这也就是某些包含了不可散列对象的元组，它们也不可散列的原因。

接下来就是 **copy** 和 **deepcopy** 的问题了。

<br>

[md效果不好，原先是 jupyter notebook 版本](https://nbviewer.jupyter.org/github/sxnhys/mypython/blob/master/FluentPython/8_ObjectReferences_Mutability_Recycling/1_variables_identity_equality_aliases.ipynb)