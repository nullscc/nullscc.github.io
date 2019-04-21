---
layout: post
title: 'skynet源码分析03-节点内服务间消息的传递'
tags: skynet
---
理解节点内服务间消息的传递是很重要的，这样对处理一些奇怪的问题比较有帮助。

本文配合[2016年下旬最新版skynet源码注释](https://github.com/nullscc/skynet_with_note)更佳

## 消息传递:
消息的传递是依靠两个消息队列并且配合工作线程来进行的
* 服务的:struct message_queue
* 全局的:struct global_queue

## 工作线程在干嘛？
工作线程的作用是从全局的消息队列中取出单个服务的消息队列，再从服务的消息队列中取出消息，然后调用消息的回调函数来处理收到的消息
```lua
		static void *thread_worker(void *p)
		q = skynet_context_message_dispatch(sm, q, weight);
			uint32_t handle = skynet_mq_handle(q);                        // 得到服务的地址
			struct skynet_context * ctx = skynet_handle_grab(handle);     // 得到服务的结构体
			int overload = skynet_mq_overload(q);                         // 判断是否过载了
			if (overload) {
			skynet_error(ctx, "May overload, message queue length = %d", overload);
			}
			skynet_monitor_trigger(sm, msg.source , handle);              // 将此条处理给 moniter 线程监控
			if (ctx->cb == NULL) {
			skynet_free(msg.data);
			} else {
			dispatch_message(ctx, &msg);                     
						if (!ctx->cb(ctx, ctx->cb_ud, type, msg->session, msg->source, msg->data, sz))      
							// 调用服务对应的消息处理函数
			}
			skynet_monitor_trigger(sm, 0,0);
		}
```

## ctx->cb从哪里来？

即处理消息的回调函数从哪里来？
### 对于C服务来说:
```lua
	skynet_context_new
			skynet_module_instance_init
					xxx_init
							skynet_callback(ctx, inst, xxx/*cb*/);
									context->cb = cb;
									context->cb_ud = ud;
```

### 对于lua服务：
每个lua服务都会调用skynet.start函数
```lua
	function skynet.start(start_func)
				c.callback(skynet.dispatch_message)
						static int lcallback(lua_State *L) { //主要用来注册lua服务的消息处理函数
							skynet_callback(context, gL, _cb);
									void skynet_callback(struct skynet_context * context, void *ud, skynet_cb cb)
												context->cb = cb;
												context->cb_ud = ud;
```

从上面代码来看，不管是lua服务还是C服务都会调用 skynet_callback 注册消息处理函数

## 消息的发送:

### 对lua服务来说
lua服务的消息发送一般是通过调用skynet.send(其它类似函数也是一样的流程)
```lua
	//系统会首先skynet.register_protocol --skynet.lua
	function skynet.register_protocol(class)
		proto[class.name] = class
		proto[class.id] = class

	----- register protocol 默认的三种类型 注册消息处理函数
	do
		local REG = skynet.register_protocol
		REG {
			name = "lua",
			id = skynet.PTYPE_LUA,
			pack = skynet.pack,
			unpack = skynet.unpack,
		}
		REG {
			name = "response",
			id = skynet.PTYPE_RESPONSE,
		}
		REG {
			name = "error",
			id = skynet.PTYPE_ERROR,
			unpack = function(...) return ... end,
			dispatch = _error_dispatch,
		}
	end

	//然后调用skynet.send发送消息
	function skynet.send(addr, typename, ...)
		local p = proto[typename]
		return c.send(addr, p.id, 0 , p.pack(...)) --lua-skynet.c
			{ "send" , lsend },
				static int lsend(lua_State *L) {
					struct skynet_context * context = lua_touserdata(L, lua_upvalueindex(1)); //得到源服务的结构体
					uint32_t dest = (uint32_t)lua_tointeger(L, 1); //得到目标服务的地址，如果不是整数就返回0
					int type = luaL_checkinteger(L, 2); //得到消息类型
					int session = 0;  //得到会话id
					if (lua_isnil(L,3)) {
						type |= PTYPE_TAG_ALLOCSESSION;
					} else {
						session = luaL_checkinteger(L,3);
					}
					int mtype = lua_type(L,4); //得到传递的参数类型
					int mtype = lua_type(L,4); //得到传递的参数类型
					switch (mtype) {
					case LUA_TSTRING: { //如果参数类型是字符串
						void * msg = (void *)lua_tolstring(L,4,&len);  //得到传递的消息的字符串
						if (dest_string) { //如果是字符串地址
							//skynet_sendname最终还是会调用skynet_send
							session = skynet_sendname(context, 0, dest_string, type, session , msg, len);
						} else {  //如果是数字地址
							session = skynet_send(context, 0, dest, type, session , msg, len); 
								//skynet_server.c
								int skynet_send(struct skynet_context * context, uint32_t source, uint32_t destination , int type, int session, void * data, size_t sz) {
									else { //如果目的地址是本节点的
									struct skynet_message smsg;
									smsg.source = source;
									smsg.session = session;
									smsg.data = data;
									smsg.sz = sz;
									if (skynet_context_push(destination, &smsg)) { //将消息压入到目的地址服务的消息队列中供work线程取出

	-- 作用是将第二个参数赋值给proto[typename].dispatch
	skynet.dispatch("lua", function(session, source, cmd, subcmd, ...)
		function skynet.dispatch(typename, func)
			local p = proto[typename]
			if func then  --lua类型的消息一般走这里
				local ret = p.dispatch
				p.dispatch = func
				return ret
			else
				return p and p.dispatch
			end
```

### 对于C服务来说：
下面是几个主要的C服务消息发送的过程
```lua
	//logger服务
	void skynet_error(struct skynet_context * context, const char *msg, ...) 
	​skynet_context_push(logger, &smsg);

	//定时器消息
	function skynet.timeout(ti, func)  
	local session = c.intcommand("TIMEOUT",ti)
			static int lintcommand(lua_State *L)
			result = skynet_command(context, cmd, parm);
					const char * skynet_command(struct skynet_context * context, const char * cmd , const char * param)
					static struct command_func cmd_funcs[] = {
							{ "TIMEOUT", cmd_timeout },
							static const char * cmd_timeout(struct skynet_context * context, const char * param)
									int skynet_timeout(uint32_t handle, int time, int session)
									if (time <= 0)
											skynet_context_push(handle, &message)

	//socket消息
			static void * thread_socket(void *p) //socket
			int skynet_socket_poll()
					int type = socket_server_poll(ss, &result, &more);
					switch (type)
					forward_message(SKYNET_SOCKET_TYPE_DATA, false, &result);
					forward_message(SKYNET_SOCKET_TYPE_CLOSE, false, &result);
					forward_message(SKYNET_SOCKET_TYPE_CONNECT, true, &result);
					forward_message(SKYNET_SOCKET_TYPE_ERROR, true, &result);
					forward_message(SKYNET_SOCKET_TYPE_ACCEPT, true, &result);
					forward_message(SKYNET_SOCKET_TYPE_UDP, false, &result);
							static void forward_message(int type, bool padding, struct socket_message * result)
							skynet_context_push((uint32_t)result->opaque, &message)

	//其他几个调用skynet_context_push的地方：
	static void signal_hup()
			skynet_context_push(logger, &smsg);

	//skynet_timer.c文件中
	static void * thread_timer(void *p)
			void skynet_updatetime(void)
			static void timer_update(struct timer *T)
					static inline void timer_execute(struct timer *T)
					static inline void dispatch_list(struct timer_node *current)
							skynet_context_push(event->handle, &message);
```

由上面的分析可以看到，不管是lua服务还是C服务，最终都是通过调用`skynet_context_push`来实现，`skynet_context_push`的工作主要是将消息结构加入到对应服务的消息队列中而已。