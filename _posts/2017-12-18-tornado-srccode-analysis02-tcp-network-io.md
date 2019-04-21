---
layout: post
title: 'tornado源码浅析2-TCP网络IO处理'
tags: python
---
看一个框架我喜欢先从网络方面分析，然后再来分析它是怎么达到并发/并行的效果的，所以这里先看看tornado的网络处理

为了清晰，这里就先从tornado官方的demo入手分析吧。在`demos/tcpecho`文件夹下面有三个文件，我们看其中的`client.py`和`server.py`

这里还是把server的完整代码贴出来吧(为了研究的纯粹性client就先不看了，毕竟大家都只知道echo是个什么东西了):
```python
    import logging
    from tornado.ioloop import IOLoop
    from tornado import gen
    from tornado.iostream import StreamClosedError
    from tornado.tcpserver import TCPServer
    from tornado.options import options, define
    
    define("port", default=9888, help="TCP port to listen on")
    logger = logging.getLogger(__name__)
    
    class EchoServer(TCPServer):
        @gen.coroutine
        def handle_stream(self, stream, address):
            while True:
                try:
                    data = yield stream.read_until(b"\n")
                    logger.info("Received bytes: %s", data)
                    if not data.endswith(b"\n"):
                        data = data + b"\n"
                    yield stream.write(data)
                except StreamClosedError:
                    logger.warning("Lost client at host %s", address[0])
                    break
                except Exception as e:
                    print(e)
    
    
    if __name__ == "__main__":
        options.parse_command_line()
        server = EchoServer()
        server.listen(options.port)
        logger.info("Listening on TCP port %d", options.port)
        IOLoop.current().start()
```
从上面我们可以看到自定义了一个`EchoServer`类，它继承自`TCPServer`，并且重写了TCPServer的`handle_stream`方法，所以我们知道我们需要分析的是TCPServer类，下面的IOLoop暂时不用理，先记住yield是让出执行，而不是生产数据就够了。

我们看看`TCPServer`的listen方法：
```python
    def listen(self, port, address=""):
        sockets = bind_sockets(port, address=address)
            
            def bind_sockets(port, address=None, family=socket.AF_UNSPEC, ...
                sock = socket.socket(af, socktype, proto) # 创建socket描述符
                sock.setblocking(0)                       # 设置非阻塞
                sock.bind(sockaddr)
                bound_port = sock.getsockname()[1]
                sock.listen(backlog)
                sockets.append(sock)
            return sockets

        self.add_sockets(sockets)

            for sock in sockets:
               self._sockets[sock.fileno()] = sock
               self._handlers[sock.fileno()] = add_accept_handler(sock, self._handle_connection)
```                     
ok，这里我们看到创建监听描述符与将它设置为非阻塞的动作，那么并且调用后面又调用了`add_accept_handler`方法，虽然没贴出来，但是看名字也知道它的作用啦：当有描述符完成三路握手了就调用`self._handle_connection`，那么，是怎么做到的呢？来看看`add_accept_handler`的实现
```python
    def add_accept_handler(sock, callback):
        io_loop = IOLoop.current()
            removed = [False]
            def accept_handler(fd, events):
                for i in xrange(_DEFAULT_BACKLOG):
                    if removed[0]:
                        return
                    try:
                        connection, address = sock.accept()
                    ...
                    callback(connection, address)
            def remove_handler():
                io_loop.remove_handler(sock)
                removed[0] = True
            io_loop.add_handler(sock, accept_handler, IOLoop.READ)
            return remove_handler
```
这里的`io_loop.add_handler`的作用很明显：当描述符上发生了`IOLoop.READ`事件，调用`accept_handler`，具体怎么实现的另开文章说，这里先记住～

这里我们知道了当有连接(监听描述符)完成了三路握手，就会调用`accept_handler`，而它的动很简单：

1. 创建已完成描述符
2. 调用callback，callback就是`_handle_connection` 

好，至此我们完成了tcp的三路握手，并且创建了已连接描述符，来看看是怎么通过这个已连接描述符传输网络数据给框架使用者使用的吧。
```python
    # 不考虑ssl情况
    def _handle_connection(self, connection, address):
            stream = IOStream(connection, max_buffer_size=self.max_buffer_size, read_chunk_size=self.read_chunk_size)
            future = self.handle_stream(stream, address)
            if future is not None:
                self.io_loop.add_future(gen.convert_yielded(future), lambda f: f.result())
```
可见我们这里的主要执行流程是`self.handle_stream`，最后一句暂时不用管，那个匿名函数最后(handle_stream返回后)才会执行，当然这里的`_handle_connection`是立即就执行完了。

虽然`_handle_connection`立马返回了，但是里面的流程(handle_stream)还是在继续执行，只不过当数据网络数据还没准备好它就暂停不执行而已。好，这里到重点了，看看`handle_stream`的处理吧。(当然是`demos/server.py`下的`handle_stream啦`)
```python
    @gen.coroutine
    def handle_stream(self, stream, address):
        while True:
            try:
                data = yield stream.read_until(b"\n")
                logger.info("Received bytes: %s", data)
                if not data.endswith(b"\n"):
                    data = data + b"\n"
                yield stream.write(data)
            except StreamClosedError:
                logger.warning("Lost client at host %s", address[0])
                break
            except Exception as e:
                print(e)
```

这里再强调一遍，由于它被`gen.coroutine`装饰了，那么它就是协程语义而不是生成器语义，遇到yield代表让出执行，而不是生产数据。(记住先，别想一口吃个大胖子)

比如这句代码`data = yield
stream.read_until(b"\n")`的意思是等待有数据从`steam.read_until`中产生，如果有数据了，那么将数据返回给赋给data，如果没有数据，那么执行流程就停在这里。

看看`steam.read_until`的处理吧。这里的stream是类`IOStream`的一个实例
```python
    def read_until(self, delimiter, callback=None, max_bytes=None):
        future = self._set_read_callback(callback)

            self._read_future = Future()
            return self._read_future

        self._read_delimiter = delimiter
        self._read_max_bytes = max_bytes
        try:
            self._try_inline_read()
                def _try_inline_read(self):
                   self._run_streaming_callback()
                   pos = self._find_read_pos()
                   if pos is not None:
                       self._read_from_buffer(pos)
                       return
                   self._check_closed()
                   try:
                       pos = self._read_to_buffer_loop()
                   except Exception:
                       self._maybe_run_close_callback()
                       raise
                   if pos is not None:
                       self._read_from_buffer(pos)
                       return
                   if self.closed():
                       self._maybe_run_close_callback()
                   else:
                       self._add_io_state(ioloop.IOLoop.READ)
        
               except UnsatisfiableReadError as e:
            ...
            return future
        except:
            ...
        return future
```

这里几个比较重要的东东是：

* `self._set_read_callback`: 它的作用是得到一个`Future`对象，当数据准备好有调用它的`set_result`达到异步执行的效果`
* `_run_streaming_callback` : 它的作用是先看看tcp的缓冲区有没有数据，并尝试读到buffer中，并将 buffer 中拿到数据给框架使用者使用
* `_read_to_buffer_loop` : 它的作用是从socket描述符中读取数据到 buffer 
* `self._read_delimiter` :
  由于tcp是流式套接字，所以读取数据时可能需要一个特定的分隔符(web类应用)，这个就代表这个分隔符，如果读到了这个分隔符，那么`self._read_to_buffer_loop()`，会返回一个非None值，这样就会执行`_read_from_buffer`
* `self._read_from_buffer(pos)` : 它的作用是满足条件了调用`future.set_result`完成此次读取的异步调用

代码比较简单懂，这里总结下`stream.read_until`的流程:

1. 调用yield让出执行
2. 框架底层的轮询机制发现有数据从socket描述符过来
3. 读取数据到buffer，如果数据未满足了分隔符的条件(read_until的参数)，那么继续等待下次数据的到来
4. 如果读取的到的数据满足分隔符的条件了，那么调用`Future.set_result`恢复yield的让出执行，即数据返回到框架使用者的代码那里，即`data = yield stream.read_until(b"\n")`语句中的data赋值完成，继续执行下一步

下面的写操作也是差不多的，这里不再赘述。

这里只是把tcp的三路握手即数据传输过程说了一遍，其中涉及到的异步处理还米有说，因为异步处理也算是比较复杂的过程，一起分析可能会晕。所以单独开一骗出来说说IOLoop与coroutine