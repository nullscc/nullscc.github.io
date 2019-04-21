---
layout: post
title: 'python中%与str.format的区别'
tags: python
---
我们通常见到的python中字符串格式化主要有这么两种：%与format函数

那么这两种格式化字符串有什么区别呢？

看一个例子：
```python
    t = （1, 2, 3）
    print("----:{}".format(t))
    print("++++:%s" % t)

    '''
    output:

        ----:(1, 2, 3)
        Traceback (most recent call last):
        File "strformat.py", line 19, in <module>
            print("++++:%s" % t)
        TypeError: not all arguments converted during string formatting
    '''
```
说明 % 不可以输出tuple这类的复杂数据结构。

为什么呢？理解下 % 和 format 函数就知道为什么啦。


## % 的介绍
实际上我们可以看看python文档关于 % 的描述：

> String objects have one unique built-in operation: the % operator (modulo). This is also known as the string formatting or interpolation operator. Given format % values (where format is a string),% conversion specifications in format are replaced with zero or more elements of values. The effect is similar to using the sprintf() in the C language.
>
> If format requires a single argument, values may be a single non-tuple object. [5] Otherwise, values must be a tuple with exactly the number of items specified by the format string, or a single mapping object (for example, a dictionary).

简单来说：

* % 是操作符，它是字符串对象独有的
* 此操作符可以有三类参数：单个对象(非tuple、字典)、tuple、字典

一个例子：
```python
    >>> print('%(language)s has %(number)03d quote types.' %
    ...       {'language': "Python", "number": 2})
    Python has 002 quote types.
```
很显然，% 放在字符串后面是操作符，而format是一个函数

先来说说 % 的用法：

* 操作符后面的参数类型可以是什么上面说了
* 此操作符的动作是代替字符串中的转换说明符：就是`%03d`之类的东西
* 转换说明符的书写规则为(需要按顺序)：

    1. 起始为 %
    2. 圆括号包括的字符序列(可选)
    3. 转换标志，比如控制左对齐右对齐之类的，比如`%03d`中的`0`就是转换标志(可选)
    4. 最小域宽度，表示最小要占几个字符的位置，比如`%03d`中的`3`就表示最少要占3个字符的位置(可选)
    5. . 后面跟一个精度值(或者`*`，表示从后面的元素参数中读取一个值来代替 `*`下面会有例子)表示字符串或浮点数保留多少位(可选)
    6. 长度修改(但是会被python忽略)(可选)
    7. 转换类型，`%03d`表示转换类型为`d`，即整型(可选)

    总结起来就是`%[(name)][flags][width].[precision]typecode`

转换标志有下面几个：

* `# ` 这个直接用例子来说就是一个八进制数是`0o10`这样，如果打印出来就是10，这样我怎么直接从打印输出中识别它是八进制数字呢？这时`#`就派上用场了，它会在10前面加上`0o`
* 0 表示如果宽度不够用 0 来填充
* `-` 表示向左对齐(它和0同时存在时，0不起作用)
* ' ' 空格，表示如果一个数是整数，那么会在前面补一个空格，主要是为了与负号对齐
* `+` 表示会在数字的前面加上 + 或者 - (如果它和空格同时存在，那么空格不起作用)

下面是例子，看懂这些基本概念就弄懂啦：
```python
    print("% d" % 10)
    print("%05d" % -10)
    print("%+10d" % 10)
    print("%-10d" % -10)
    print("%10.3f" % -10.123456)
    print("%.*f" % (4, 1.2))
    print("%.5s" % "helloworld")
    print("%f" % 1.2) # 默认保留6位小数
    print("%*f" % (10, 1.2))
    print("%*f %d" % (10, 1.2, 100))

    print("%#o" % 10)
    print("%o" % 10)
    print("%e" % 10) # 默认保留6位小数

    print("%o" % 0o10)
    print("%#o" % 0o10)
    '''
    output:

        10
        -0010
            +10
        -10
        -10.123
        1.2000
        hello
        1.200000
        1.200000
        1.200000 100
        0o12
        12
        1.000000e+01
        10
        0o10
    '''
```
需要注意的是，由于python中字符串有明确的字符串长度，%s转换类型不会假设`\0`是字符串的结尾

## str.format 的介绍
str.format的详细介绍还是放到下一篇文章，因为实在挺多的。

这里暂时需要明白的是：str.format相较于 % 操作符更多变，而且支持的种类更多，而 % 只支持很有限的数据类型。在实际工作中最好还是用 str.format