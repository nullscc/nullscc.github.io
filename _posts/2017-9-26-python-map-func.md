---
layout: post
title: 'python的map函数'
tags: python
---
python的map函数说起来很简单，对map参数中的可迭代对象应用所有的元素到函数进行执行。

但是其实远不止这么简单。

先看看官方文档(python3.6)怎么说的：

> Return an iterator that applies function to every item of iterable, yielding the results. If additional iterable arguments are passed, function must take that many arguments and is applied to the items from all iterables in parallel. With multiple iterables, the iterator stops when the shortest iterable is exhausted. For cases where the function inputs are already arranged into argument tuples, see itertools.starmap().

官方文档说了这么几个点：

1. 返回一个应用所有可迭代对象元素到函数的迭代器
2. 如果有额外的可迭代参数传递过来了，那么函数必须接收可迭代对象的多个参数，如果多个可迭代对象的元素个数不一样，那么只应用最短的(有点绕，看下面的代码就知道了)
3. 如果函数输入已经在一个tuple参数里面了，那么使用`itertools.starmap()`

嗯，老规矩，看代码，猜输出吧。如果猜错了说明对它还不熟悉：
```python
    def add100(x):
        print("excute add100")
        return x+100

    hh = (11, 22, 33)
    print("before map")
    m = map(add100, hh)
    print("after map")
    print(list(m))

    def x100(x):
        return x*100
    ii = [11, 22, 33]
    m = map(x100, ii)
    print(list(m))

    def addab(a, b):
        return a+b

    ii = [11, 22]
    print(list(map(addab, hh, ii)))


    list1 = [11,22,33]
    m = map(None,list1)
    print(dir(m))
    print(next(m))
```

上面代码假设在python3下运行，输出这里还是不贴了，跑一下就知道了。

这里除了官方文档说的外还需要注意以下几点：

* python3中是临时算的，它会返回一个map对象，这个对象是可迭代的，我们去操作这个对象来得到我们需要的数据，而不是一次性将所有的值算出来，python2中是直接一次性算出来的
* 可以有多个可迭代对象参数，但是map的函数参数必须接收多个可迭代对象的多个值
* 在python3中函数不能为None，python2中可以

分别在python2与python3中运行以下代码就可以看出map函数在python2与python3中的区别了
```python
    def add100(x):
        print("excute add100")
        return x+100

    hh = (11, 22, 33)
    print("before map")
    m = map(add100, hh)
    print("after map")

    print(m)
    print("before list")
    print(list(m))
    print("after list")

    list1 = [11,22,33]
    m = map(None,list1)
    print(m)
```