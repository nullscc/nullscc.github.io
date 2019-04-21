---
layout: post
title: 'tornado源码浅析3-IOLoop与协程'
tags: python
---
我们知道tornado是可以实现单进程很高的并发量的，那么它是怎么做到的呢？这要看看IOLoop的实现了。

为了便于理解这里我们以python2.7为基础分析，因为在python3中IOLoop实际上就是asyncioloop，代码上体现为：
```python
    @classmethod
    def configurable_default(cls):
        if asyncio is not None:
            from tornado.platform.asyncio import AsyncIOLoop
            return AsyncIOLoop

 				class AsyncIOLoop(BaseAsyncIOLoop):
  				   def initialize(self, **kwargs):
 				       loop = asyncio.new_event_loop()
 				       try:
 				           super(AsyncIOLoop, self).initialize(loop, close_loop=True, **kwargs)
								
       return PollIOLoop
```

具体的解释在大纲里面说了～，从这里可以看出为什么tornado文档中说`async ... await`比`gen.coroutine`快的原因了。(从后面可以看出gen.coroutine是纯粹用python写了一个IOLoop来管理生成器来达到协程的执行效果的)

所以这里我们使用`PollIOLoop`作为IOLoop的实现来分析。看看它的`start()`函数：
```python
    def start(self):
        if self._running:
            raise RuntimeError("IOLoop is already running")
        if os.getpid() != self._pid:
            raise RuntimeError("Cannot share PollIOLoops across processes")
                try:
            while True:
                ncallbacks = len(self._callbacks)
				due_timeouts = []
                if self._timeouts:
                    now = self.time()
                    while self._timeouts:
              			... 
                for i in range(ncallbacks):
                    self._run_callback(self._callbacks.popleft())
                for timeout in due_timeouts:
                    if timeout.callback is not None:
                        self._run_callback(timeout.callback)
                due_timeouts = timeout = None

                if self._callbacks:
                    poll_timeout = 0.0
                elif self._timeouts:
                    poll_timeout = self._timeouts[0].deadline - self.time()
                    poll_timeout = max(0, min(poll_timeout, _POLL_TIMEOUT))
                else:
                    # No timeouts and no callbacks, so use the default.
                    poll_timeout = _POLL_TIMEOUT

                if not self._running:
                    break

                if self._blocking_signal_threshold is not None:
                    # clear alarm so it doesn't fire while poll is waiting for
                    # events.
                    signal.setitimer(signal.ITIMER_REAL, 0, 0)

                try:
                    event_pairs = self._impl.poll(poll_timeout)
                self._events.update(event_pairs)
                while self._events:
                    fd, events = self._events.popitem()
                    try:
                        fd_obj, handler_func = self._handlers[fd]
                        handler_func(fd_obj, events)
               fd_obj = handler_func = None

        finally:
			...
```
可以看出这里它做的事有：

1. 定时处理回调
2. 处理`self._callbacks`队列
3. 处理网络事件(epoll/kqueue/select)

这里只说说第二点，它才是协程异步并发的关键，其中`self._callbacks`是一个无限大小的队列，这里不断从这个队列里面取出已经加入的callback进行调用，那么什么时候调用呢？来看看吧～

这里为了简便，从网络上摘抄下来一段代码：
```python
	import tornado.ioloop
	from tornado.gen import coroutine
	from tornado.concurrent import Future
	
	@coroutine
	def asyn_sum(a, b):
	    future = Future()
	
	    def callback(a, b):
	        print("calculating the sum of %d+%d:"%(a,b))
	        future.set_result(a+b)
	    tornado.ioloop.IOLoop.instance().add_callback(callback, a, b)
	
	    result = yield future
	
	    print("after yielded")
	    print("the %d+%d=%d"%(a, b, result))
	
	def main():
	    asyn_sum(2,3)
	    tornado.ioloop.IOLoop.instance().start()
	
	if __name__ == "__main__":
	    main()
```
这段代码很简短，弄懂这里，tornado协程的实现就差不多有个概念了。

最核心的代码还是在`@coroutine`这里，看看它的源码(在gen.py文件)：
```python
	def coroutine(func, replace_callback=True):
	    return _make_coroutine_wrapper(func, replace_callback=True)

	def _make_coroutine_wrapper(func, replace_callback):
	    wrapped = func
	    @functools.wraps(wrapped)
	    def wrapper(*args, **kwargs):
	        future = _create_future()
	        try:
	            result = func(*args, **kwargs)
	        else:
	            if isinstance(result, GeneratorType):
	                try:
	                    yielded = next(result)
	                except (StopIteration, Return) as e:
	                    future_set_result_unless_cancelled(future, _value_from_stopiteration(e))
	                else:
	                    _futures_to_runners[future] = Runner(result, future, yielded)
	                yielded = None
	                try:
	                    return future
	                finally:
	                    future = None
	        future_set_result_unless_cancelled(future, result)
	        return future
	    return wrapper
```
这里为了分析方便删减了一些暂时不用去关心的代码。如果对装饰器不太熟悉的话还是先去了解装饰器吧～

`_make_coroutine_wrapper`的流程基本可以概括为:

1. 创建一个future(它是一个Future类，具体是什么待会儿说)
2. 调用`result = func(*args, **kwargs)`，这里就等价于 result=`async_sum(2, 3)`, 很显然result是个`GeneratorType`,所以`asyn_sum`的执行流程停在`result = yield future`，我们接下来关注它是怎么回来的就行啦。
3. 由于result是个`GeneratorType`，所以会调用`yielded = next(result)`，这样就等价于`yielded=future`，所以代码会执行`_futures_to_runners[future] = Runner(result, future, yielded)`

很显然，核心工作在Runner类中，来看看Runner类的实现吧:
```python
    class Runner(object):
        def __init__(self, gen, result_future, first_yielded):
            self.gen = gen
            self.result_future = result_future
            self.future = _null_future
            ...
            self.io_loop = IOLoop.current()
            if self.handle_yield(first_yielded):
                gen = result_future = first_yielded = None
                self.run()

            def handle_yield(self, yielded):
                self.future = convert_yielded(yielded)
                def inner(f):
                    f = None # noqa
                    self.run()
                self.io_loop.add_future(self.future, inner)
                return False
```

注意这段代码精简了非常多，只是为了分析它是怎么恢复yield的执行，而不用手动调用`next`或者迭代它。

在上面的`handle_yield`方法的第一行，`self.future`就被赋值为`async_num`中的第一次yield的future，这很重要～

然后调用`self.io_loop.add_future(self.future, inner)`，所以来看看`self.io_loop.add_future`干了啥
```python
	self.io_loop.add_future(gen.convert_yielded(future), lambda f: f.result())
	
	    def add_future(self, future, callback):
	        assert is_future(future)
	        callback = stack_context.wrap(callback)
	        future.add_done_callback(lambda future: self.add_callback(callback, future))
	        
	            def add_done_callback(self, fn):
	                if self._done:
	                    fn(self)
	                else:
	                    self._callbacks.append(fn)
	 
	            def add_callback(self, callback, *args, **kwargs):
	                if self._closing:
	                    return
	                self._callbacks.append(functools.partial(stack_context.wrap(callback), *args, **kwargs))
```
可见它做的工作是:

1. 将`lambda future: self.add_callback(callback, future)`加入到`Future`的`_callbacks`队列
2. 而`self.add_callback`的也是将`functools.partial(stack_context.wrap(callback), *args, **kwargs)`加入到`_callback`队列中，而callback就是上面的inner函数的闭包

注意第1条中的`_callbacks`队列是Future类的队列，而第2条中的`_callbacks`是IOLoop的队列，两个是不一样的。

还记得上面的`async_num`中的callback函数吗？它就是在模拟一个异步操作，后面的`tornado.ioloop.IOLoop.instance().add_callback(callback, a, b)`将这个异步操作加入到IOLoop的执行队列: `_callnack`中，IOLoop的start函数从`_callbacks`中取出这个callback(async_num中的callback闭包)时，它就执行了，执行完毕它会调用`future.set_result(a+b)`

那么`future.set_result`做了啥子呢？
```python
class Future(object):
    def set_result(self, result):
        self._result = result
        self._set_done()

    def _set_done(self):
        self._done = True
        from tornado.ioloop import IOLoop
        loop = IOLoop.current()
        for cb in self._callbacks:
            loop.add_callback(cb, self)
        self._callbacks = None
```

可以看到它做的事情是:

1. 从它自身的`_callbacks`队列中取出先前加入的回调，在这里就是`lambda future: self.add_callback(callback, future)`
2. 将取出的回调加入到IOLoop的`_callbacks`回调中，等待下一轮被调用

还记得在Runner中的`self.io_loop.add_future(self.future, inner)`的调用吗，这样会导致inner函数被调用，而它又会调用`self.run()`，那么来看看`self.run()`(精简版)吧。
```python
    def run(self):
	    if self.running or self.finished:
	        return
	    try:
	        self.running = True
	        while True:
	            future = self.future
	            if not future.done():
	                return
	            self.future = None
	            try:
	                try:
	                    value = future.result()
	                future = None
	
	                if exc_info is not None:
	                    try:
	                        yielded = self.gen.throw(*exc_info)
	                    finally:
	                        exc_info = None
	                else:
	                    yielded = self.gen.send(value)
	            except (StopIteration, Return) as e:
	                self.finished = True
	                self.future = _null_future
	                future_set_result_unless_cancelled(self.result_future,
	                                                   _value_from_stopiteration(e))
	                self.result_future = None
	                self._deactivate_stack_context()
	                return
	            if not self.handle_yield(yielded):
	                return
	            yielded = None
	    finally:
	        self.running = False
```
其实核心点只有两行：
```python
	value = future.result()
	yielded = self.gen.send(value)
```

上面的代码的第一行取出异步执行的结果，第二行调用生成器的send方法将结果发送给yield让出执行的地方，在例子中造成的效果就是，`aysnc_num`中的`result = yield future`result赋值完成，此次异步操作完成

这里只分析了单次让出的情况，而且只是yield参数是Future类的实例(实际上tornado中大多数都是这种情形)的情况，其余的情况都是大同小异

那么这里有个问题，每次异步操作都要创建一个Future类的实例，并且调用`add_future`吗？当然不是，我们只需要在类中封装好就行了，只需要返回这个Future给yield抛出即可。看看异步http请求的fetch方法就知道啦
```python
    def fetch(self, request, callback=None, raise_error=True, **kwargs):
        if not isinstance(request, HTTPRequest):
            request = HTTPRequest(url=request, **kwargs)
        else:
            if kwargs:
                raise ValueError("kwargs can't be used if request is an HTTPRequest object")
        request.headers = httputil.HTTPHeaders(request.headers)
        request = _RequestProxy(request, self.defaults)
        future = TracebackFuture()
        if callback is not None:
            callback = stack_context.wrap(callback)

            def handle_future(future):
                exc = future.exception()
                if isinstance(exc, HTTPError) and exc.response is not None:
                    response = exc.response
                elif exc is not None:
                    response = HTTPResponse(
                        request, 599, error=exc,
                        request_time=time.time() - request.start_time)
                else:
                    response = future.result()
                self.io_loop.add_callback(callback, response)
            future.add_done_callback(handle_future)

        def handle_response(response):
            if raise_error and response.error:
                future.set_exception(response.error)
            else:
                future.set_result(response)
        self.fetch_impl(request, handle_response)
        return future
```

总结一下tornado是怎么达到异步执行的效果的：

0. 创建一个Future对象或类似的东西
1. 将非阻塞的异步操作加入到IOLoop的`_callbacks`队列中(可能不是立即)
2. yield一个Future或者类似Future的东西，这个Future会被`gen.coroutine`装饰器接收，并且会在将一个回调注册到Future的`_callbacks`队列中
3. 从IOLoop中的`_callbacks`的取出异步操作并执行，当异步操作完成以后，会调用`Future`的`set_result`，它从future的`_callback`队列取出之前加入的回调并执行，它最终会调用`Runner.run()`，在此函数中取出future的执行结果，然后调用生成器对象的`send`方法让生成器从yield处恢复执行
4. 如果被`@gen.coroutine`修饰的函数还有其他的yield，那么会一直在`Runner.run()`中循环，知道此次协程调用结束

那么可以这样说，tornado的协程就是:callback+生成器+闭包