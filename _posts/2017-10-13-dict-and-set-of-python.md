---
layout: post
title: 'python中的字典与集合'
tags: python
---
python中的字典和集合是内置的数据结构，在正确的地方使用它好处多多

## 字典
定义一个字典的方法：
```python
    >>> a = dict(one=1, two=2, three=3)
    >>> b = {'one': 1, 'two': 2, 'three': 3}
    >>> c = dict(zip(['one', 'two', 'three'], [1, 2, 3]))
    >>> d = dict([('two', 2), ('one', 1), ('three', 3)])
    >>> e = dict({'three': 3, 'one': 1, 'two': 2})
    >>> a == b == c == d == e
    True
```

看看python3中的字典类`dict`的初始化方法
```python
    class dict(**kwarg)
    class dict(mapping, **kwarg)
    class dict(iterable, **kwarg)
```

它有三种初始化方法：

* 单纯的关键字参数，`a = dict(one=1, two=2, three=3)`
* 使用mapping对象初始化，`e = dict({'three': 3, 'one': 1, 'two': 2})`
* 使用可迭代对象初始化
```python
        >>> c = dict(zip(['one', 'two', 'three'], [1, 2, 3]))
        >>> d = dict([('two', 2), ('one', 1), ('three', 3)])
```
    使用可迭代对象初始化，需要可迭代对象每次返回两个元素，第一个元素作为key，第二个作为value

那么其他两个初始化方法的`**kwarg`是干嘛用的呢，它和第一种方法一样，是一个关键字参数，它照样会将关键字参数加入到字典中，如果在第一个参数和关键字参数的key定义有重复，那么会采用关键字参数中的定义

### python2中的字典与python3中字典的小不同
看看以下程序在python2与python3中的输出：
```python
    d = {"hello":"python", "world":"php"}
    print(type(d))
    print(type(d.keys()))
    print(type(d.values()))
    print(type(d.items()))
```
python2:
```python
    <type 'dict'>
    <type 'list'>
    <type 'list'>
    <type 'list'>
```
python3:
```python
    <class 'dict'>
    <class 'dict_keys'>
    <class 'dict_values'>
    <class 'dict_items'>
```

可见它们实际上是不同的，那么其实有什么用呢？python3中可以支持python2中不支持的一些方便的操作，如果`+`、`-`、`&`等运算操作可以直接作用在python3的`dict_keys`与`dict_items`类型上，而python2不行
```python
    a = {
        'x' : 1,
        'y' : 2,
        'z' : 3
    }

    b = {
        'w' : 10,
        'x' : 11,
        'y' : 2
    }

    print(a.keys() & b.keys())
    print(a.keys() - b.keys())
    print(a.items() & b.items())
```

## 集合
集合创建方法：
```python
    s = {"hello", "world"}
    class set([iterable])
```

第二种方法是的参数是一个可选的可迭代对象

需要注意的是集合的元素要是可哈希的，至于什么是可哈希的我也没搞懂，目前为止我理解成不可变的元素都是可哈希的

还有种集合是frozenset，字面意思是不可变，没用过～
```python
    class frozenset([iterable])
```
和字典一样，python3中的集合也可以直接进行运算（python2也可以）
```python
    s1 = {'x', 'y', 'z'}
    s2 = {'w', 'x', 'y'}

    print(s1 & s2)
    print(s1 - s2)
```

## 集合与字典的共同之处

* 都可以进行`&`、`-`（字典在python2中进行）
* 都是hash实现，set的in操作的时间复杂度和字典是一样的，都跟长度没有太大关系