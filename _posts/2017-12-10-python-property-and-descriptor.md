---
layout: post
title: 'python的property(描述器descriptor相关知识)'
tags: python
---
今天cookbook，发现里面的property看的有点懵，所以去了解了下它的实现，看了以后就感觉挺简单的，只是灵活性太高，有点绕而已

其实它使用了描述器来实现功能，所以知道什么是描述器就好办了。

什么是描述器？ 
有以下三个方法之一:`__set__`、`__get__`、`__delete__`

属性访问的默认顺序：

1. `a.__dict__['x']`
2. `type(a).__dict__['x']`
3. `type(a)的基类的__dict__['x']`

当类或者实例的属性成了描述器，那么它会重写上述类属性的访问顺序

有哪些内置的描述器？

* super(通过访问`__mro__`)
* property
* function
* staticmethod
* classmethod

有两类描述器：

1. non-data descriptor : 只定义了 `__get__` 的描述器
2.  data descriptor: 既定义了`__get__`又定义了 `__set__`

那么non-data和data的区别是什么呢？ 
它们决定了优先级，如果data descriptor和实例的`__dict__`属性相同，那么data descriptor的优先级高  
如果non-data descriptor和实例的`__dict__`属性相同，那么实例属性优先  
注: 如果存在`__getattr__()`那么它的访问优先级是最低的

那么描述器是怎么影响属性访问的呢？

1. 描述器影响属性访问是通过`object.__getattribute__()`来实现的，所以确保你没有重写此方法
2. 对于实例来说，b.x 会被转换成`type(b).__dict__['x'].__get__(b, type(b))`
3. 对于类来说，B.x 会被转换为`B.__dict__['x'].__get__(None, B)`

Property类是怎么实现的?以下是它的python代码等价物:
```python
	class Property(object):
	    "Emulate PyProperty_Type() in Objects/descrobject.c"
	
	    def __init__(self, fget=None, fset=None, fdel=None, doc=None):
	        self.fget = fget
	        self.fset = fset
	        self.fdel = fdel
	        if doc is None and fget is not None:
	            doc = fget.__doc__
	        self.__doc__ = doc
	
	    def __get__(self, obj, objtype=None):
	        if obj is None:
	            return self
	        if self.fget is None:
	            raise AttributeError("unreadable attribute")
	        return self.fget(obj)
	
	    def __set__(self, obj, value):
	        if self.fset is None:
	            raise AttributeError("can't set attribute")
	        self.fset(obj, value)
	
	    def __delete__(self, obj):
	        if self.fdel is None:
	            raise AttributeError("can't delete attribute")
	        self.fdel(obj)
	
	    def getter(self, fget):
	        return type(self)(fget, self.fset, self.fdel, self.__doc__)
	
	    def setter(self, fset):
	        return type(self)(self.fget, fset, self.fdel, self.__doc__)
	
	    def deleter(self, fdel):
	        return type(self)(self.fget, self.fset, fdel, self.__doc__)
```

函数是怎么实现方法访问的？(其实函数和方法区别就是它们的第一个参数是类实例本身)
```python
	class Function(object):
	    . . .
	    def __get__(self, obj, objtype=None):
	        "Simulate func_descr_get() in Objects/funcobject.c"
	        if obj is None:
	            return self
	        return types.MethodType(self, obj)
```
其中`types.MethodType`的第一个参数是函数，而第二个参数是要绑定到哪个实例

好了，原理说完了，拿个例子来总结下:
```python
	class Student(object):
	
	    @property
	    def score(self):
	        return self._score
	
	    @score.setter
	    def score(self, value):
	        if not isinstance(value, int):
	            raise ValueError('score must be an integer!')
	        if value < 0 or value > 100:
	            raise ValueError('score must between 0 ~ 100!')
	        self._score = value
```
上面定义了一个score函数，但是用`@property`修饰了，那么在类定义时就会做这样一个转换：
```python
	score = property(score)
```

好，那么score本来是一个函数，那么现在变成一个descriptor了，所以当我们使用`Student`的实例t这样访问它时(t.score)，就变成`type(t).__dict__['score'].__get__(t, type(t))`

我们知道，`type(t).__dict__['score']`就是property类的一个实例，而这个实例就是score，所以会访问property的`__get__`方法，参见上面的python版本的property的实现可知，它会调用fget方法。

那么fget方法是什么呢？就是在类定义时装饰器做的转换时的`score`函数，它会返回`self._score`

同理，由于之前调用过`@property`了，所以score是类:property的一个实例，所以`@score.setter`会转换为:
```python
	score = score.setter(score)		
```

由于原先score已经是一个描述器了，而且有`__get__`方法，经过这次转换，score就成了data descriptor了。

所以调用t.score会调用`type(t).__dict__['score'].__set__(t, value)`