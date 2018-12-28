---
title: Python垃圾回收和弱引用
tags: 
- Python
- Fluent Python
comments: on
---

----

## **垃圾回收**

1. **引用计数** ：CPython 中的主要垃圾回收算法，每个对象都会统计有多少引用指向自己；当引用计数归零时，对象立即销毁。
2. **分代垃圾回收** ：CPython 2.0 增加的新的算法，用于检测**引用循环**中涉及的对象组    

即将销毁实例时，解释器会调用 `__del__` 方法，给实例最后的机会，释放外部资源，但它不会销毁实例。

### `del`    
  
`del` 语句**删除的是名称，而不是对象**。但是在两种情况下可能会导致对象被回收：   
1. 删除的变量保存的是对象的**最后一个引用** *（引用计数）*
2. 删除变量后无法得到对象，如两个对象互相引用 *（分代垃圾回收）*

`sys.getrefcount` 方法可以监控对象的引用计数，因为该方法调用本身也会增加对象的引用计数，所以结果会比实际想的要多1


```
''' 使用 weakref.finalize 注册回调函数，监视对象生命周期 '''
import weakref, sys

s1 = {1, 2 ,3}
s2 = s1

print(sys.getrefcount(s1))

def bye():
    print('Gone with the wind ... ')

# 注册回调函数 bye，当 s1 指向的对象被销毁，执行 bye 函数
ender = weakref.finalize(s1, bye)

ender.alive
```
	# Out
    3
    True




```python
# del 只是删除 s1 这个名称，但是它指向的对象还有 s2 作为引用
del s1
print(sys.getrefcount(s2))
ender.alive
```
	# Out
    2
    True

```python
# s2 被重新绑定，导致 {1, 2, 3} 无法获取，即引用计数归零，从而对象被销毁，bye函数回调
s2 = 'byebye'
ender.alive
```
	# Out
    Gone with the wind ... 
    False
---

## **弱引用**

**弱引用**：不会增加对象的引用数量，不会妨碍**所指对象（referent）**被当作垃圾回收。    
弱引用在缓存应用中很有用，因为不想仅仅因为对象被缓存引用着而始终被保持。

`weakref.ref` 类的实例获取所指对象，提供的是底层接口，尽量不要手动创建并处理`weakref.ref`实例。    
以下仅是演示弱引用 ... 


```python
# 创建弱引用对象
a_set = {0, 1}
wref = weakref.ref(a_set)
wref
```
	# Out
    <weakref at 0x03B7AA80; to 'set' at 0x03ACD3F0>

```python
#返回被引用的对象，因为这里是控制台会话，返回的值（即[Out]）会绑定到 _ 变量
wref()
```
	# Out
    {0, 1}

```python
# 重新绑定 a_set，减少了 {0, 1} 的引用计数，但是 _ 变量仍然指向它
a_set = {2, 3}
wref()
```
	# Out
    {0, 1}

```python
# 接下来的 [Out] 值会重新绑定 _ 变量，导致 {0, 1} 引用计数归零（ipython内核中还有__和___变量）
wref() is None
```
	# Out
    False

```python
# 此时 _ 变量绑定的是 False，{0, 1} 引用计数归零（在Python原生控制台里是这样，但是这里的控制台的ipython内核，结果不一样）
wref() is None
```
	# Out
    False

```python
# 在ipython内核中，使用 dir() 可以看到一些全局变量，globals可以看到他们的值，{0, 1}还有被 _5, _6 变量引用（即对应的[Out]值，具体视情况而定，这里就是 5 和 6）,
# 除此之外，在 Out 以及 _oh 全局变量中，它们是保存当前所有[Out]值的字典（是同一个字典对象），5,6 key对应的value还是{0, 1}的引用，
# 最后，还有一个大坑，在ipython内核中，有_、__、___三个变量，分别表示当前Out值、上一个Out值、上上个Out值，所以到这个地方为止，___还是{0, 1}的引用
# 本以为这样就OK了，但是{0, 1}的引用还存在，已经想方设法绞尽脑汁在找，就是找不到是谁还在引用{0, 1}，但至少弱引用不是；力竭，但愿以后能够发现
print(_5, _6, _oh[5], _oh[6])
print(_, __, ___)
_5, _6 = None, None
_oh[5], _oh[6] = None, None
___ = None
wref() is None
```
	# Out
    {0, 1} {0, 1} {0, 1} {0, 1}
    False False {0, 1}
    False



**一般不要直接创建处理 `weakref.ref` 实例，最好使用 `weakref` 集合和 `finalize`，即 `WeakKeyDictionary`、`WeakValueDictionary`、`WeakSet`、`finalize`**

### `WeakValueDictionary`

`WeakValueDictionary` 类实现的是一种可变映射，值是对象的弱引用，被引用的对象被回收后，对应的键会主动删除，常用于缓存。    
`WeakKeyDictionary`与之对应，它存储的键是对象的弱引用。


```python
class Cheese:
    
    def __init__(self, kind):
        self.kind = kind
        
    def __repr__(self):
        return 'Cheese(%r)' % self.kind
```


```python
stock = weakref.WeakValueDictionary()
catalog = [Cheese('A'), Cheese('B'), Cheese('C'), Cheese('D')]

for cheese in catalog:
    stock[cheese.kind] = cheese
```


```python
sorted(stock.keys())
```

	# Out
    ['A', 'B', 'C', 'D']


```python
# 删除了 catalog 列表，对应的值弱引用字典中对应的键被删除，但是 'D' 还存在一个引用，for循环中的 cheese，它是全局变量
del catalog
sorted(stock.keys())
```
	# Out
    ['D']

```python
# 删除了 cheese 之后，所有键被删除
del cheese
sorted(stock.keys())
```
	# Out
    []

<br>

[md效果不好，原先是 jupyter notebook 版本](https://nbviewer.jupyter.org/github/sxnhys/mypython/blob/master/FluentPython/8_ObjectReferences_Mutability_Recycling/4_garbage_collection_And_weak_references.ipynb)
