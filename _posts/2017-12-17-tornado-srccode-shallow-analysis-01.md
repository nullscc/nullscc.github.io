---
layout: post
title: 'tornado源码浅析1-大纲'
tags: python
---
tornado的源码其实算是比较少的了，可能是因为不像c系静态语言那样很多轮子要自己造吧。

由于公司使用的是tornado，而且业务量也不小，依据以前用skynet的经验来看，这种框架不知道实现机制的话是很危险的。
所以看看它的源码自然提上日程了，原本以为会有很多黑魔法，没想到啥黑魔法都没有，勉强算得上黑魔法的东西可能就是那个`Configurable`类了

本以为要花很长时间才能看懂，没想到只花了一天多一点就把机制弄的差不多了。这里先从总体上说说。

tornado的框架我把它分为两大块:

1. IOStream，它提供数据支撑，比如网络数据的读取都在这里了
2. IOLoop, 它配合python的生成器提供了真正的协程语义，并且它还处理epoll/kqueue/select

所以看它的源码也是从这两大块来看，这里先说先基础的东西。

## Configurable 
这个类很巧妙，它是一个基类，那么这个类用来干嘛的呢？从名字看它是一个可配置的东东，可是，它配置的是个虾米东西？

来看看源码:
```python
    """
    Configurable subclasses must define the class methods
    `configurable_base` and `configurable_default`, and use the instance
    method `initialize` instead of ``__init__``.
    """
    def __new__(cls, *args, **kwargs):
        base = cls.configurable_base()
        init_kwargs = {}
        if cls is base:
            impl = cls.configured_class()
            if base.__impl_kwargs:
                init_kwargs.update(base.__impl_kwargs)
        else:
            impl = cls
        init_kwargs.update(kwargs)
        if impl.configurable_base() is not base:
            return impl(*args, **init_kwargs)
        instance = super(Configurable, cls).__new__(impl)
        instance.initialize(*args, **init_kwargs)
        return instance

    @classmethod
    def configured_class(cls):
        # type: () -> type
        """Returns the currently configured class."""
        base = cls.configurable_base()
        # Manually mangle the private name to see whether this base
        # has been configured (and not another base higher in the
        # hierarchy).
        if base.__dict__.get('_Configurable__impl_class') is None:
            base.__impl_class = cls.configurable_default()
        return base.__impl_class
```
我们知道`__new__`是用来创建一个实例的，而这里的`__new__`是通过`configured_class`来创建实例的，而它又是通过`cls.configurable_default`来创建实例的，所以事情很清楚了：  
这个类的作用就是可以定制类的基类，意思就是说，如果一个类以`Configurable`为基类，那么只要它定义了`configurable_base`和`configurable_default`，那么它就可以通过自定义这两个方法来自定义来决定创建哪个类的实例。

那么这里为啥要整的这么麻烦呢？从代码里我看到两个作用:

* 配置不同操作系统上是使用epoll、kqueue还是select
* 针对不同的python版本来决定是自己实现(当然是框架实现啦)一个IOLoop还是使用python3本身的asyncio的IOLoop

虽然我还没看过asyncio的原理，但是我基本上就认为在python3中tornado的协程就是python3本身的协程:  
其实不用认为，它就是～，为什么？platform/asyncio.py已经出卖它了，而且tornado官方文档也明确指出，在python3中async与await是"native coroutine"

## tornado中的协程
这篇文章只是一个大纲，为什么要说协程呢？

因为python本身是没有协程支持的，什么？你说生成器？生成器还不算协程，因为它没有自动从yield中恢复执行的能力。那么tornado中为什么有这个能力？那是因为tornado框架通过IOLoop来帮它恢复执行了。  
具体的原理后面会开文章说，这里我们只需要记住一点：看到被`gen.coroutine ... yield`修饰的函数，我们要把它理解成为让出执行(协程语义)，而不是理解成生产数据(生成器语义)。

什么是让出执行？也不知道有没有公认的说法，反正我就是这么理解的：  
代码执行到yield，就不会顺序的继续往下执行了，得满足某个条件才行。如果你知道go或者lua的话，不用想了，就是那个东东~~~    
虽然生成器也可以说成是让出执行，但是它必须手动恢复执行(调用next或者迭代它)，所以我一直只把yield关键字看成是生产数据。

嗯，先这样，既然是大纲嘛，不说的云里雾里算啥大纲～