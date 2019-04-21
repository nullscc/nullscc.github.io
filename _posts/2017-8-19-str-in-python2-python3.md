---
layout: post
title: 'python2与python3中的字符串'
tags: python
---
python中的字符串处理还是有点麻烦的，不过了解本质以后就比较清楚了，所以有必要了解清楚python2与python3中的字符串处理的区别。

2.x中字符串有str和unicode两种类型，str有各种编码区别，unicode是没有编码的标准形式。unicode通过编码转化成str，str通过解码转化成unicode。所以当源文件中出现非ascii字符需要通过`# -*- coding:utf-8 -*-`告诉它什么编码

3.x中将字符串和字节序列做了区别，字符串str是字符串标准形式与2.x中unicode类似，bytes类似2.x中的str有各种编码区别。bytes通过解码转化成str，str通过编码转化成bytes。

可以用sys.getdefaultencoding()得到默认编码

`#-*- coding:utf-8 -*-`告诉python解释器(包括python2与python3)：当遇到非ascii字符串时，用什么编码来存储这个字符串

在python3中，字符串采用"UTF-8"来存储，但是它是一个"str"，需要encode以后才能得到相应的bytes
```python
    Python 3.6.1 (default, Jul  8 2017, 21:11:43) 
    [GCC 4.8.5 20150623 (Red Hat 4.8.5-11)] on linux
    Type "help", "copyright", "credits" or "license" for more information.
    >>> ss="你好" 
    >>> ss=b"你好"
    File "<stdin>", line 1
    SyntaxError: bytes can only contain ASCII literal characters.
    >>> ss="你好"
    >>> print(ss.encode("UTF-8"))
    b'\xe4\xbd\xa0\xe5\xa5\xbd'
    >>> print(ss.decode("UTF-8"))
    Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
    AttributeError: 'str' object has no attribute 'decode'
    >>> print(ss.encode("GBK"))
    b'\xc4\xe3\xba\xc3'
```

在python2中，字符串默认有各种编码，需要decode才能得到unicode序列  
当对unicode解码时，python2会转化成`b.encode('ascii').decode('GBK')`这样的形式，所以得到下面最后一行的报错
```python
    Python 2.7.5 (default, Nov  6 2016, 00:28:07) 
    [GCC 4.8.5 20150623 (Red Hat 4.8.5-11)] on linux2
    Type "help", "copyright", "credits" or "license" for more information.
    >>> ss="你好"
    >>> print(type(ss))
    <type 'str'>
    >>> print(ss.decode('utf-8'))
    你好
    >>> us=ss.decode('utf-8')
    >>> us
    u'\u4f60\u597d'
    >>> us.decode('gbk')
    Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
    UnicodeEncodeError: 'ascii' codec can't encode characters in position 0-1: ordinal not in range(128)
```
python2与python3字符串前面的b，比如b"hello"  
在python2中`b"hello"`与`"hello"`无任何区别，python2中的b前缀无任何意义  
在python3中`b"hello"`表示它是一个bytes的序列,bytes序列只能是ascii字符，所以`b"你好"`会报错