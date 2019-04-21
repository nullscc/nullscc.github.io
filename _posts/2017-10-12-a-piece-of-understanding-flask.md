---
layout: post
title: 'flask的一些零碎理解'
tags: python
---
在学习使用flask的时候有些概念比较难理解，比如应用／请求上下文、g等，这里记录下一些概念的我的理解的整理，当然也包括一些小技巧。

比较零散，介于浅显和晦涩之间的一些东西。

## 配置
flask支持几种配置方式：

* `app.config.from_pyfile(config_filename)`，从一个py文件导入，它可以是相对路径也可以是绝对路径
* `app.config.from_object('yourapplication.default_settings')`，import包或模块的路径方式
* `app.config.from_envvar('YOURAPPLICATION_SETTINGS')`，通过环境变量加载，比如你的环境变量是这样的：

		`export YOURAPPLICATION_SETTINGS='/path/to/config/file'`

怎么访问这些环境变量呢？以下展示下调试模式的设置：
```python
    app.config['DEBUG'] = True
    app.debug = True
```

更新环境变量：
```python
    app.config.update(
        DEBUG=True,
        SECRET_KEY='...'
    )
```

## 视图函数的返回值
视图函数的返回值会被自动转换为一个响应对象。如果返回值是一个字符串， 它被转换为该字符串为主体的、状态码为`200 OK`的 、MIME 类型是 `text/html` 的响应对象。Flask 把返回值转换为响应对象的逻辑是这样：

* 如果返回的是一个合法的响应对象，它会从视图直接返回。
* 如果返回的是一个字符串，响应对象会用字符串数据和默认参数创建。
* 如果返回的是一个元组，且元组中的元素可以提供额外的信息。这样的元组必须是 (response, status, headers) 的形式，且至少包含一个元素。 status 值会覆盖状态代码， headers 可以是一个列表或字典，作为额外的消息标头值。
* 如果上述条件均不满足， Flask 会假设返回值是一个合法的 WSGI 应用程序，并转换为一个请求对象。

如果你想在视图里操纵上述步骤结果的响应对象，可以使用 make_response() 函数。


## “全局变量”/上下文
flask中有至少四种全局变量()，说是全局变量其实是不准确的，我自己解释不清，直接引用别人的话吧。

> 经常会碰到直接调用像current_app、request、session、g等变量。这些变量看起来似乎是全局变量，但是实质上这些变量并不是真正意义上的全局变量。如果将这些变量设置为全局变量，试想一下，多个线程同时请求程序访问这些变量时势必会相互影响。但是如果不设置为全局变量，那在编写代码时每次都要显式地传递这些变量也是一件非常令人头疼的事情，而且还容易出错。为了解决这些问题，flask设计时采取线程隔离的思路，也就是说在一次请求的一个线程中可以将其设置为全局变量，但是仅限于请求的这个线程内部，不同线程通过“线程标识符”来区别。这样就不会影响到其他线程的请求。

嗯，核心就是"线程"隔离的全局变量，这里的线程我打引号是因为它可能和我们平常看到的线程不是一回事儿～

request，它将浏览器的一些请求信息都放在这个线程隔离的局部变量中，比如：get的参数`request.args`、post提交的数据`request.form`、cookie`request.cookies`、header`request.headers`

再说说session和g，session你就把它当作全局变量用就是了（这样说肯定不对～），g的话相当于是给开发者使用的线程隔离的全局变量，什么意思呢？它在一个app的单次请求内可以多次访问/设置。在另外一个请求中g的值就和上一个请求中的g的值无关了。和request很像。

关于g的额外补充：

> 只在一个请求内，从一个函数到另一个函数共享数据，全局变量并不够好。因为这在线程环境下行不通。 Flask 提供了一个特殊的对象来确保只在活动的请求中有效，并且每个请求都返回不同的值。

应用上下文，它的存在是为了让一个进程里可以运行多个app，它是一个代理，从不同应用/请求中访问`current_app`会得到不同的值，那么如果在shell/不在请求中怎么得到应用上下文?  
有以下两种方式：
```python
    >>> ctx=app.app_context()  
    >>> ctx.push() 
    # 这样在pop之前都可以使用依赖于应用上下文的代码了

    with app.app_context():
        ...
```
请求上下文可能是最好理解的了，它是存在与一个请求期间有意义的线程隔离的全局变量，么如果在shell/不在请求中怎么得到请求上下文?  
```python
    with app.test_request_context('/hello', method='POST'):
        # now you can do something with the request until the
        # end of the with block, such as basic assertions:
        assert request.path == '/hello'
        assert request.method == 'POST'


    # 另一种可能是：传递整个 WSGI 环境给 request_context() 方法:
    from flask import request

    with app.request_context(environ):
        assert request.method == 'POST'
```

### 代理
current_app是一个代理，所以要得到真实的app对象：

    app = current_app._get_current_object()

## flask中的应用
flask中的应用是什么？，flask中的应用我认为就是一个项目，一个应用就是任意个蓝本的集合，蓝本一般对应于一个功能模块

## 请求钩子
flask中的请求钩子有四种

* `before_first_request`
* `after_request`
* `before_request`
* `teardown_request`

除了`before_first_request`外，另外三个钩子还有针对于蓝本的版本

除了`teardown_request`，另外三个直接看名字就能看出来它们的功能了，`teardown_request`的意义在于它总会执行，即使前面的执行流程抛出了异常。

以下描述了请求钩子的行为模式：

1. 在每个请求之前，执行 before_request() 上绑定的函数。 如果这些函数中的某个返回了一个响应，其它的函数将不再被调用。任何情况下，无论如何这个返回值都会替换视图的返回值。
2. 如果 before_request() 上绑定的函数没有返回一个响应， 常规的请求处理将会生效，匹配的视图函数有机会返回一个响应。
3. 视图的返回值之后会被转换成一个实际的响应对象，并交给 after_request() 上绑定的函数适当地替换或修改它。
4. 在请求的最后，会执行 teardown_request() 上绑定的函数。这总会发生，即使在一个未处理的异常抛出后或是没有请求前处理器执行过 （例如在测试环境中你有时会想不执行请求前回调）。

## session
flask种的session是基于客户端cookie的，它是一个客户端session，只不过它通过密钥加密的形式让客户端只能看到而不能修改它，我们用`session.get`能得到session的原因是：它得到的session数据也是客户端传过来的cookie

## 对于python3
想在python3中使用flask，最好使用最新的flask(>0.12)，而且python的版本最好大于3.3