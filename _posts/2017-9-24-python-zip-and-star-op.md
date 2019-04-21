---
layout: post
title: 'zip与*操作符到《流畅的python》'
tags: python
---
今天去官方文档上看了下zip的用法，然后引申到*操作符解压一个序列

zip的操作还是比较简单，看看官方文档怎么说的吧。

> Make an iterator that aggregates elements from each of the iterables.
>
> Returns an iterator of tuples, where the i-th tuple contains the i-th element from each of the argument sequences or iterables. The iterator stops when the shortest input iterable is exhausted. With a single iterable argument, it returns an iterator of 1-tuples. With no arguments, it returns an empty iterator.

基本意思就是说：返回一个会返回tuple的迭代器(就是说这个迭代器每个去访问它都会返回一个tuple)，第i个tuple中的内容是zip的每个可迭代对象的第i个数，它相当于以下函数：
```python
    def zip(*iterables):
        # zip('ABCD', 'xy') --> Ax By
        sentinel = object()
        iterators = [iter(it) for it in iterables]
        while iterators:
            result = []
            for it in iterators:
                elem = next(it, sentinel)
                if elem is sentinel:
                    return
                result.append(elem)
            yield tuple(result)
```
这里需要说明的是，返回的迭代器会生成的tuple个数是zip的可迭代参数最少的那个决定的。

然后再来说说`*`操作符，文档没找到，有点尴尬。只有自己写代码测试下了。应该和序列解包的概念差不多，就是不知道是不是所有的可迭代对象都可以使用`*`操作符来得到了。试验了下，tuple、list、dict都支持。

来个总纲式的代码：
```python
    x = [1, 2, 3]
    y = [4, 5, 6]

    z = zip(x, y)
    z1, z2, z3 = zip(x, y)
    print(z1, z2)
    x2 = zip(*z)
    print(x2)
    print(*(1, 2, 3))
    print(*{"one":1, "two":2, "three":3})

    def test():
        for i in range(3):
            yield i

    x, y, z = test()
    print(x, y, z)

    g = test()
    x = next(g)
    y = next(g)
    z = next(g)
    print(x, y, z)
```
当写到`z1, z2, z3 = zip(x, y)`时，有点不明白为啥能得到完整的值，文档也找不到相关的说明，就去群里问了下，老半天没人说出个所以然来(可能很多人知道，但是表达不出来)，直到有个群友说这个叫做序列解包，可以用于所有可迭代对象。一下恍然大悟，所以有群友说，推荐去看看《流畅的python》，看了下目录，刚好是适合我目前的需要。准备国庆好好看看这本书。

btw，python cookbook也是本不错的书，看完《流畅的python》再去看看这个，貌似先后顺序有点怪怪的，哈哈~

本来打算去刷文档的，但是刷文档实在太坑爹了，这下好了，可以看看基本的内置函数和内置模块后直接去探索python的一些比较深入的知识了

需要说明的是：`=`操作符的序列解包是需要满足一个条件：等号左边的数目要和右边的迭代器可返回的数目相同，但是可以通过这样一个技巧来写一个比较通用的：
```python
    z1, *z2, z3 = zip(x, y)
```
这样z2就是一个tuple，和`*args`一样的道理，就是需要注意`zip(x, y)`返回的可迭代对象生成太多对象或者直接是无限的就不适用于这种方式了