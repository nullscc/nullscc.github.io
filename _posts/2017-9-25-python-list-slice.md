---
layout: post
title: 'python的列表的切片'
tags: python
---
切片操作大家都知道，很简单。但是有些细微的地方可能有些人从来没遇到过。

这里只说说列表的切片

就看以下程序：
```python
    L = [1, 2, 3, 4, 5, 6, 7, 8, 9]
    print(L[1:4])
    print(L[-2::-2])
    print(L[-2:2:-2])
    print(L[2:2:-2])
```
有些人是不是有点懵逼了？

这里明白以下几点就好了：

* 切片操作：[start:end:step],start表示起点索引，end表示终点索引，step表示步长
* 切片操作是左边关闭右边打开
* 如果切片的起始或者终点索引为负，那么表示倒数第几个
* 如果步长为负，则生长方向和默认的是相反的
* 步长可以省略