---
layout: post
title: 'skynet源码分析08_master_slave模式'
tags: skynet
---
skynet是支持在不同机器上协作的，之间通过TCP互连。不过有两种模式可以选，一种是master/slave模式，一种是cluster模式，这里说说master/slave模式。

1. skynet的master/slave模式是一个master与多个slave的模式，master与每个slave相连，每个slave又两两互连。master同时会充当一个中心节点的作用，用来协调各个slave的工作。
2. 如果在config中，配置`harbor = 0`，那么skynet会工作带单节点模式下，其他参数(address、master、standalone)都不用设置。
3. 如果在config中配置 harbor 为1-255中的数字，那么skynet会工作在多节点模式下，如果配置了 standalone， 那么此节点是中心节点。
4. 只要是多节点模式， address 与 master 都需要配置，其中 address 为此节点的地址，供本节点监听， master 为外部中心节点的地址，供slave连接(或者供中心节点监听)

本文配合[2016年下旬最新版skynet源码注释](https://github.com/nullscc/skynet_with_note)更佳

## 带着问题去了解
分析master/slave模式，我是带着以下问题去了解的。

* 全局的字符串地址是怎么实现的
* 消息是怎么从一个节点传递到另外一个节点的
* `skynet.uniqueservice`是怎么创建多节点有效的服务的
* 为什么不推荐使用master/slave模式

## 从bootstrap说起
bootstrap 是 skynet 中启动的第二个服务(第一个是 logger)，它是一个临时服务(工作做完就结束了)，主要工作是做一些上层管理类服务的初始化工作。 bootstrap 的部分代码:
```lua
    local standalone = skynet.getenv "standalone"
    -- 获取 config 中的 standalone 参数，如果standalone存在，它应该是一个"ip地址:端口"

    local harbor_id = tonumber(skynet.getenv "harbor" or 0)
    -- 获取 config 中的 harbor 参数

    -- 如果 harbor 为 0 (即工作在单节点模式下)
    if harbor_id == 0 then
        assert(standalone ==  nil)	-- 如果是单节点， standalone 不能配置
        standalone = true
        skynet.setenv("standalone", "true")	-- 设置 standalone 的环境变量为true

        -- 如果是单节点模式，则slave服务为 cdummy.lua
        local ok, slave = pcall(skynet.newservice, "cdummy")
        if not ok then
            skynet.abort()
        end
        skynet.name(".cslave", slave)

    else					-- 如果是多节点模式
        if standalone then	-- 如果是中心节点则启动 cmaster 服务
            if not pcall(skynet.newservice,"cmaster") then
                skynet.abort()
            end
        end

        -- 如果是多节点模式，则 slave 服务为 cslave.lua
        local ok, slave = pcall(skynet.newservice, "cslave")
        if not ok then
            skynet.abort()
        end
        skynet.name(".cslave", slave)
    end

    if standalone then	-- 如果是中心节点则启动 datacenterd 服务
        local datacenter = skynet.newservice "datacenterd"
        skynet.name("DATACENTER", datacenter)
    end
```

从代码可以看出，从配置文件里面读取出 harbor 配置

* 如果是 0，代表是单节点，那么只启动 cdummy 作为 slave 服务的伪装即可
* 如果是非 0(1-255)，代表是多节点，启动 cslave 服务；如果是中心节点(配置了 standalone)，还需要启动 cmaster 与 datacenter 服务

## master/slave模式的C层面的初始化
除了 bootstrap 中启动的服务，还有一个很重要的C服务需要启动:harbor服务(源文件为 service-src/service_harbor.c)，下面先看看C层面的初始化(仅仅看master/slave)

* main 函数中的 skynet_start 函数
```lua
        void skynet_start(struct skynet_config * config)
            skynet_harbor_init(config->harbor); // 初始化 harbor id，用来后续判断是否是非本节点的服务地址
            skynet_handle_init(config->harbor); // 将 harbor id 保存在 struct handle_storage 的高八位
```

* 怎么启动的harbor服务
如果是单节点模式，会启动 cdummy 服务，在 cdummy 的 skynet.start 函数中有:`harbor_service = assert(skynet.launch("harbor", harbor_id, skynet.self()))`

同样，如果是多节点模式，会启动 cslave 服务，在 cdummy 的 skynet.start 函数中有:`harbor_service = assert(skynet.launch("harbor", harbor_id, skynet.self()))`

可见不管是多节点还是单节点模式都会启动 harbor 服务，在skynet进程启动过程中会看到类似于:`[:00000005] LAUNCH harbor 0 4`的字样

## 查看相关各服务的启动工作
从初始化过程我们可以看到，让master/slave模式工作起来的服务主要有:cmaster、cslave、harbor服务，看看这三个服务做了些什么工作。

### 简单说说harbor服务的消息处理函数
注册消息处理函数(可以看出如果有消息过来，会调用mainloop来处理)
```c
    int harbor_init(struct harbor *h, struct skynet_context *ctx, const char * args)
        skynet_callback(ctx, h, mainloop);
```

mainloop的处理:
```c
    static int mainloop(struct skynet_context * context, void * ud, int type, int session, uint32_t source, const void * msg, size_t sz)
        struct harbor * h = ud;
        switch (type) {
            case PTYPE_SOCKET: {		// 收到远端 harbor 的消息
                ...
            case PTYPE_HARBOR: {	// 收到本地 slave 的命令
                harbor_command(h, msg,sz,session,source);
            default: {		// 需要发送消息到远端 harbor 
                // remote message out
                const struct remote_message *rmsg = msg;
                if (rmsg->destination.handle == 0) {	// 如果数字地址为0 即说明采用的是字符串地址
                    if (remote_send_name(h, source , rmsg->destination.name, type, session, rmsg->message, rmsg->sz)) {
                        return 0;
                    }
                } else {
                    if (remote_send_handle(h, source , rmsg->destination.handle, type, session, rmsg->message, rmsg->sz)) {
```

可以看到 harbor 服务的消息处理主要分为三大类:

* 从网络过来的数据包(PTYPE_SOCKET，  一般是两个节点之间的harbor通信)
* 本地发送的请求(PTYPE_HARBOR， 一般是 cslave 服务的请求)
* 需要发送到另一个节点的消息(default, 这个一般是一个节点的服务要调用 skynet.send/skynet.call 发送消息到另外一个节点)

### cmaster服务的工作
```c
    skynet.start(function()
        -- 得到中心节点的地址
        local master_addr = skynet.getenv "standalone"
        skynet.error("master listen socket " .. tostring(master_addr))

        -- 监听中心节点
        local fd = socket.listen(master_addr)

        -- 调用 socket.start 正式开始监听
        socket.start(fd , function(id, addr)
            -- 如果有远端连接过来，会调用此函数，这里是有 slave 连接过来了会调用此函数
            skynet.error("connect from " .. addr .. " " .. id)

            -- 启动数据传输
            socket.start(id)

            -- 调用 handshake 在中心节点记录下这个 slave，并通知其他slave说这个slave连接上来了，让别的slave都去连接这个新的slave)
            local ok, slave, slave_addr = pcall(handshake, id)
            if ok then
                -- 监控 slave 的协程，其实就是对处理 slave 发过来的消息
                skynet.fork(monitor_slave, slave, slave_addr)
            else
                skynet.error(string.format("disconnect fd = %d, error = %s", id, slave))
                socket.close(id)
            end
        end)
    end)
```

从上面可以看到, cmaster 的主要工作为:

* 监听配置文件中 standalone 指定的地址，以便让其他节点连接上来
* 如果有其他 slave 节点连接上来了，记录下这个 slave，并且告诉其他的 slave 节点
* 调用 monitor_slave 函数接收并处理从 slave 节点过来的网路包

### cslave服务的工作
```c
    skynet.start(function()
        -- 得到中心节点的地址
        local master_addr = skynet.getenv "master"

        -- 得到 harbor id
        local harbor_id = tonumber(skynet.getenv "harbor")

        -- 得到本节点的地址
        local slave_address = assert(skynet.getenv "address")

        -- 监听本节点
        local slave_fd = socket.listen(slave_address)
        skynet.error("slave connect to master " .. tostring(master_addr))

        -- 连接 master 节点
        local master_fd = assert(socket.open(master_addr), "Can't connect to master")

        -- 注册消息处理函数，用于处理本地请求
        skynet.dispatch("lua", function (_,_,command,...)
            local f = assert(harbor[command])
            f(master_fd, ...)
        end)

        skynet.dispatch("text", monitor_harbor(master_fd))

        -- 启动 harbor 服务
        harbor_service = assert(skynet.launch("harbor", harbor_id, skynet.self()))

        -- 发送一个 "H" 消息，告诉master:hi, gay!我连接上来了，并且告诉 master:harbor id 和 节点地址
        local hs_message = pack_package("H", harbor_id, slave_address)
        socket.write(master_fd, hs_message)

        -- 远端节点会告诉你有多少个节点已经连接上来了，待会他们会来连接你的
        local t, n = read_package(master_fd)
        assert(t == "W" and type(n) == "number", "slave shakehand failed")
        skynet.error(string.format("Waiting for %d harbors", n))

        -- 开辟一个协程，用于处理中心节点过来的网络包
        skynet.fork(monitor_master, master_fd)
        if n > 0 then
            local co = coroutine.running()
            socket.start(slave_fd, function(fd, addr)
                skynet.error(string.format("New connection (fd = %d, %s)",fd, addr))

                -- 与已连接的老前辈slave 一一建立连接
                if pcall(accept_slave,fd) then
                    local s = 0
                    for k,v in pairs(slaves) do
                        s = s + 1
                    end
                    if s >= n then
                        skynet.wakeup(co)
                    end
                end
            end)
            skynet.wait()
        end
        socket.close(slave_fd)
        skynet.error("Shakehand ready")
        skynet.fork(ready)
    end)
```

从上面代码看到，slave 还比 master 节点要复杂？是的。master节点只需要协调slave节点的工作即可了。

slave节点不仅要连接master，还要连接其他多个slave，还要与harbor服务通信，还要处理本节点其他服务的请求，详细工作表述如下:

* 监听本节点的地址
* 连接master节点并注册"lua" "text"类型的消息处理函数，并启动 harbor 服务，其中monitor_harbor是用来处理本地harbor服务发来的消息的
* 告诉 master我连接上来了，并告诉 master 我是谁(以便master告诉其他的slave，让其他的slave来连接(通过向老的slave发送"C"))，master收到后告诉你有多少个节点已连接了
* 开辟一个协程:monitor_master 来处理中新节点的消息
* 老的slave收到master的"C"的网路包(在 monitor_master 函数中收到)，调用 connect_slave 函数去连接新上来的 slave
* 还是在 connect_slave 函数中，老的slave连接新的slave成功的后，老的slave调用`socket.abandon(fd)`后调用`skynet.send(harbor_service, "harbor", string.format("S %d %d",fd,slave_id))`给harbor服务发送一个 "S"
* 老的slave的harbor服务收到"S"后，调用`skynet_socket_start(h->ctx, fd);`(对应上面的`socket.abandon(fd)`)，并调用 handshake 函数将自己的 harbor id 发给对端新的 slave
* 新的 slave 这时还在执行 accept_slave 函数，收取到老的slave的 harbor id，然后调用`socket.abandon(fd)`并调用`skynet.send(harbor_service, "harbor", string.format("A %d %d", fd, id))`给 harbor 服务发送一个"A xxx xxx"
* harbor 收到 "A xx xx" 后("harbor"类型)，调用`skynet_socket_start(h->ctx, fd);`(对应上一步的`socket.abandon(fd)`)，并调用 handshake 函数将自己的 harbor id 发给老的slave，
* 需要注意的是:这时候两端的harbor收到 "A" 对方的 "S"后，主动连接的一方的`slave->status = STATUS_HANDSHAKE;`，被动连接的一方`slave->status = STATUS_HEADER;`,harbor服务会执行到 push_socket_data 中去，这时候连接建立完成。

到这里我们可以得出一个结论:master与slave网络通信处理一直是在各自的cmaster/cslave服务中，slave与slave的网络通知在连接准备阶段是在cslave中处理，在连接准备完成后，slave与slave的交互全部直接通过各自节点的harbor服务


## 多节点字符串地址的注册
skynet为服务注册一个字符串地址，主要接口有两个:`skynet.register`、`skynet.name`，这两个接口都会调用一个共同的函数:globalname
可以看到，globalname首先判断这个字符串是不是 '.' 开头的，如果是，就注册一个仅仅本节点可见的字符串地址，如果不是，则调用`harbor.globalname(name, handle)`注册一个全skynet网络可见的字符串地址。
```c
    harbor.globalname(name, handle)
        skynet.send(".cslave", "lua", "REGISTER", name, handle)
            function harbor.REGISTER(fd, name, handle)
                globalname[name] = handle	-- 在 slave 服务中缓存住这个全节点有效的名字
                response_name(name)			-- 检查是否有此名字查询的请求阻塞在这里，如果有:返回
                socket.write(fd, pack_package("R", name, handle))	-- 发消息给 master 说:自己要注册这个名字， 然后由 master 将此请求转发给所有 slave
                skynet.redirect(harbor_service, handle, "harbor", 0, "N " .. name)	-- 发消息给 harbor 服务说:我注册这个名字，以便被查找
```

master服务收到"R"的处理如下(向所有的 slave 节点广播 'N' 命令):
```lua
    local function dispatch_slave(fd)
        local t, name, address = read_package(fd)
        if t == 'R' then		-- 注册全局名字
            -- register name
            assert(type(address)=="number", "Invalid request")
            if not global_name[name] then
                global_name[name] = address
            end
            local message = pack_package("N", name, address)
            for k,v in pairs(slave_node) do
                socket.write(v.fd, message)	-- 向所有的 slave 节点广播 'N' 命令
            end
```

cslave服务收到远端服务的"N "的处理如下(缓存住这个地址，然后向harbor服务发送一个'N'):
```c
    elseif t == 'N' then	-- 收到master的从另外 slave 过来的注册新名字的通知
        globalname[id_name] = address	-- 缓存住全局名字
        response_name(id_name)			-- 用于此名字查询的被阻塞请求结果的返回
        if connect_queue == nil then	-- 如果已经准备好了，就给harbor服务发消息，让harbor服务记录下这个地址
            skynet.redirect(harbor_service, address, "harbor", 0, "N " .. id_name)
        end
```

harbor服务收到本地服务的"N "的处理如下:
```c
    case 'N' : {	// 新命名全局名字
        if (s <=0 || s>= GLOBALNAME_LENGTH) {   -- 不能超过16个字符
            skynet_error(h->ctx, "Invalid global name %s", name);
            return;
        }
        update_name(h, rn.name, rn.handle);
            struct keyvalue * node = hash_search(h->map, name);
            if (node == NULL) {
                node = hash_insert(h->map, name);
```

如果这个名字不存在，就会调用 hash_insert 在harbor服务里将这个名字储存起来。

上面几个流程的总结如下:

1. 某服务调起 harbor.REGISTER 请求，请求发给 slave 服务， slave 本身缓存住这个请求在 globalname
2. 写一个 'R' 的命令给 master ， 并且写一个 'N' 命令给此节点的 harbor 服务
3. master 收到这个 'R' 命令， 缓存住名字在 global_name 中， 并且向所有的 slave 节点广播一个 'N' 命令
4. 其他的 slave 节点的 harbor 服务会收到 'N' 命令，并向 harbor 服务写一个 'N'命令

如果A节点的A服务发起了一个注册名字的请求，那么等上面的流程走完以后，有以下服务知道A节点A服务的地址
* A节点的 slave 服务
* master 服务
* A节点的 harbor 服务
* 其他节点的 harbor 服务


## 查询一个全局字符串地址
查询一个全局的字符串地址的接口为:`harbor.queryname`
```lua
    function harbor.queryname(name)
        return skynet.call(".cslave", "lua", "QUERYNAME", name)
        function harbor.QUERYNAME(fd, name)
            if name:byte() == 46 then	-- "." , local name 如果是本节点的服务名字，就直接返回地址
                skynet.ret(skynet.pack(skynet.localname(name)))
            local result = globalname[name]	-- 如果已经缓存过(是此节点的服务)，也直接返回
            if result then
                skynet.ret(skynet.pack(result))
                return
            end
        local queue = queryname[name]
            if queue == nil then	-- 如果为空 说明此名字还没查询过
                socket.write(fd, pack_package("Q", name))
                queue = { skynet.response() }
                queryname[name] = queue
            else					-- 如果不为空 说明此名字已经查询过 但是由于某种原因还没返回(还没注册、slave还没连接上来) 将其加入队列 等注册上来后再返回
                table.insert(queue, skynet.response())
            end
```

可见查询时会向 master 发送一个"Q"命令，slave本身会并创建一个闭包队列，master节点收到'Q'的处理:
```lua
    elseif t == 'Q' then
        -- query name
        local address = global_name[name]
        if address then
            socket.write(fd, pack_package("N", name, address))
```

slave收到"N"后(这里主要看response_name):
```lua
    elseif t == 'N' then	-- 收到master的从另外 slave 过来的注册新名字的通知
        globalname[id_name] = address	-- 缓存住全局名字
        response_name(id_name)			-- 用于此名字查询的被阻塞请求结果的返回
            local address = globalname[name]
            if queryname[name] then
                local tmp = queryname[name]
                queryname[name] = nil
                for _,resp in ipairs(tmp) do
                    resp(true, address)
                end
            end
        if connect_queue == nil then	-- 如果已经准备好了，就给harbor服务发消息，让harbor服务记录下这个地址
            skynet.redirect(harbor_service, address, "harbor", 0, "N " .. id_name)
        end
```

上面的操作是将请求时的闭包队列一一取出来，发送唤醒这个闭包队列，以便查询的一方收到结果，至此此次查询结束。查询全局字符串地址的流程总结为:

1. 调起 harbor.QUERYNAME 请求， 发给 slave 服务
2. 如果确实是一个非本节点的全局名字服务，会有两种情况:
    * 已经查询过，直接返回；
    * 还没有查询过，如果此查询队列为空:发送一个 'Q' 命令给 master 服务 创建查询队列，如果已有查询队列:将其加入查询队列即可，master 服务收到 'Q' 后:看有没有此名字的服务注册上来过，如果有: 发送一个 'N' 命令给这个 slave， slave 收到这个 'N' 命令后 缓存主全局名字，并且给查询的服务返回相应的消息 并且给 harbor 服务发一个 'N' 命令(让harbor服务记录下这个地址)


## 从skynet.send函数看多节点模式消息的发送
我们知道，skynet.send 与 skynet.call都是可以直接发送到不同节点的地址的，这是如何实现的呢？
```c
    function skynet.send(addr, typename, ...)
        return c.send(addr, p.id, 0 , p.pack(...))
            static int lsend(lua_State *L)
                if (dest_string) {	//如果是字符串地址
                    //skynet_sendname 最终还是会调用 skynet_send
                    session = skynet_sendname(context, 0, dest_string, type, session , msg, len);
                } else {	//如果是数字地址
                    session = skynet_send(context, 0, dest, type, session , msg, len);	
                }
```

### 先看数字地址的情况
```c
    session = skynet_send(context, 0, dest, type, session , msg, len);	
        int skynet_send(struct skynet_context * context, uint32_t source, uint32_t destination , int type, int session, void * data, size_t sz) 
            if (skynet_harbor_message_isremote(destination)) { //如果目的地址不是本节点的(通过地址的高八位来判断)
                skynet_harbor_send(rmsg, source, session);
                    rmsg->destination.handle = destination;
                    skynet_context_send(REMOTE, rmsg, sizeof(*rmsg) , source, type , session);	// 发送消息到 harbor 服务
                        skynet_mq_push(ctx->queue, &smsg);
```

通过高八位判断是不是非本节点的地址，如果非本节点，将消息压入到harbor服务的消息队列。harbor服务会收到处理这个消息:
```c
    default: {		// 需要发送消息到远端 harbor 
        // remote message out
        const struct remote_message *rmsg = msg;
        if (rmsg->destination.handle == 0) {	// 如果数字地址为0 即说明采用的是字符串地址
            if (remote_send_name(h, source , rmsg->destination.name, type, session, rmsg->message, rmsg->sz)) {
                return 0;
            }
        } else {
            if (remote_send_handle(h, source , rmsg->destination.handle, type, session, rmsg->message, rmsg->sz)) {
                return 0;
            }
        }
```

由于是数字地址，所以会调用`remote_send_handle`(只考虑非本节点的)
```c
    if (remote_send_handle(h, source , rmsg->destination.handle, type, session, rmsg->message, rmsg->sz))
        send_remote(context, s->fd, msg,sz,&cookie);
            skynet_socket_send(ctx, fd, sendbuf, sz_header+4);	// 在这里发送一个 "D" 给本地管道，本地管道收到后 封装成网络包发出去
                int64_t wsz = socket_server_send(SOCKET_SERVER, id, buffer, sz);
                    send_request(ss, &request, 'D', sizeof(request.u.send));
```

远端harbor服务收到这个消息:
```c
    case PTYPE_SOCKET: {		// 收到远端 harbor 的消息
        const struct skynet_socket_message * message = msg;
        switch(message->type) {
        case SKYNET_SOCKET_TYPE_DATA:
            push_socket_data(h, message);
                // go though
                case STATUS_CONTENT: {
                    int need = s->length - s->read;
                    if (size < need) {
                        memcpy(s->recv_buffer + s->read, buffer, size);
                        s->read += size;
                        return;
                    }
                    memcpy(s->recv_buffer + s->read, buffer, need);
                    // 分发到对应的服务上去
                    forward_local_messsage(h, s->recv_buffer, s->length);
```

### 再看字符串地址的情况
```c
    session = skynet_sendname(context, 0, dest_string, type, session , msg, len);
        copy_name(rmsg->destination.name, addr);
        rmsg->destination.handle = 0;
        skynet_harbor_send(rmsg, source, session);  -- 会往harbor服务压入一条消息(和前面的数字地址地址一样)
```

harbor服务收到消息:
```c
    if (rmsg->destination.handle == 0) {	// 如果数字地址为0 即说明采用的是字符串地址
        if (remote_send_name(h, source , rmsg->destination.name, type, session, rmsg->message, rmsg->sz)) {
            if (node->value == 0) {		// 如果harbor服务没有缓存这个字符串地址(可能是注册字符串地址的'N'命令还没发过来)
                if (node->queue == NULL) {
                    node->queue = new_queue();
                }
                struct remote_message_header header;
                header.source = source;
                header.destination = type << HANDLE_REMOTE_SHIFT;
                header.session = (uint32_t)session;
                push_queue(node->queue, (void *)msg, sz, &header);	// 把这个消息加入到队列中，等待后续处理(将来收到'Q'命令的响应会pop_queue，然后继续将消息转发出去)
                char query[2+GLOBALNAME_LENGTH+1] = "Q ";	
                query[2+GLOBALNAME_LENGTH] = 0;
                memcpy(query+2, name, GLOBALNAME_LENGTH);
                skynet_send(h->ctx, 0, h->slave, PTYPE_TEXT, 0, query, strlen(query)); // 那么发送一个"Q"给 cslave 服务
                return 1;
            } else {
                return remote_send_handle(h, source, node->value, type, session, msg, sz);
```

可以看出harbor服务会根据 rmsg->destination.handle 是否为0来区分是一个字符串地址还是数字地址。远端harbor服务收到这个包，会根据 rmsg->destination 来找到自己节点的那个服务然后转发给那个服务。

所以，发消息给对端另外节点的流程是:

* 看是字符串地址还是数字地址，如果是数字地址

        1. 发送消息到harbor，harbor服务会给管道发送一个'D'的命令，管道收到这个'D'命令，将消息通过网络发到对端的harbor服务
        2. 对端harbor服务收到后，根据地址将消息压入到对应服务的消息队列
* 如果是字符串地址

        1. 发送消息到harbor，harbor服务会给管道发送一个'D'的命令，管道收到这个'D'命令，将消息通过网络发到对端的harbor服务
        2. 根据`rmsg->destination.handle`检测到是字符串地址，看本地缓存有没有此字符串地址，如果没有:首先将此次请求加入到队列中，然后发送消息给本节点的cslave服务，本地的cslave收到后会将地址返回给harbor服务(或通过cmaster转发回去)，当重新收到这个地址时，从队列中取出，将消息压入到相应的服务中去。
        3. 如果本地缓存有此字符串地址，直接将此字符串地址对应的数字地址取出，然后往对应的服务压入消息


## skynet.uniqueservice的实现
skynet.uniqueservice 函数的第一参数若为true，那么就会创建一个全skynet网络都有效的服务。
```lua
    function skynet.uniqueservice(global, ...)  -- .service 为 service_mgr
        if global == true then
            return assert(skynet.call(".service", "lua", "GLAUNCH", ...))
        else
            return assert(skynet.call(".service", "lua", "LAUNCH", global, ...))
        end
    end
```

### 先看global为字符串的情况(不考虑snax)
```lua
    return assert(skynet.call(".service", "lua", "LAUNCH", global, ...))
        return waitfor(service_name, skynet.newservice, realname, subname, ...)
            local function waitfor(name , func, ...)
                local s = service[name]
                if type(s) == "number" then	-- 如果已经有了，就直接返回
                    return s
                end
                local co = coroutine.running()

                if s == nil then
                    s = {}
                    service[name] = s
                elseif type(s) == "string" then
                    error(s)
                end

                assert(type(s) == "table")

                if not s.launch and func then	-- 如果是首次调用,可以直接返回
                    s.launch = true
                    return request(name, func, ...)
                    local function request(name, func, ...)	-- 这里的 func 为 skynet.newservice
                        local ok, handle = pcall(func, ...)
                        local s = service[name]
                        assert(type(s) == "table")
                        if ok then
                            service[name] = handle
                        else
                            service[name] = tostring(handle)
                        end

                        for _,v in ipairs(s) do		-- 唤醒阻塞的协程
                            skynet.wakeup(v)
                        end

                        if ok then
                            return handle
                end

                table.insert(s, co)				-- 后续的调用都需要阻塞在这里
                skynet.wait()
                s = service[name]
                if type(s) == "string" then
                    error(s)
                end
                assert(type(s) == "number")
                return s
            end
```

可以看出如果global不为ture，则认为它是一个字符串，如果是首次针对此字符串调用 skynet.uniqueservice ，那么会调用 skynet.newservice 创建一个服务并返回地址

如果不是首次调用并且首次调用还没有完全完成，那么会将当前协程插入到一个协程队列中，然后 skynet.wait 等待首次调用完成后唤醒它

如果不是首次调用并且首次调用已经完全完成了，那么会直接返回一个地址(首次调用完成后创建的服务的地址)。

### 再看global为true的情况(不考虑snax)
skynet.start中可以看到，会针对 standalone 不同注册不同的消息处理函数
```lua
    if skynet.getenv "standalone" then 	-- 主节点,单节点模式
        skynet.register("SERVICE")      -- 只有主节点中会注册
        register_global()
    else								-- 多节点模式中的非主节点
        register_local()
    end
```

这里分三种情况，之前看 bootstrap 初始化时已经知道(注意在多节点模式中只有主节点中才会注册一个"SERVICE"名字服务):

* 单节点模式，standalone = true
* 多节点模式中的主节点， standalone = "ip:port"
* 多节点模式中的非主节点， standalone = nil

只看多节点模式，首先是主节点中，如果要创建一个全skynet网络有效的服务。
```lua
    return assert(skynet.call(".service", "lua", "GLAUNCH", ...))
        function cmd.GLAUNCH(name, ...) --local function register_global()
            return cmd.LAUNCH(global_name, ...)
```

后面就和创建本地服务一样，可以看出，在主节点中，就只是在主节点中创建了一个本地服务。

再看非主节点:
```lua
    return assert(skynet.call(".service", "lua", "GLAUNCH", ...))
        function cmd.GLAUNCH(...)       -- local function register_local()
            return waitfor_remote("LAUNCH", ...)
                return waitfor(local_name, skynet.call, "SERVICE", "lua", cmd, global_name, ...)
                -- 注意此时的 func变为 skynet.call 了， cmd为 "LAUNCH"
```

可以看到会向 "SERVICE" 服务发送一个 "LAUNCH" 消息，然后再看主节点中的 service_mgr 服务收到这个消息怎么处理的:
```lua
    function cmd.LAUNCH(service_name, subname, ...)
        return waitfor(service_name, skynet.newservice, realname, subname, ...)
```

可以看到会在本地创建一个服务，创建完成后会返回给远端的非主节点的 service_mgr 服务，并储存这个服务的地址，以便后续调用

顺便一提 skynet.queryservice 对应 skynet.uniqueservice ，如果第一个参数为true，那么会阻塞的等待某个服务创建好才返回(当然服务如果已经存在就直接返回了)，这没啥好说的，都体现在 waitfor 函数里面了。

## master/slave模式的优劣
在我个人看来,master/slave最大的缺点在于不能很好的处理某个节点异常断开的情况。

但是相比于cluster，它可以依靠一条socket连接便可以在双方自由通信。

所以"官方"建议的是在不跨机房的机器中使用master/slave，在不同机房中或者跨互联网网络中使用cluster。(我觉得即便是不跨机房中，也要考虑异常断开情况，当然skynet有提供监控这种异常的手段，但是不可避免的还有一些额外工作要做。)