---
layout: post
title: 'skynet源码分析09_cluster与socketchannel'
tags: skynet
---
cluster就是集群的意思了，和master/slave模式不同的是，它支持断线重连，但是不能在同一条连接上既主动请求又能主动推送消息。

由于cluster和socketchannel分不开，并且skynet框架本身用到socketchannel的地方不多，我又暂时不想去研究mysql、redis库，所以就把它们放在一起了。

本文在理解[ skynet源码分析07_socket阻塞库(socket.lua)](http://nulls.cc/2017/08/12/skynet_srccode_analysis07_socket_block_lib/)的基础上配合[2016年下旬最新版skynet源码注释](https://github.com/nullscc/skynet_with_note)更佳

## 先看看clusterd服务的创建与初始化
创建clusterd服务:
```lua
	skynet.init(function()
		clusterd = skynet.uniqueservice("clusterd")
	end)
```

调用 skynet.init 以便在"适当"的调用"clusterd = skynet.uniqueservice("clusterd")"时候创建 clusterd 服务

clusterd服务的初始化:
```lua
	local config_name = skynet.getenv "cluster"

	skynet.start(function()
		loadconfig()
		local function loadconfig()
			local f = assert(io.open(config_name))
			local source = f:read "*a"
			f:close()
			local tmp = {}
			assert(load(source, "@"..config_name, "t", tmp))()
			for name,address in pairs(tmp) do
				assert(type(address) == "string")
				if node_address[name] ~= address then
					-- address changed
					if rawget(node_channel, name) then
						node_channel[name] = nil	-- reset connection
					end
					node_address[name] = address
				end
			end
		end
		skynet.dispatch("lua", function(session , source, cmd, ...)
			local f = assert(command[cmd])
			f(source, ...)
		end)
	end)
```

先调用loadconfig将配置文件的cluster读取出来，配置文件的cluster是一个文件路径，里面存放的是各一条或多条cluster节点的配置，通过node_channel与node_address将它们储存起来。

## 再从最上层的API分析
结合[cluster的wiki](https://github.com/cloudwu/skynet/wiki/Cluster)从 cluster.open、cluster.register两个 API 入手。
### cluster.open
```lua
	function cluster.open(port)
		if type(port) == "string" then
			skynet.call(clusterd, "lua", "listen", port)
		else
			skynet.call(clusterd, "lua", "listen", "0.0.0.0", port)
		end
end
```

看看 clusterd 收到 listen 消息时怎么处理的
```lua
	function command.listen(source, addr, port)
		local gate = skynet.newservice("gate")
		if port == nil then
			addr, port = string.match(node_address[addr], "([^:]+):(.*)$")
		end
		skynet.call(gate, "lua", "open", { address = addr, port = port })
		skynet.ret(skynet.pack(nil))
	end
```

通过向gate发送一个"open"消息来监听这个端口。

### cluster.register
当一个cluster节点调用 cluster.open 监听一个端口以后，另一个cluster可以调用 cluster.call 连向监听一方的 cluster 发送消息了。但是监听一方的cluster节点的服务地址怎么得到是一个问题，所以一般会先调用 cluster.register 注册一个字符串地址。
```lua
	function command.register(source, name, addr)
		assert(register_name[name] == nil)
		addr = addr or source
		local old_name = register_name[addr]
		if old_name then
			register_name[old_name] = nil
		end
		register_name[addr] = name
		register_name[name] = addr
		skynet.ret(nil)
		skynet.error(string.format("Register [%s] :%08x", name, addr))
	end
```

在cluster节点中注册一个字符串地址很简单， 将节点的数字地址存在register_name即完成了。

## A向B请求然后得到B返回的过程
分析下A向B请求然后得到B返回的过程

### A节点调用cluster.call发送消息给B节点
```lua
	function cluster.call(node, address, ...)
		-- skynet.pack(...) will free by cluster.core.packrequest
		return skynet.call(clusterd, "lua", "req", node, address, skynet.pack(...))
			function command.req(...)
				local ok, msg, sz = pcall(send_request, ...)
				if ok then
					skynet.ret(xxx)
	end
```

可见会调用 send_request 来向另外一个节点发消息的，那么是怎么做到的呢？
```lua
	local function send_request(source, node, addr, msg, sz)
		local session = node_session[node] or 1
		-- msg is a local pointer, cluster.packrequest will free it
		-- request 为前面10个字节[(sz+9)、0(0x80, 0x81)、addr(int32)、session(int32)]加上msg组成的字符串
		-- session 会自增1
		local request, new_session, padding = cluster.packrequest(addr, session, msg, sz)
		node_session[node] = new_session

		-- node_channel[node] may yield or throw error
		local c = node_channel[node]

		return c:request(request, session, padding)
	end
```

底层的接口就不看了，无非就是打包、解包。当首次发起 cluster.call 时， node_channel[node]为nil，但是 node_channel 有元表:`local node_channel = setmetatable({}, { __index = open_channel })`，所以会在 open_channel 中"找到"这个值。
```lua
	local function open_channel(t, key)
		local host, port = string.match(node_address[key], "([^:]+):(.*)$")
		local c = sc.channel {
			host = host,
			port = tonumber(port),
			response = read_response,
			nodelay = true,
		}
		assert(c:connect(true))
		t[key] = c
		return c
	end
```

socket_channel.channel的看着很简单，这里贴上注释过后的代码先过一遍:
```lua
	function socket_channel.channel(desc)
		local c = {
			__host = assert(desc.host),	-- ip地址
			__port = assert(desc.port),	-- 端口
			__backup = desc.backup,		-- 备用地址(成员需有一个或多个{host=xxx, port=xxx})
			__auth = desc.auth,			-- 认证函数
			__response = desc.response,	-- It's for session mode 如果是session模式，则需要提供此函数
			__request = {},	-- request seq { response func or session }	-- It's for order mode -- 消息处理函数，成员为函数
			__thread = {}, -- coroutine seq or session->coroutine map	-- 存储等待回应的协程
			__result = {}, -- response result { coroutine -> result }	-- 存储返回的结果，以便唤醒的时候能从对应的协程里面拿出
			__result_data = {},											-- 存储返回的结果，以便唤醒的时候能从对应的协程里面拿出
			__connecting = {},	-- 用于储存等待完成的队列，队列的成员为协程:co
			__sock = false,		-- 连接成功以后是一个 table，第一次元素为 fd，元表为 channel_socket_meta
			__closed = false,	-- 是否已经关闭
			__authcoroutine = false,	-- 如果存在 __auth,那么这里存储的是认证过程的协程
			__nodelay = desc.nodelay,	-- 配置是否启用 TCP 的 Nagle 算法
			-- __dispatch_thread 消息处理函数的协程co
			-- __connecting_thread 等待连接完成的协程
		}

		return setmetatable(c, channel_meta)
	end
```

然后来看看 c:connect,由于 c 中没有 connect ，去它的元table:channel_meta 中找， channel_meta 中也没 connect ，去 channel_meta 的元table:channel 中找，这下能找到了:
```lua
	function channel:connect(once)
		if self.__closed then
			if self.__dispatch_thread then
				-- closing, wait
				assert(self.__connecting_thread == nil, "already connecting")
				local co = coroutine.running()
				self.__connecting_thread = co
				skynet.wait(co)
				self.__connecting_thread = nil
			end
			self.__closed = false
		end

		return block_connect(self, once)
	end
```

由于是首次执行 channel:connect ，所以直接执行 block_connect
```lua
	local function block_connect(self, once)
		local r = check_connection(self)
		if r ~= nil then
			return r
		end
		local err

		-- 如果正在等待连接完成的队列大于0，则将当前协程加入队列
		if #self.__connecting > 0 then
			-- connecting in other coroutine
			local co = coroutine.running()
			table.insert(self.__connecting, co)
			skynet.wait(co)
		else	-- 尝试连接，如果连接成功，依次唤醒等待连接完成的队列
			self.__connecting[1] = true
			err = try_connect(self, once)
			self.__connecting[1] = nil
			for i=2, #self.__connecting do
				local co = self.__connecting[i]
				self.__connecting[i] = nil
				skynet.wakeup(co)
			end
		end

		r = check_connection(self)
		if r == nil then
			skynet.error(string.format("Connect to %s:%d failed (%s)", self.__host, self.__port, err))
			error(socket_error)
		else
			return r
		end
	end
```

是首次执行:直接跑到 else 分支，调用 try_connect 函数,然后调用 skynet.wakeup(co) 挂起当前的协程。 try_connect 应该就是重点了。
```lua
	-- 尝试连接，如果没有明确指定只连接一次，那么一直尝试重连
	local function try_connect(self , once)
		local t = 0
		while not self.__closed do
			local ok, err = connect_once(self)
			if ok then
				if not once then
					skynet.error("socket: connect to", self.__host, self.__port)
				end
				return
			elseif once then
				return err
			else
				skynet.error("socket: connect", err)
			end
			if t > 1000 then	-- 如果 once 不为真，则一直尝试连接
				skynet.error("socket: try to reconnect", self.__host, self.__port)
				skynet.sleep(t)
				t = 0
			else
				skynet.sleep(t)
			end
			t = t + 100
		end
	end
```

由于最初初始化的时候 self.__closed 为false，所以这里while成立。执行 connect_once(去掉"干扰代码")，如果连接失败，从此函数后面的代码可以看出如果 once 不为真，就会一直尝试连接。
```lua
	local function connect_once(self)
		local fd,err = socket.open(self.__host, self.__port)
		if not fd then	-- 如果连接不成功，连接备用的地址
			fd = connect_backup(self)
		end
		if self.__nodelay then
			socketdriver.nodelay(fd)
		end

		self.__sock = setmetatable( {fd} , channel_socket_meta )
		self.__dispatch_thread = skynet.fork(dispatch_function(self), self)

		if self.__auth then
			self.__authcoroutine = coroutine.running()
			local ok , message = pcall(self.__auth, self)
			if not ok then
				close_channel_socket(self)
				if message ~= socket_error then
					self.__authcoroutine = false
					skynet.error("socket: auth failed", message)
				end
			end
			self.__authcoroutine = false
			if ok and not self.__sock then
				-- auth may change host, so connect again
				return connect_once(self)
			end
			return ok
		end

		return true
	end
```

从上面可以看出， connect_once函数的工作为:

1. 调用 socket.open 主动连接远端cluster节点，如果连接不成功，尝试连接备用地址
2. 如果 __nodelay 为 true，设置不使用Nagle算法
3. 设置 __sock = {fd} 并且元表为 channel_socket_meta
4. 调用skynet.fork开辟一个消息处理协程，并设置 __dispatch_thread 处理协程为skynet.fork的返回值
5. 如果__auth存在，则执行认证过程

如果连接成功的话，那么 c:connect 函数会正常返回。 所以 send_request 函数中 node_channel[node] 也正常返回了。接下来执行 `return c:request(request, session, padding)`
```lua
	function channel:request(request, response, padding)
		assert(block_connect(self, true))	-- connect once
		local fd = self.__sock[1]

		if padding then
			-- padding may be a table, to support multi part request
			-- multi part request use low priority socket write
			-- socket_lwrite returns nothing
			socket_lwrite(fd , request)
			for _,v in ipairs(padding) do
				socket_lwrite(fd, v)
			end
		else
			if not socket_write(fd , request) then
				close_channel_socket(self)
				wakeup_all(self)
				error(socket_error)
			end
		end

		if response == nil then
			-- no response
			return
		end

		return wait_for_response(self, response)
	end
```

channel:request 的一般工作流程为:

1. 调用一次 block_connect，从前面可以看到 block_connect 函数的功用在于:如果没有连接对端cluster节点，那么连接；如果已经连接，直接返回
2. 调用`socket_write(fd , request)` 将消息包发出去
3. 调用 wait_for_response 等待消息返回

由此可知，调用 channel:request 将消息发到对端 cluster 节点，调用 wait_for_response 等待消息返回(从字面意思都能看出)，看看 wait_for_response 的实现(主要代码)。
```lua
	local function wait_for_response(self, response)
		local co = coroutine.running()
		push_response(self, response, co)
		skynet.wait(co)

		local result = self.__result[co]
		self.__result[co] = nil
		local result_data = self.__result_data[co]
		self.__result_data[co] = nil

		return result_data
	end
```

调用 push_response 将 response 加入到队列__request(以当前协程为key)，这里的response就是前面提到的 open_channel 中的 read_response，然后让出协程，此次消息发送结束。

### B节点收到A节点的消息请求
由于B节点是监听的一方，前面提到过，是通过向 gate 服务发送一个 "open" 消息完成监听的。那么当B节点收到A节点的请求时，gate服务会先收到这个数据，然后将数据转发给B节点的clusterd服务:`skynet.send(watchdog, "lua", "socket", "open", fd, addr)`。clusterd收到从gate服务转发过来的socket消息的处理函数为:command.socket
```lua
	function command.socket(source, subcmd, fd, msg)
		if subcmd == "data" then
			local sz
			local addr, session, msg, padding = cluster.unpackrequest(msg)
			if padding then
				local req = large_request[session] or { addr = addr }
				large_request[session] = req
				table.insert(req, msg)
				return
			else
				local req = large_request[session]
				if req then
					large_request[session] = nil
					table.insert(req, msg)
					msg,sz = cluster.concat(req)
					addr = req.addr
				end
				if not msg then
					local response = cluster.packresponse(session, false, "Invalid large req")
					socket.write(fd, response)
					return
				end
			end
			local ok, response
			if addr == 0 then       -- 如果为 0 代表是查询地址
				local name = skynet.unpack(msg, sz)
				local addr = register_name[name]
				if addr then
					ok = true
					msg, sz = skynet.pack(addr)
				else
					ok = false
					msg = "name not found"
				end
			else
				ok , msg, sz = pcall(skynet.rawcall, addr, "lua", msg, sz)
			end
			if ok then
				response = cluster.packresponse(session, true, msg, sz)
				if type(response) == "table" then
					for _, v in ipairs(response) do
						socket.lwrite(fd, v)
					end
				else
					socket.write(fd, response)
				end
			else
				response = cluster.packresponse(session, false, msg)
				socket.write(fd, response)
			end
		elseif subcmd == "open" then
			skynet.error(string.format("socket accept from %s", msg))
			skynet.call(source, "lua", "accept", fd)
```
当收到 "lua" "open" 时，向gate服务发送一个 "accept" 函数完成三路握手。

当收到 "lua" "data" 时，代表有请求过来了，处理请求(只考虑简单情况):

* 调用 cluster.unpackrequest(msg) 将网络包解析出来(对应cluster.packrequest)
* 如果是查询字符串地址(后面会说到)，从本地的 register_name 中取出数字地址，再调用 socket.write(fd, response)
* 如果是A节点发送一个请求给B节点中的服务的请求，那么调用`pcall(skynet.rawcall, addr, "lua", msg, sz)`向B节点本身的服务请求数据，返回的数据调用 socket.write 发送出去

从上面代码可以看出，被请求的一方总是会有返回值返回给远端的。

### B节点收到A节点的消息返回
从前面的分析可以知道，B是主动调用 connect 函数进行连接的一方，它是通过 socketchannel来实现连接的，而socketchannel又会通过 socket.lua 来建立连接，所以返回的消息会先发送到 socket.lua 中，所以必须要调用 socket.read 读取系列函数中的一个来接收数据。
从前面可以知道B节点注册的消息处理函数为:dispatch_by_session
```lua
	local function dispatch_by_session(self)
		local response = self.__response
		while self.__sock do
			local ok , session, result_ok, result_data, padding = pcall(response, self.__sock)
```

其中`pcall(response, self.__sock)`中response函数是最初提供的，见 open_channel 函数中的 `response = read_response`， read_response 函数的实现:
```lua
	local function read_response(sock)
		local sz = socket.header(sock:read(2))	-- sock:read(2)为读取前两个字节 socket.header(sock:read(2))为得到前两个字节表示的 number 类型的值
		local msg = sock:read(sz)
		return cluster.unpackresponse(msg)	-- session, ok, data, padding
	end
```

可见read_response函数会返回解包后的数据。回到 dispatch_by_session 函数(一般流程的主要代码):
```lua
	local function dispatch_by_session(self)
		local response = self.__response
		-- response() return session
		while self.__sock do
			local ok , session, result_ok, result_data, padding = pcall(response, self.__sock)
			if ok and session then
				local co = self.__thread[session]
				if co then
					self.__thread[session] = nil
					self.__result[co] = result_ok
					if result_ok and self.__result_data[co] then
						table.insert(self.__result_data[co], result_data)
					else
						self.__result_data[co] = result_data
					end
					skynet.wakeup(co)
				end
	end
```

结合 read_response函数 可见 dispatch_by_session 的工作为:
	1. 调用 read_response 将远端cluster节点发送过来的数据读取出来
	2. 然后将其返回值放在 __result 与 __result_data 然后调用 skynet.wakeup 唤醒 cluster.call 中的 wait_for_response 函数
	3. wait_for_response 函数将结果从 __result 与 __result_data 中取出然后返回。

至此一次跨cluster节点的请求与返回流程结束。

## cluster.query
cluster.query函数是用来查询远程节点的字符串地址
```lua
	function cluster.query(node, name)
		-- 注意第5个参数为0
		return skynet.call(clusterd, "lua", "req", node, 0, skynet.pack(name))
```

和发送数据请求差不多，它会请求 clusterd 服务发送一个请求消息给对端的cluster节点，对端cluster节点收到后会返回给它。

## 代理服务的实现
代理服务是通过cluster.proxy函数来实现的。
### skynet.forward_type
在看cluster.proxy之前需要先看看看 skynet.forward_type 的实现:
```lua
	function skynet.forward_type(map, start_func)
		c.callback(function(ptype, msg, sz, ...)
			local prototype = map[ptype]
			if prototype then
				dispatch_message(prototype, msg, sz, ...)
			else
				dispatch_message(ptype, msg, sz, ...)
				c.trash(msg, sz)
			end
		end, true)
		skynet.timeout(0, function()
			skynet.init_service(start_func)
		end)
	end
```

skynet.forward_type 的工作为:

1. c.callback 为服务注册一个消息处理函数
2. skynet.forward_type 的第一个参数是需要转换消息类型的表，如果在表中存在的消息类型，就会转换成另外一个类型
3. 调用 dispatch_message 处理收到的消息，由于是代理的，所以c.callback的第二个参数为 true，代表处理完成后不释放消息的内存(因为还要转发到另外的服务当中去)
4. 调用 skynet.init_service(start_func) 来调用 skynet.forward_type 的第二个函数

### cluster.proxy
cluster.proxy可以生成一个本地代理服务(发送到此服务的消息都会被转发到远端的cluster节点)
```lua
	function cluster.proxy(node, name)
		return skynet.call(clusterd, "lua", "proxy", node, name)
			local fullname = node .. "." .. name
			proxy[fullname] = skynet.newservice("clusterproxy", node, name)
			skynet.ret(skynet.pack(proxy[fullname]))
	end
```

调用 cluster.proxy 会创建一个 clusterproxy 服务， 并得到一个本地服务的地址，看看 clusterproxy 服务的实现:
```lua
	local forward_map = {
		[skynet.PTYPE_SNAX] = skynet.PTYPE_SYSTEM,
		[skynet.PTYPE_LUA] = skynet.PTYPE_SYSTEM,
		[skynet.PTYPE_RESPONSE] = skynet.PTYPE_RESPONSE,	-- don't free response message
	}

	skynet.forward_type( forward_map ,function()
		local clusterd = skynet.uniqueservice("clusterd")
		local n = tonumber(address)
		if n then
			address = n
		end
		skynet.dispatch("system", function (session, source, msg, sz)
			skynet.ret(skynet.rawcall(clusterd, "lua", skynet.pack("req", node, address, msg, sz)))
		end)
	end)
```

由于 PTYPE_SNAX、PTYPE_LUA类型的消息被转换成了 PTYPE_SYSTEM 类型，所以只要是这两类的消息都会被转发成 PTYPE_SYSTEM 类型的消息，而 PTYPE_SYSTEM 类型的消息处理函数工作为:向clusterd发送一个"req"类型的消息，这样消息能发到对端的cluster节点。
当对端cluster节点收到这个请求返回，那么 PTYPE_SYSTEM 类型的消息处理函数也返回，这样会返回到代理服务中。

## 简单说说socketchannel的两个模式
直接借用[skynet SocketChannel的wiki](https://github.com/cloudwu/skynet/wiki/SocketChannel)来介绍这两种模式:
>1. 每个请求包对应一个回应包，由 TCP 协议保证时序。redis 的协议就是一个典型。每个 redis 请求都必须有一个回应，但不必收到回应才可以发送下一个请求。
>2. 发起每个请求时带一个唯一 session 标识，在发送回应时，带上这个标识。这样设计可以不要求每个请求都一定要有回应，且不必遵循先提出的请求先回应的时序。MongoDB 的通讯协议就是这样设计的。

所以在socketchannel中对应的第一种模式为 order模式，相应的消息处理函数为:dispatch_by_order

第二种模式为 session模式，相应的消息处理函数为:dispatch_by_session

这里 order模式就不拿代码详细分析了，和 session模式差不了太多，结合wiki和cluster的分析很容易得出两者的区别

## 小结
1. socketchannel有两种模式，分为: order模式与cluster模式
2. socketchannel支持断线重连，主要是依赖每次发包如果发现断线了就尝试重连，而且两个cluster节点连接过程是在第一个消息请求时(即如果没有消息要发送，那么两个节点就不会连接)。
3. cluster模式的两个节点主动监听的一方不能主动发起请求，如果A B两个cluster节点想要实现A能向B请求而且B也能向A请求，需要建立两条连接
4. A节点向B节点请求消息，B节点总会有返回给A的