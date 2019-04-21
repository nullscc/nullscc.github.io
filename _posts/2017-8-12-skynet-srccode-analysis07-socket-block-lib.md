---
layout: post
title: 'skynet源码分析07_socket阻塞库(socket.lua)'
tags: skynet
---
为什么要有socket阻塞库呢，直接引用[skynet项目关于socket的wiki](https://github.com/cloudwu/skynet/wiki/Socket):
> skynet 的 C API 采用异步读写，你可以使用 C 调用，监听一个端口，或发起一个 TCP 连接。但具体的操作结果要等待 skynet 的事件回调。skynet 会把结果以 PTYPE_SOCKET 类型的消息发送给发起请求的服务。（参考skynet_socket.h）
>
> 在处理实际业务中，这样的 API 很难使用，所以又提供了一组阻塞模式的 lua API 用于 TCP socket 的读写。它是对 C API 的封装。
>
>所谓阻塞模式，实际上是利用了 lua 的 coroutine 机制。当你调用 socket api 时，服务有可能被挂起（时间片被让给其他业务处理)，待结果通过 socket 消息返回，coroutine 将延续执行。

本文在理解[skynet源码分析06-socket处理流程](http://nulls.cc/2017/08/12/skynet_srccode_analysis06_socket_process/)的基础上配合[2016年下旬最新版skynet源码注释](https://github.com/nullscc/skynet_with_note)更佳

## API源码解析
由于socket库的本身并不复杂，直接看代码吧，这里只考虑TCP协议的。如果要看例子的话可以结合后面的master/slave与cluster模式。

### 消息处理函数
消息处理函数的注册如下:
```c
	skynet.register_protocol {
		name = "socket",
		id = skynet.PTYPE_SOCKET,	-- PTYPE_SOCKET = 6
		unpack = driver.unpack,
		dispatch = function (_, _, t, ...)
			socket_message[t](...)
		end
	}
```

其中t是从底层的 forward_message 中传递过来的，是一个枚举变量(从1-7)，也就是说调用的socket.lua的服务都会注册这样一个"socket"类型的消息处理函数。

### connect函数
```lua
	-- 初始化 buffer，创建 socket_pool 对应的 id 结构
	-- 会阻塞，然后等相应的动作完成后才能返回
	local function connect(id, func)
		local newbuffer
		if func == nil then
			newbuffer = driver.buffer()
		end
		local s = {
			id = id,
			buffer = newbuffer,		
			-- 缓冲区(此socket库的实现原理是:远端发送消息过来，会收到数据，收到数据以后将数据全部储存在此buffer中，如果需要读取，则直接从此缓冲区中读取即可)

			connected = false,
			connecting = true,
			read_required = false,
			co = false,
			callback = func,		-- 主动监听的一方如果被远端连接了，那么调用此函数(参数为 (已连接描述符 "远端ip:端口"))
			protocol = "TCP",
		}
		assert(not socket_pool[id], "socket is not closed")
		socket_pool[id] = s
		suspend(s)
			s.co = coroutine.running()
			skynet.wait(s.co)
		local err = s.connecting
		s.connecting = nil
		if s.connected then
			return id
		else
			socket_pool[id] = nil
			return nil, err
		end
	end
```
connect函数会创建一个table对应每个socket描述符，此table里面会有一个 callback，后面会提到它的作用。 suspend 函数顾名思义会挂起正在执行的协程。

### socket.open
```lua
	function socket.open(addr, port)
		local id = driver.connect(addr,port)	
		return connect(id)
	end
```

driver.connect 函数在底层会向管道发送一个 'O' 的命令，然后挂起当前的协程；管道的读端收到 'O' 后会调用 connect 函数主动与远端连接，如果连接建立成功，会收到"socket"类型的消息(forward_message)， 消息处理函数中的 t 为 SKYNET_SOCKET_TYPE_CONNECT
```lua
	socket_message[2] = function(id, _ , addr)
		local s = socket_pool[id]
		if s == nil then
			return
		end
		-- log remote addr
		s.connected = true
		wakeup(s)
	end
```

收到此消息以后，调用 wakeup 唤醒 socket.open 中的 connect，connect 会返回。
所以 socket.open 相当于阻塞模式的connect函数。

### socket.bind
```lua
	function socket.bind(os_fd)
		local id = driver.bind(os_fd)
		return connect(id)
	end
```

主要函数在于 driver.bind:
```lua
	static int lbind(lua_State *L)
		int id = skynet_socket_bind(ctx,fd);
			return socket_server_bind(SOCKET_SERVER, source, fd);
				send_request(ss, &request, 'B', sizeof(request.u.bind));
```

可见 driver.bind 会向管道发送一个 "B"，管道的读端收到后:
```lua
	static int ctrl_cmd(struct socket_server *ss, struct socket_message *result)
		case 'B':	//返回:SOCKET_ERROR、SOCKET_OPEN
			return bind_socket(ss,(struct request_bind *)buffer, result);
				struct socket *s = new_fd(ss, id, request->fd, PROTOCOL_TCP, request->opaque, true);    
				-- 这里的fd就是 driver.bind 的参数
					s->type = SOCKET_TYPE_BIND;
					return SOCKET_OPEN;

	// skynet_socket_poll 中的 socket_server_poll 会返回 SOCKET_OPEN
		case SOCKET_OPEN:	//本地打开socket连接进行监听 or 连接建立成功 or 转换网络包目的地址
			forward_message(SKYNET_SOCKET_TYPE_CONNECT, true, &result);
```

所以上层会收到"socket"类型的消息(forward_message)， 消息处理函数中的 t 为 SKYNET_SOCKET_TYPE_CONNECT，消息处理函数调用 wakeup 唤醒 socket.bind 中的 connect socket.bind函数返回。

### socket.listen
```c
	function socket.listen(host, port, backlog)
		return driver.listen(host, port, backlog)
			int id = skynet_socket_listen(ctx, host,port,backlog);
				return socket_server_listen(SOCKET_SERVER, source, host, port, backlog);
					int fd = do_listen(addr, port, backlog);
						int id = reserve_id(ss);	//为此socket描述符分配一个id给上层使用
						send_request(ss, &request, 'L', sizeof(request.u.listen));
```

还是老套路，发送'L'给本地管道，请求监听，但是这时还没有正式开始监听(因为此 socket 的type 为 SOCKET_TYPE_PLISTEN)，还要配合下面的 socket.start

### socket.start
```c
	function socket.start(id, func)
		driver.start(id)
			skynet_socket_start(ctx,id);
				socket_server_start(SOCKET_SERVER, source, id);
					send_request(ss, &request, 'S', sizeof(request.u.start));
		return connect(id, func)
	end
```

会向管道发送一个'S'，看看管道收到'S'的处理。
```c
	case 'S':	//listen与accept后都会调用'S' 返回:SOCKET_ERROR、SOCKET_OPEN
		return start_socket(ss,(struct request_start *)buffer, result);
			struct socket *s = &ss->slot[HASH_ID(id)];
			if (s->type == SOCKET_TYPE_PACCEPT || s->type == SOCKET_TYPE_PLISTEN) {
				if (sp_add(ss->event_fd, s->fd, s)) {
					force_close(ss, s, result);
					result->data = strerror(errno);
					return SOCKET_ERROR;
				}
				s->type = (s->type == SOCKET_TYPE_PACCEPT) ? SOCKET_TYPE_CONNECTED : SOCKET_TYPE_LISTEN;
				s->opaque = request->opaque;
				result->data = "start";
				return SOCKET_OPEN;
			} else if (s->type == SOCKET_TYPE_CONNECTED) {
				// todo: maybe we should send a message SOCKET_TRANSFER to s->opaque
				s->opaque = request->opaque;
				result->data = "transfer";
				return SOCKET_OPEN;
			}
```

可以看到会有两种类型的处理情况(SOCKET_TYPE_CONNECTED 一般不会跑到这里来):SOCKET_TYPE_PACCEPT 与 SOCKET_TYPE_PLISTEN 。
SOCKET_TYPE_PACCEPT : 这表示监听的一方连接已经成功建立了(三路握手)，准备好数据交互了
SOCKET_TYPE_PLISTEN : 这表示监听的一方已经调用过listen函数了，准备监听了，如果是主动监听的一方，在 socket.listen 函数之后的 socket.start 的第二个参数需要加上一个函数，当远端调用 connect 函数建立连接时，底层会传过来一个 t 为 SKYNET_SOCKET_TYPE_ACCEPT 的消息。
```lua
	-- SKYNET_SOCKET_TYPE_ACCEPT = 4
	socket_message[4] = function(id, newid, addr)
		local s = socket_pool[id]
		if s == nil then
			driver.close(newid)
			return
		end
		s.callback(newid, addr)
	end
```

这时 socket.start 的第二个参数会被调用(s.callback)，并且从底层会传上来两个参数(已连接描述符对应的id, 远端地址)给此函数使用

### socket.read
从前面的消息处理函数可以知道，从远端过来的消息会进入 socket_message[1] 函数进行处理
```lua
	socket_message[1] = function(id, size, data)
		local s = socket_pool[id]
		local sz = driver.push(s.buffer, buffer_pool, data, size)
		local rr = s.read_required
			local rr = s.read_required
			local rrt = type(rr)
			if rrt == "number" then
				-- read size
				if sz >= rr then
					s.read_required = nil
					wakeup(s)
				end
			else
				if s.buffer_limit and sz > s.buffer_limit then
					skynet.error(string.format("socket buffer overflow: fd=%d size=%d", id , sz))
					driver.clear(s.buffer,buffer_pool)
					driver.close(id)
					return
				end
				if rrt == "string" then
					-- read line
					if driver.readline(s.buffer,nil,rr) then
						s.read_required = nil
						wakeup(s)
					end
				end
			end
```

核心的在于 driver.push，他的作用在于将接受到的数据放在buffer里面，再看看 socket.read 函数(只考虑一般情况)
```lua
	function socket.read(id, sz)
		local ret = driver.pop(s.buffer, buffer_pool, sz)
		if ret then
			return ret
		end
```

一般情况下，会调用 driver.pop 从 buffer里面取出数据交付给API的调用者。
到这里就很明白了，socket库的核心在于将收到的数据储存在自己维护的缓存里面，当API调用者想要数据时，调用 socket.read、socket.readall、socket.readline 函数从buffer缓存里面取出数据。
而阻塞也只不过是利用底层的C API的非阻塞接口加上上层的协层让出机制来达成。

### socket.block
```lua
	function socket.block(id)
		local s = socket_pool[id]
		if not s or not s.connected then
			return false
		end
		assert(not s.read_required)
		s.read_required = 0
		suspend(s)
		return s.connected
	end
```

从`s.read_required = 0`可以看出，此API的作用在于等待远端有任意数据过来。

### socket.abandon
```lua
	-- abandon use to forward socket id to other service
	-- you must call socket.start(id) later in other service
	function socket.abandon(id)
		local s = socket_pool[id]
		if s and s.buffer then
			driver.clear(s.buffer,buffer_pool)
		end
		socket_pool[id] = nil
	end
```

socket.abandon 仅仅是将连接池中对应此连接的table置为nil，这意味着:socket库再也接受不到任何数据了，所以调用者需要尽快的在别的服务调用 socket.start

### socket.close
```lua
	-- 关闭 socket 连接
	function socket.close(id)
		local s = socket_pool[id]
		if s == nil then
			return
		end
		if s.connected then
			driver.close(id)
				skynet_socket_close(ctx, id);
					socket_server_close(SOCKET_SERVER, source, id);
						send_request(ss, &request, 'K', sizeof(request.u.close));
							case 'K':	//返回:SOCKET_CLOSE、
								return close_socket(ss,(struct request_close *)buffer, result);
									force_close(ss,s,result);
										if (s->type != SOCKET_TYPE_PACCEPT && s->type != SOCKET_TYPE_PLISTEN) {
											sp_del(ss->event_fd, s->fd);
										}
										if (s->type != SOCKET_TYPE_BIND) {
											if (close(s->fd) < 0) {
			-- notice: call socket.close in __gc should be carefully,
			-- because skynet.wait never return in __gc, so driver.clear may not be called
			if s.co then
				-- reading this socket on another coroutine, so don't shutdown (clear the buffer) immediately
				-- wait reading coroutine read the buffer.
				assert(not s.closing)
				s.closing = coroutine.running()
				skynet.wait(s.closing)
			else
				suspend(s)
			end
			s.connected = false
		end
		close_fd(id)	-- clear the buffer (already close fd)
```

可见 socket.close 会"异步"调用 close 函数关闭连接，当然还是有可能会被阻塞的。

### socket.close_fd
```lua
	function socket.close_fd(id)
		assert(socket_pool[id] == nil,"Use socket.close instead")
		driver.close(id)
	end

socket.close_fd 是不管socket描述符现在情况如何，强制关闭一个连接(极其罕见的情况会使用，避免 socket.close 是原因之一)。
```

### socket.shutdown
```lua
	function socket.shutdown(id)
		close_fd(id, driver.shutdown)
			local s = socket_pool[id]
			if s then
				if s.buffer then
					driver.clear(s.buffer,buffer_pool)
				end
				if s.connected then
					func(id)
				end
			end
	end
```

清除一个连接的缓存数据，再"强行"关闭这个连接(一般不建议使用这个 API ，但如果你需要在 __gc 元方法中关闭连接的话，shutdown 是一个比 close 更好的选择（因为在 gc 过程中无法切换 coroutine）。)