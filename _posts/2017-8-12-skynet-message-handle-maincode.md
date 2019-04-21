---
layout: post
title: 'skynet消息处理相关代码(协程)'
tags: skynet
---
这里只是一些代码片段，为了方便对照着分析而做的,为了简单把lualib/skynet下的主要函数抽出来。  
可在网址的目录页快速跳到对应的函数。

对应的"详细"分析:[skynet源码分析04之消息处理(协程)](http://nulls.cc/post/skynet_srccode_analysis04_message_handle.html)

## suspend
算是消息处理的核心函数了
```lua
	-- suspend is local function
	function suspend(co, result, command, param, size)
		if not result then	-- 当协程错误发生时
			local session = session_coroutine_id[co]
			if session then -- coroutine may fork by others (session is nil)
				local addr = session_coroutine_address[co]
				if session ~= 0 then
					-- only call response error
					c.send(addr, skynet.PTYPE_ERROR, session, "")
				end
				session_coroutine_id[co] = nil
				session_coroutine_address[co] = nil
			end
			error(debug.traceback(co,tostring(command)))
		end
		if command == "CALL" then
			session_id_coroutine[param] = co --以session为key记录协程
		elseif command == "SLEEP" then
			session_id_coroutine[param] = co
			sleep_session[co] = param
		elseif command == "RETURN" then
			local co_session = session_coroutine_id[co]
			local co_address = session_coroutine_address[co]
			if param == nil or session_response[co] then
				error(debug.traceback(co))
			end
			session_response[co] = true
			local ret
			if not dead_service[co_address] then
				ret = c.send(co_address, skynet.PTYPE_RESPONSE, co_session, param, size) ~= nil
				if not ret then
					-- If the package is too large, returns nil. so we should report error back
					c.send(co_address, skynet.PTYPE_ERROR, co_session, "")
				end
			elseif size ~= nil then
				c.trash(param, size)
				ret = false
			end
			return suspend(co, coroutine_resume(co, ret))
			--coroutine_resume会恢复处理函数中的协程执行(会从skynet.ret中的coroutine_yield("RETURN", msg, sz)处返回)，到这里处理函数执行完毕了，即co_create中的f函数执行完毕了
		elseif command == "RESPONSE" then
			local co_session = session_coroutine_id[co]
			local co_address = session_coroutine_address[co]
			if session_response[co] then
				error(debug.traceback(co))
			end
			local f = param
			local function response(ok, ...)
				if ok == "TEST" then
					if dead_service[co_address] then
						release_watching(co_address)
						unresponse[response] = nil
						f = false
						return false
					else
						return true
					end
				end
				if not f then
					if f == false then
						f = nil
						return false
					end
					error "Can't response more than once"
				end

				local ret
				if not dead_service[co_address] then
					if ok then
						ret = c.send(co_address, skynet.PTYPE_RESPONSE, co_session, f(...)) ~= nil
						if not ret then
							-- If the package is too large, returns false. so we should report error back
							c.send(co_address, skynet.PTYPE_ERROR, co_session, "")
						end
					else
						ret = c.send(co_address, skynet.PTYPE_ERROR, co_session, "") ~= nil
					end
				else
					ret = false
				end
				release_watching(co_address)
				unresponse[response] = nil
				f = nil
				return ret
			end
			watching_service[co_address] = watching_service[co_address] + 1
			session_response[co] = true
			unresponse[response] = true
			return suspend(co, coroutine_resume(co, response))
		elseif command == "EXIT" then	-- 到这里对于收到消息的一方来说这次消息完全处理完毕
			-- coroutine exit
			local address = session_coroutine_address[co]
			release_watching(address)
			session_coroutine_id[co] = nil
			session_coroutine_address[co] = nil
			session_response[co] = nil
		elseif command == "QUIT" then
			-- service exit
			return
		elseif command == "USER" then
			-- See skynet.coutine for detail
			error("Call skynet.coroutine.yield out of skynet.coroutine.resume\n" .. debug.traceback(co))
		elseif command == nil then
			-- debug trace
			return
		else
			error("Unknown command : " .. command .. "\n" .. debug.traceback(co))
		end
		dispatch_wakeup()
		dispatch_error_queue()
	end
```

## co_create
主要负责新协程的创建以及老协程的复用
```lua
	local function co_create(f)
		local co = table.remove(coroutine_pool)				-- 从协程池取出一个协程
		if co == nil then 									-- 如果没有可用的协程
			co = coroutine.create(function(...)				-- 创建新的协程
				f(...)										-- 当调用coroutine.resume时，执行函数f
				while true do		
					f = nil									-- 将函数置空
					coroutine_pool[#coroutine_pool+1] = co 	-- 协程执行完后，回收协程
					f = coroutine_yield "EXIT"				-- 协程执行完后，让出执行
					f(coroutine_yield())
				end
			end)
		else
			coroutine_resume(co, f)							-- 这里coroutine_resume对应的是上面的coroutine_yield "EXIT"
		end
		return co
	end
```

## raw_dispatch_message
当有消息过来时就会跑到这里来
```lua
	local function raw_dispatch_message(prototype, msg, sz, session, source)
		-- skynet.PTYPE_RESPONSE = 1, read skynet.h
		if prototype == 1 then 		-- 处理远端发送过来的返回值
			local co = session_id_coroutine[session]
			if co == "BREAK" then
				session_id_coroutine[session] = nil
			elseif co == nil then
				unknown_response(session, source, msg, sz)
			else
				session_id_coroutine[session] = nil
				suspend(co, coroutine_resume(co, true, msg, sz))
				-- 唤醒yield_call中的coroutine_yield("CALL", session)
			end
		else
			local p = proto[prototype]
			if p == nil then
				if session ~= 0 then	-- 如果是需要返回值的，那么告诉源服务，说"我对你来说是dead_service不要再发过来了"
					c.send(source, skynet.PTYPE_ERROR, session, "")
				else
					unknown_request(session, source, msg, sz, prototype)
				end
				return
			end
			local f = p.dispatch
			if f then
				local ref = watching_service[source]
				if ref then
					watching_service[source] = ref + 1
				else
					watching_service[source] = 1
				end
				local co = co_create(f)
				session_coroutine_id[co] = session
				session_coroutine_address[co] = source
				suspend(co, coroutine_resume(co, session,source, p.unpack(msg,sz)))
			else
				unknown_request(session, source, msg, sz, proto[prototype].name)
			end
		end
	end
```

## yield_call
```lua
	local function yield_call(service, session)
		watching_session[session] = service
		local succ, msg, sz = coroutine_yield("CALL", session)	--会让出到raw_dispatch_message中的第二个suspend函数中，即执行:suspend(true, "CALL", session)
		watching_session[session] = nil
		if not succ then
			error "call failed"
		end
		return msg,sz
	end
```

## skynet.call
```lua
	function skynet.call(addr, typename, ...)
		local p = proto[typename]
		local session = c.send(addr, p.id , nil , p.pack(...))		-- 发送消息
		-- 由于skynet.call是需要返回值的，所以c.send的第三个参数表示由框架自动分配一个session，以便返回时根据相应的session找到对应的协程进行处理
		if session == nil then
			error("call to invalid address " .. skynet.address(addr))
		end
		return p.unpack(yield_call(addr, session))					-- 阻塞等待返回值
	end
```

## skynet.ret与skynet.retpack
```lua
	function skynet.ret(msg, sz)
		msg = msg or ""
		return coroutine_yield("RETURN", msg, sz)
		--会让出到raw_dispatch_message函数中，参数给suspend,就成为:suspend(co, true, "RETURN", msg, sz)
	end

	function skynet.retpack(...)
		return skynet.ret(skynet.pack(...))
	end
```

## skynet.dispatch
```lua
	-- 将func赋值给p.dispatch
	function skynet.dispatch(typename, func)
		local p = proto[typename]
		if func then  --lua类型的消息一般走这里
			local ret = p.dispatch
			p.dispatch = func
			return ret
		else
			return p and p.dispatch
		end
	end
```

## skynet.dispatch_message
```lua
	function skynet.dispatch_message(...)
		local succ, err = pcall(raw_dispatch_message,...)
		while true do
			local key,co = next(fork_queue)
			if co == nil then
				break
			end
			fork_queue[key] = nil
			local fork_succ, fork_err = pcall(suspend,co,coroutine_resume(co))
			if not fork_succ then
				if succ then
					succ = false
					err = tostring(fork_err)
				else
					err = tostring(err) .. "\n" .. tostring(fork_err)
				end
			end
		end
		assert(succ, tostring(err))
	end
```

## skynet.start
```lua
	function skynet.start(start_func)
		c.callback(skynet.dispatch_message)
		skynet.timeout(0, function()
			skynet.init_service(start_func)
		end)
	end
```