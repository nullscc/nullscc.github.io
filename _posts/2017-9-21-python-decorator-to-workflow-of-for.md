---
layout: post
title: '由python装饰器到for循环的工作方式'
tags: python
---
嗯~标题起的很高达上，但是此篇文章没有任何for循环工作方式的介绍。

事情是这样的，今天面试让当场在笔记本上写代码，是写一个装饰器的，然后我很高兴，因为我自认为我对装饰器的理解还不错~，而且有电脑可以调试应该不成什么问题。然后，不知道是不是被上家公司老板给逗到了，脑袋秀逗了~

题目是这样的：一个生成器，生成100个数，要在这个生成器上加上一个装饰器，然后这个生成器的输出就从一个number变成一个4元素的list。

题目还是很好理解的，面试官把基本的代码都帮我写好了:
```python
    def foo():
        pass

    @foo
    def func():
        for i in range(100)
            yield i
    if __name__ == "__main__":
        for i in func():
            print(i)
            assert isinstance(i, list)
```
然后我写成这样了(抽自己一巴掌先~)：
```python
    def foo(func):
        def wrapper():
            L = []
            for i in range(4):
                L.append(func())
        return wrapper

    @foo
    def func():
        for i in range(100)
            yield i
    if __name__ == "__main__":
        for i in func():
            print(i)
            assert isinstance(i, list)
```
因为在可以有电脑调试，我还特地看了下`func()`是什么类型，它是一个`generator`，然后想了半天居然还是不知道怎么做~

出现这种情况的原因还是自己对生成器不够熟悉，而不是对装饰器不够熟悉。

常规来说`func()`返回的是一个`generator`对象，当得到这个对象后直接对这个对象进行for循环操作或者`next()`函数来操作

这里的知识点有这么几个：

* 装饰器的理解
* 闭包的理解

改正后的程序是这样的：
```python
    def foo(func):
        g = func()
        def wrapper():
            while True:
                L = []
                for i in range(4):
                        L.append(next(g))
                yield L
        return wrapper

    @foo
    def func():
        for i in range(100):
            yield i
```
本来我以为这样`foo`会陷入到死循环，但是实际上不会，因为`L.append(next(g))`会抛出一个`StopIteration`异常，一旦foo函数发生异常，那么while循环就中断了。而for循环会处理这个异常，所以程序输出不会有任何问题

就酱，去看看for循环的工作方式