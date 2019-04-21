---
layout: post
title: 'python中的除法'
tags: python
---
python中python2与python3的除法有点不一样，究竟为什么不一样还没去细究，这两天有时间去撸撸python文档，好好认识一下python的语言特性。

直接看代码吧。

python2：
```python
    Python 2.7.5 (default, Nov  6 2016, 00:28:07) 
    [GCC 4.8.5 20150623 (Red Hat 4.8.5-11)] on linux2
    Type "help", "copyright", "credits" or "license" for more information.
    >>> 21.5/2
    10.75
    >>> 21.5//2
    10.0
    >>> 21/2
    10
    >>> 21//2
    10
```
python3：
```python
    Python 3.6.1 (default, Aug 11 2017, 18:46:26) 
    [GCC 4.8.5 20150623 (Red Hat 4.8.5-11)] on linux
    Type "help", "copyright", "credits" or "license" for more information.
    >>> 21.5/2
    10.75
    >>> 21.5//2
    10.0
    >>> 21/2
    10.5
    >>> 21//2
    10
```

python中的除法有三种：

* 传统除法：如果是整数除法则执行地板除，如果是浮点数除法则执行精确除法。
* 精确除法/true除法：除法总是会返回真实的商，不管操作数是整形还是浮点型。
* floor除法：就是`//`操作符，返回不大于结果的最大整数

然后从上面代码中可以得出以下结论了:

1. python3中`/`操作数是true除法，而python2中是传统除法
2. `//`是地板除，如果操作数带小数，那么保留小数，如果不带小数，则只有整数