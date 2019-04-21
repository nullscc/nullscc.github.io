---
layout: post
title: 'python中的装饰器'
tags: python
---
记得刚学python那会儿，看到装饰器直接跳过，当时就想，这是个虾米玩意儿，看着好恐怖的样子，这么高端，应该用不上。

现在看来它就是python的一个语法糖的magic。我的理解，装饰器就是一个函数，只不过这个函数通过`@`这样一个语法糖加上动态语言的特性让它看着怪吓人的。

来看个最简单的：
```python
    def decorator(func):
        print("start decorator")
        def wrapper():
            print("excute func")
            return func()
        return wrapper

    @decorator
    def hello():
        print("hello decorator")

    print("before hello")
    hello()

    '''
    output:

        start decorator
        before hello
        excute func
        hello decorator
    '''
```

看输出我们知道，装饰器在声明的就会被执行。其实上面的`@decorator`的效果就是：
```python
    hello = decorator(hello)
```

来让hello函数带上参数：
```python
    def decorator(func):
        print("start decorator")
        def wrapper(arg):
            print("excute func")
            return func(arg)
        return wrapper

    @decorator
    def hello(arg):
        print("hello decorator arg:{}".format(arg))

    print("before hello")
    hello("hi")

    '''
    output:

        start decorator
        before hello
        excute func
        hello decorator arg:hi
    '''
```

带上参数后，调用`hello("hi")`就等价于：
```python
    decorator(hello)("hi")
```

那么同理，`hello`可以带任意参数
```python
    def decorator(func):
        print("start decorator")
        def wrapper(*arg, **kwargs):
            print("excute func")
            return func(*arg, **kwargs)
        return wrapper

    @decorator
    def hello(*arg, **kwargs):
        print("hello decorator arg:{0} kwargs:{1}".format(arg, kwargs))

    print("before hello")
    hello("hi", one=1, two=2)

    '''
    output:

        start decorator
        before hello
        excute func
        hello decorator arg:('hi',) kwargs:{'one': 1, 'two': 2}
    '''
```

那么我们是不是有时看到有些装饰器本身也是带函数的呢？它是怎么实现的？其实也很简单：
```python
    def decorator(decorator_arg):
        print("start decorator")
        def mydecorator(func):
            def wrapper(*arg, **kwargs):
                print("excute func, decorator_arg is:{0}".format(decorator_arg))
                return func(*arg, **kwargs)
            return wrapper
        return mydecorator

    @decorator("decorator arg test")
    def hello(*arg, **kwargs):
        print("hello decorator arg:{0} kwargs:{1}".format(arg, kwargs))

    print("before hello")
    hello("hi", one=1, two=2)

    '''
    output:

        start decorator
        before hello
        excute func, decorator_arg is:decorator arg test
        hello decorator arg:('hi',) kwargs:{'one': 1, 'two': 2}
    '''
```

上面的hello函数调用等价于：
```python
    decorator("decorator arg test")(hello)("hi", one=1, two=2)
```
那么多个装饰器的情况呢？看看代码：
```python
    def decorator(decorator_arg):
        print("start decorator")
        def mydecorator(func):
            def wrapper(*arg, **kwargs):
                print("excute func, decorator_arg is:{0}".format(decorator_arg))
                return func(*arg, **kwargs)
            return wrapper
        return mydecorator

    def decorator2():
        def mydecorator(func):
            def wrapper(*arg, **kwargs):
                print("in decorator2")
                return func(*arg, **kwargs)
            return wrapper
        return mydecorator

    @decorator2()
    @decorator("decorator arg test")
    def hello(*arg, **kwargs):
        print("hello decorator arg:{0} kwargs:{1}".format(arg, kwargs))

    print("before hello")
    hello("hi", one=1, two=2)

    '''
    output:

        start decorator
        before hello
        in decorator2
        excute func, decorator_arg is:decorator arg test
        hello decorator arg:('hi',) kwargs:{'one': 1, 'two': 2}
    '''
```
上面的hello函数调用等价于：
```python
    decorator2(decorator)("decorator arg test")(hello)("hi", one=1, two=2)
```
好吧，有点晕了，好好理解几遍应该就还好了。

然后上面所有的装饰器都有一个问题，就是`hello.__name__`的值成为`wrapper`了。  
这肯定有问题的，那么怎么解决呢？看下面代码(使用`functools.wraps`装饰器来将它的`__name__`属性修改回正确的值)：
```python
    import functools

        def decorator(decorator_arg):
            print("start decorator")
            def mydecorator(func):
                @functools.wraps(func)
                def wrapper(*arg, **kwargs):
                    print("excute func, decorator_arg is:{0}".format(decorator_arg))
                    return func(*arg, **kwargs)
                return wrapper
            return mydecorator

        def decorator2():
            def mydecorator(func):
                @functools.wraps(func)
                def wrapper(*arg, **kwargs):
                    print("in decorator2")
                    return func(*arg, **kwargs)
                return wrapper
            return mydecorator

        @decorator2()
        @decorator("decorator arg test")
        def hello(*arg, **kwargs):
            print("hello decorator arg:{0} kwargs:{1}".format(arg, kwargs))

        print("before hello")
        hello("hi", one=1, two=2)
```
最后，怎么给装饰器加上装饰器，让装饰器能带任意的参数，这个有点晕，一般人一下子肯定写不出来，me too~
```python
    def decorator_with_args(decorator_to_enhance):
        def decorator_maker(*args, **kwargs):
            def decorator_wrapper(func):
                return decorator_to_enhance(func, *args, **kwargs)
            return decorator_wrapper
        return decorator_maker

    @decorator_with_args
    def decorated_decorator(func, *args, **kwargs):
        def wrapper(function_arg1, function_arg2):
            print("Decorated with", args, kwargs)
            return func(function_arg1, function_arg2)
        return wrapper

    @decorated_decorator(42, 404, 1024)
    def decorated_function(function_arg1, function_arg2):
        print("Hello", function_arg1, function_arg2)

    decorated_function("python and", "go")
    '''
    output:

        Decorated with (42, 404, 1024) {}
        Hello python and go
    '''
```
当调用`decorated_function`时，等价于：
```python
    decorated_function = decorated_decorator(42, 404, 1024)(decorated_function) = decorator_with_args(decorated_decorator)(42, 404, 1024)(decorated_function)
```