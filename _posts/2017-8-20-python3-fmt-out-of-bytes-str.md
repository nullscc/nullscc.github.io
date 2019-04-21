---
layout: post
title: 'python3中对bytes与str的格式化输出的区别'
tags: python
---
今天在python3中用到了socketserver库，此库向网络中发送数据的接口需要被发送的数据是bytes  
所以在发送以'\r\n'为行尾标识的字符串时出现了意想不到的结果。

原因是因为格式化输出在python3中对bytes和str处理是不一样的。  

直接看代码及输出：
```python
    #!/usr/bin/python3
    #-*- coding:utf-8 -*-

    bs = b'hello\r\n'
    ds = bs.decode("UTF-8")
    ss = 'hello\r\n'
    print("bs:", bs, len(bs))
    print("ds:", ds, len(ds))
    print("ss:", ss, len(ss))

    print("-----------------")

    print("bs:{}".format(bs))
    print("bs:%s" % bs)
```
输出：
```bash
    [null@localhost py3]$ ./bytes.py 
    bs: b'hello\r\n' 7
    ds: hello
    7
    ss: hello
    7
    -----------------
    bs:b'hello\r\n'
    bs:b'hello\r\n'
```