---
layout: post
title: 'python的sorted函数'
tags: python
---
这里主要介绍python3下的sorted函数，它跟python2下有很大的不同，主要不同点在于python2下有个cmp参数，而在python3中完全废弃了。但是python3中的sorted比python2中的sorted在数据量比较大的时候排序效率高的多

python3中的sorted没有cmp参数数，函数原型为：
```python
	sorted(iterable, *, key=None, reverse=False)
```

在python3中没有cmp参数了，当我们需要使用一个旧式的函数时，使用`functools.cmp_to_key()`去转换一个旧式的比较函数，就是的比较函数是指接收两个参数的那种比较函数，而不是像key这种直接返回一个需要比较的元素，以下是使用key函数的优势(没懂具体的意思，反正就是说在需要比较的元素很多的时候key函数能显著提升，因为key函数对每个输入记录之调用1次）
> This(key) is easier to use and faster to run. When using the cmp parameter, the sorting compares pairs of values, so the compare-function is called multiple times for every item. The larger the set of data, the more times the compare-function is called per item. With the key function the sorting instead keeps the key value for each item and compares those, so the key function is only called once for every item. This results in much faster sorts for large sets of data.

内建的sorted函数能保证稳定：两个元素值相等时，他们的前后相对位置不变

## 介绍下list.sort

list.sort的特点是：

* 原地排序，直接在原列表的基础上排序
* 只对列表起作用，因为它是列表这个类的实例方法

一个例子：
```python
	>>> a = [5, 2, 3, 1, 4]
	>>> a.sort()
	>>> a
	[1, 2, 3, 4, 5]
```

## list.sort和sorted的区别

* sorted会返回一个可迭代对象，而list.sort是原地排序
* list.sort只能对列表进行排序，而sorted能对所有的可迭代对象排序

## list.sort和sorted的共同点

* 都可以对列表排序
* 都支持列表的正排、反排 
* 稳定排序，多次排序，元素的前后相对位置是固定的

## Operator模块
由于sorted使用key函数，它有一些很通用的key函数，比如返回这些属性的key函数：可索引对象的第n个元素，类的属性，python提供了一个Operator模块来做这些通用的事。

Operator主要提供了三个函数(itemgetter、attrgetter、methodcaller、)来做这个事：
首先列两个类，用来写下面的例子：
```python
	student_tuples = [
		('john', 'A', 15),
		('jane', 'B', 12),
		('dave', 'B', 10),
	]
	class Student:
		def __init__(self, name, grade, age):
			self.name = name
			self.grade = grade
			self.age = age
			def __repr__(self):
				return repr((self.name, self.grade, self.age))
```

例子：
```python
	>>> from operator import itemgetter, attrgetter
	>>>
	>>> sorted(student_tuples, key=itemgetter(2))
	[('dave', 'B', 10), ('jane', 'B', 12), ('john', 'A', 15)]
	>>>
	>>> sorted(student_objects, key=attrgetter('age'))
	[('dave', 'B', 10), ('jane', 'B', 12), ('john', 'A', 15)]
```

还可以多重排序
```python
	>>> sorted(student_tuples, key=itemgetter(1,2))
	[('john', 'A', 15), ('dave', 'B', 10), ('jane', 'B', 12)]
	>>>
	>>> sorted(student_objects, key=attrgetter('grade', 'age'))
	[('john', 'A', 15), ('dave', 'B', 10), ('jane', 'B', 12)]
```

稳定排序的支持，sorted的排序是稳定的排序(保护原始顺序)，可以利用这个特性做一些复杂的排序，比如先对age做升序排列，然后对grade做降序排列
```python
	>>> s = sorted(student_objects, key=attrgetter('age'))     # sort on secondary key
	>>> sorted(s, key=attrgetter('grade'), reverse=True)       # now sort on primary key, descending
	[('dave', 'B', 10), ('jane', 'B', 12), ('john', 'A', 15)]
```

## DSU
DSU是Decorate-Sort-Undecorate，命令主要来源于以下步骤：

1. 初始的列表进行转换，获得用于排序的新值。
2. 将转换为新值的列表进行排序。
3. 还原数据并得到一个排序后仅包含原始值的列表。

看以下代码：
```python
	>>> decorated = [(student.grade, i, student) for i, student in enumerate(student_objects)]
	>>> decorated.sort()
	>>> [student for grade, i, student in decorated]               # undecorate
	[('john', 'A', 15), ('jane', 'B', 12), ('dave', 'B', 10)]
```

它利用了元组是按照字典序：先比较第一个，相同则比较第二个，以此类推

很多情况下是不需要包含下标的，不过这样做有一些好处：

* 排序是稳定的，如果有两个相同的key，那么排序后的列表会保留它们的顺序
* 原始元素可以是不可比较的

但是在python排序提供key函数之后，这个技巧已经不常用了。

## 旧时排序函数到key函数的转变
python3中没有cmp参数，所以有时候我们需要一个从旧式的cmp到key的转换

比如这样一个旧式函数：
```python
	def reverse_numeric(x, y):
		return y - x
```

可以定义这样一个函数来让cmp转换到key(当然使用`functools.cmp_to_key()`是更明智的选择)：
```python
	def cmp_to_key(mycmp):
	    'Convert a cmp= function into a key= function'
	    class K:
	        def __init__(self, obj, *args):
	            self.obj = obj
	        def __lt__(self, other):
	            return mycmp(self.obj, other.obj) < 0
	        def __gt__(self, other):
	            return mycmp(self.obj, other.obj) > 0
	        def __eq__(self, other):
	            return mycmp(self.obj, other.obj) == 0
	        def __le__(self, other):
	            return mycmp(self.obj, other.obj) <= 0
	        def __ge__(self, other):
	            return mycmp(self.obj, other.obj) >= 0
	        def __ne__(self, other):
	            return mycmp(self.obj, other.obj) != 0
	    return K
```
然后你可以在python3中这样做：
```python
	sorted([5, 2, 4, 1, 3], key=cmp_to_key(reverse_numeric))
```
## 其他注意事项

* 针对时区相关的排序：使用`locale.strxfrm()`作为key函数，或者使用`locale.strcoll()`作为旧式比较函数
* `reverse`参数始终保持排序稳定性，有趣的是，使用两次`reversed`函数可以模拟同样的效果
* 排序是使用`__lt__`来比较两个对象，所以，可以增加一个`__lt__`方法
	
		>>> Student.__lt__ = lambda self, other: self.age < other.age
		>>> sorted(student_objects)
		[('dave', 'B', 10), ('jane', 'B', 12), ('john', 'A', 15)]

* key参数指定的函数可以不直接依赖于被比较的对象，它也可以依赖于外部的资源。比如下面的例子：
```python
		>>> students = ['dave', 'john', 'jane']
		>>> newgrades = {'john': 'F', 'jane':'A', 'dave': 'C'}
		>>> sorted(students, key=newgrades.__getitem__)
		['jane', 'dave', 'john']
```