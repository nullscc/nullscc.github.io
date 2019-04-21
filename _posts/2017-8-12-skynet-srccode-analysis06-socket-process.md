---
layout: post
title: 'skynet源码分析06-socket处理流程'
tags: skynet
---
其实skynet的socket处理流程一句话就能说清的，就是中间处理有点绕。在我看来核心点在于理解各个过程的类型转换(`struct socket` 的 type 变化)，及gate服务即可。

本文配合[2016年下旬最新版skynet源码注释](https://github.com/nullscc/skynet_with_note)更佳

## 初始化
```lua
    void skynet_socket_init()
        SOCKET_SERVER = socket_server_create();
            poll_fd efd = sp_create();	//生成epoll专用的描述符
            if (pipe(fd)) {		//创建一个管道，fd[1]为写入端，fd[0]为读取端
                if (sp_add(efd, fd[0], NULL)) {		//将管道的读取端给epoll管理，skynet本地需要监听、绑定某个端口时都会从上层往管道的写端发送cmd
                    for (i=0;i<MAX_SOCKET;i++) {			// MAX_SOCKET为65536
                        struct socket *s = &ss->slot[i];	//ss->slot[i]为一个struct socket
                        s->type = SOCKET_TYPE_INVALID;
```

很简单：创建一个管道，管道的读端给epoll管理；分配一个`struct socket_server`统领全局(其中有65535个`struct socket`结构体，每个此结构体都对应一个socket描述符，并将所有的 type 设置为`SOCKET_TYPE_INVALID`)

## 等待有事件发生
这里说的有事件发生，包括两种:1.管道的读端有数据可读 2.socket描述符可读/可写。看代码：
```lua
    static void * thread_socket(void *p)
        int r = skynet_socket_poll();
            int type = socket_server_poll(ss, &result, &more);
                if (ss->checkctrl) {	//控制是否去检查本地从管道写过来的请求
                    if (has_cmd(ss)) {	//判断管道的接收描述符是不是有请求过来
                        int type = ctrl_cmd(ss, result);	//如果有就得到请求的类型
                    if (ss->event_index == ss->event_n) { //如果event_index等于event_n，说明已经处理完了
                    ss->event_n = sp_wait(ss->event_fd, ss->ev, MAX_EVENT);		//等待有事情发生， 返回的是需要处理的事件个数
                    switch (s->type) {
                    case SOCKET_TYPE_CONNECTING:// 主动connect得到远端相应
                        return report_connect(ss, s, result);	// 正常的话描述符类型为 SOCKET_TYPE_CONNECTED
                    case SOCKET_TYPE_LISTEN: {	// listen完以后管道再接收一个"S"命令状态就变为SOCKET_TYPE_LISTEN了
                        int ok = report_accept(ss, s, result);
                        if (ok > 0) {	//accept成功后会大于0
                            return SOCKET_ACCEPT;		
                    ...
                    default:
                        if (e->read) {	//有数据可读,在sp_wait中进行设置
                            int type;
                            if (s->protocol == PROTOCOL_TCP) {
                                type = forward_message_tcp(ss, s, result);		// 正常的话返回 SOCKET_DATA
            switch (type) {
                forward_message(SKYNET_SOCKET_TYPE_xxx, false/xxx, &result);
                    sm = (struct skynet_socket_message *)skynet_malloc(sz);
                    sm->type = type;
                    sm->id = result->id;
                    sm->ud = result->ud;

                    struct skynet_message message;
                    message.source = 0;
                    message.session = 0;
                    message.data = sm;
                    message.sz = sz | ((size_t)PTYPE_SOCKET << MESSAGE_TYPE_SHIFT);
                    
                    if (skynet_context_push((uint32_t)result->opaque, &message)) {
```

从上面代码可以看到，socket线程做的事:

1. `socket_server_poll`监控有没有事件发生
2. 如果有事件发生，看是不是从管道过来的，如果是管道过来的就调用`ctrl_cmd`去处理，如果是从 socket 描述符过来的，根据 s->type 进行相应的处理
3. 最后根据`socket_server_poll`的返回值调用 `forward_message`组装一个装有数据为`struct skynet_socket_message`的结构体的消息结构:`struct skynet_message`(消息类型为 PTYPE_SOCKET)
4. 消息结构`struct skynet_message`组装完毕后，调用`skynet_context_push`将其压入对应服务的消息队列，这样gate服务就知道远端有数据过来了。

## 从TCP的三路握手及客户端数据的接收看socket处理
"等待有事件发生"这个流程说起来很简单，其实要理清楚并不容易。我们还是从TCP的三路握手来进行分析吧(从已知推断未知可以省不少力气)。
我们从 example/watchdog.lua 与 example/agent.lua入手分析。
从例子的watchdog服务我们很容易知道进行三路握手的上层接口:

* 监听:
```lua
        skynet.call(gate, "lua", "open" , conf)
            function CMD.open(source, conf )
                socket = socketdriver.listen(address, port)
                socketdriver.start(socket)
                return handler.open(source, conf)
                    watchdog = conf.watchdog or source
```

* client连接成功
```lua
        -- watchdog服务会收到 'socket' 'open' 的消息
        function SOCKET.open(fd, addr)
            agent[fd] = skynet.newservice("agent")
            skynet.call(agent[fd], "lua", "start", { gate = gate, client = fd, watchdog = skynet.self() })
```

* gate服务转发client过来的数据
如果我们我们想某个服务直接收到消息，而不是发给watchdog(上面的`skynet.call(agent[fd], "lua", "start"`就在做这样的事)
```lua
        skynet.call(gate, "lua", "forward", fd)
            c.agent = address or source
            gateserver.openclient(fd)
                socketdriver.start(fd)
```

* 当客户端有数据过来
当客户端有数据过来agent会收到'client'类型的消息

## 从socketdriver与gate的接口分析socket处理流程
从三路握手及数据的接收来看，主要流程在 socketdriver库与gate服务。所以主要分析在这两处的工作情况。
### 首先看socketdriver.listen的调用
```lua
    socketdriver.listen
        { "listen", llisten },
            static int llisten(lua_State *L) 
                int id = skynet_socket_listen(ctx, host,port,backlog);
                    uint32_t source = skynet_context_handle(ctx);		// 哪个服务调用就取得那个服务的地址
                    return socket_server_listen(SOCKET_SERVER, source, host, port, backlog);
                        int fd = do_listen(addr, port, backlog);
                            int listen_fd = do_bind(host, port, IPPROTO_TCP, &family);
                            if (listen(listen_fd, backlog) == -1) {
                        int id = reserve_id(ss);	//为此socket描述符分配一个id给上层使用
                        request.u.listen.opaque = opaque;		//opaque就是调用listen的服务的地址                            
                        send_request(ss, &request, 'L', sizeof(request.u.listen));
                            request->header[6] = (uint8_t)type;
                            request->header[7] = (uint8_t)len;
                            int n = write(ss->sendctrl_fd, &request->header[6], len+2);	// 写到管道的发送端/写端
```

从上面看出`socketdriver.listen`发生的事情主要为:
1. 调用bind绑定端口
2. 调用listen监听端口
3. 调用write向本地的管道的写端写了一个 'L'命令
从现在开始我们需要关注`struct socket`中的`type`字段，即`s-type`，由于没有看到`s-type`的变化，所以仍为最初的`SOCKET_TYPE_INVALID`

接上面的`socketdriver.listen`后续流程:由于向管道写了一个'L'命令，所以`socket_server_poll`有事件发生了:管道的读端可读
`socket_server_poll`中的has_cmd的select返回为1，执行`ctrl_cmd`
```lua
    int type = ctrl_cmd(ss, result);	//如果有就得到请求的类型
        switch (type) {
            case 'L':	//返回:-1、SOCKET_ERROR
                return listen_socket(ss,(struct request_listen *)buffer, result);
                    struct socket *s = new_fd(ss, id, listen_fd, PROTOCOL_TCP, request->opaque, false);	
                    // 这里的request->opaque就是调用最上层调用监听动作服务的地址  注意这里最后一个参数是false，代表这里还没有将监听描述符给epoll管理
                    s->type = SOCKET_TYPE_PLISTEN;
                    return -1;
```

由于`ctrl_cmd`返回-1，所以此socket描述符此轮处理结束。此时`s->type = SOCKET_TYPE_PLISTEN`，此socket描述符还没有被epoll管理

### socketdriver.listen后紧接着的socketdriver.start
```lua
    socketdriver.start
        static int lstart(lua_State *L) {
            skynet_socket_start(ctx,id);
                uint32_t source = skynet_context_handle(ctx);
                socket_server_start(SOCKET_SERVER, source, id);
                    struct request_package request;
                    request.u.start.id = id;
                    request.u.start.opaque = opaque;
                    send_request(ss, &request, 'S', sizeof(request.u.start));
```

由于向管道写了一个'S'命令，所以`socket_server_poll`有事件发生了:管道的读端可读
```lua
    int type = ctrl_cmd(ss, result);
        switch (type) {
            case 'S':	//listen与accept后都会调用'S' 返回:SOCKET_ERROR、SOCKET_OPEN
                return start_socket(ss,(struct request_start *)buffer, result);
                    int id = request->id;
                    result->id = id;
                    result->opaque = request->opaque;		// 服务的地址
                    result->ud = 0;
                    result->data = NULL;
                    struct socket *s = &ss->slot[HASH_ID(id)];
                    if (s->type == SOCKET_TYPE_PACCEPT || s->type == SOCKET_TYPE_PLISTEN) {
                        if (sp_add(ss->event_fd, s->fd, s)) {   // 将s->fd加入epoll
                            ...
                        s->type = (s->type == SOCKET_TYPE_PACCEPT) ? SOCKET_TYPE_CONNECTED : SOCKET_TYPE_LISTEN;
                        s->opaque = request->opaque;
                        result->data = "start";
                        return SOCKET_OPEN;
```

此时此socket描述符已经被epoll管理了，并且`s->type = SOCKET_TYPE_LISTEN`
由于`ctrl_cmd`返回`SOCKET_OPEN`，所以`socket_server_poll`也返回`SOCKET_OPEN`
```lua
    int skynet_socket_poll()
        int type = socket_server_poll(ss, &result, &more);
            switch (type) {
                case SOCKET_OPEN:
                    forward_message(SKYNET_SOCKET_TYPE_CONNECT, true, &result);
                    static void forward_message(int type, bool padding, struct socket_message * result) {
                        struct skynet_socket_message *sm;
                        sm = (struct skynet_socket_message *)skynet_malloc(sz);
                        sm->type = type;
                        if (padding) {
                            sm->buffer = NULL;
                            memcpy(sm+1, result->data, sz - sizeof(*sm));
                        struct skynet_message message;
                        message.source = 0;
                        message.session = 0;
                        message.data = sm;
                        message.sz = sz | ((size_t)PTYPE_SOCKET << MESSAGE_TYPE_SHIFT);
                        
                        if (skynet_context_push((uint32_t)result->opaque, &message)) {
```

由于`result->opaque`是gate服务的地址(前面已经提过)，所以gate服务会收到一个消息。消息类型为`PTYPE_SOCKET`
```lua
    -- gate服务注册的 PTYPE_SOCKET 的消息处理函数
    skynet.register_protocol {
        name = "socket",
        id = skynet.PTYPE_SOCKET,	-- PTYPE_SOCKET = 6
        unpack = function ( msg, sz )
            return netpack.filter( queue, msg, sz)
        end,
        dispatch = function (_, _, q, type, ...)
            queue = q
            if type then
                MSG[type](...)
            end
        end
    }
```

gate服务收到 PTYPE_SOCKET 类型的消息后，先解包
```lua
    netpack.filter( queue, msg, sz)
        static int lfilter(lua_State *L)
            struct skynet_socket_message *message = lua_touserdata(L,2);
            int size = luaL_checkinteger(L,3);
            char * buffer = message->buffer;
            switch(message->type) {
                case SKYNET_SOCKET_TYPE_CONNECT:	// 'L'->'S'后
                // ignore listen fd connect
                return 1;		// 这里其实返回的是上层传过来的 queue
```

我们看到解包以后返回的仅仅是一个queue，而这个queue还是上层传递过来的，所以可以认为这次流程没有任何效果。
但是我们需要关注gate服务的queue了，如果第一次分析，并不需要关注这个东西，我们关注流程就行了。queue其实是用来接收数据的时候配合protobuf/sproto以便接收到一个完整的包。因为我们最初默认的收包大小为`#define MIN_READ_BUFFER 64`,可能我们收到的不是一个完整的协议包，这时候就需要靠queue来临时将这些"不完整"的数据储存起来。

### 客户端调用connect函数连接
如果这时客户端调用connect函数想要与服务端建立一条TCP的socket连接，由于这时候此socket描述符已经被epoll管理，所以`socket_server_poll`中的`sp_wait`函数会返回
```lua
    ss->event_n = sp_wait(ss->event_fd, ss->ev, MAX_EVENT);		//等待有事情发生， 返回的是需要处理的事件个数
        struct event *e = &ss->ev[ss->event_index++];
            struct socket *s = e->s;    // 取出自定义数据
            switch (s->type) {
                case SOCKET_TYPE_LISTEN: {	// listen完以后管道再接收一个"S"命令状态就变为SOCKET_TYPE_LISTEN了
                    int ok = report_accept(ss, s, result);
                    static int report_accept(struct socket_server *ss, struct socket *s, struct socket_message *result)
                        int client_fd = accept(s->fd, &u.s, &len);	// 返回已连接描述符
                        int id = reserve_id(ss);
                        socket_keepalive(client_fd);
                        struct socket *ns = new_fd(ss, id, client_fd, PROTOCOL_TCP, s->opaque, false);
                        ns->type = SOCKET_TYPE_PACCEPT;		// 创建已连接描述符后的状态为 SOCKET_TYPE_PACCEPT
                        result->opaque = s->opaque;			// 调用的服务的地址，继承的监听描述符的
                        result->id = s->id;
                        result->ud = id;					// 当是accept时ud为id
                        result->data = NULL;
                        if (inet_ntop(u.s.sa_family, sin_addr, tmp, sizeof(tmp))) {
                            snprintf(ss->buffer, sizeof(ss->buffer), "%s:%d", tmp, sin_port);
                            result->data = ss->buffer;		// data为服务的 "地址:端口号"组成的字符串
                    if (ok > 0) {	//accept成功后会大于0
                        return SOCKET_ACCEPT;	
```

从上面可以看到如果客户端调用connect函数，服务端知道了后会调用accept函数，如果成功会返回一个新的已连接描述符，为新的已连接描述符分配id给上层使用，并且已连接描述符`s->type = SOCKET_TYPE_PACCEPT;`，但是此时此已连接描述符还没有加入epoll事件管理器

由于`socket_server_poll`函数返回`SOCKET_ACCEPT`，回到`skynet_socket_poll`处理此返回值
```lua
    case SOCKET_ACCEPT:	//说明acccpt成功了
        forward_message(SKYNET_SOCKET_TYPE_ACCEPT, true, &result);
            static void forward_message(int type, bool padding, struct socket_message * result) 
                struct skynet_socket_message *sm;
                if (padding) {
                    if (result->data) {
                        size_t msg_sz = strlen(result->data);
                        if (msg_sz > 128) {
                            msg_sz = 128;
                        }
                        sz += msg_sz;
                sm = (struct skynet_socket_message *)skynet_malloc(sz);
                sm->type = type;
                sm->id = result->id;
                sm->ud = result->ud;
                if (padding) {
                    sm->buffer = NULL;
                    memcpy(sm+1, result->data, sz - sizeof(*sm));
                struct skynet_message message;
                message.source = 0;
                message.session = 0;
                message.data = sm;
                message.sz = sz | ((size_t)PTYPE_SOCKET << MESSAGE_TYPE_SHIFT);
                if (skynet_context_push((uint32_t)result->opaque, &message)) {
```

所以gate服务又会收到一条消息，这里就直接跳到`lfilter`(解包过程)了
```lua
    static int lfilter(lua_State *L)
        switch(message->type) {
            case SKYNET_SOCKET_TYPE_ACCEPT:
                lua_pushvalue(L, lua_upvalueindex(TYPE_OPEN));	// 将字符串 "open" 压栈
                // ignore listen id (message->id);
                lua_pushinteger(L, message->ud);				// 将ss->slot的数组下标压栈
                pushstring(L, buffer, size);					// 将字符串压栈，这里的字符串其实是 "192.168.1.1:1234" 这样的形式，就是ip和端口组成的字符串
                return 4;										// 返回 queue open id "192.168.1.1:1234"
```

这里`netpack.lfilter`会返回四个参数:queue open id "192.168.1.1:1234"

所以gate服务会执行 MSG.open(id,"192.168.1.1:1234")
```lua
    function MSG.open(fd, msg)
        if nodelay then
            socketdriver.nodelay(fd)	--会向底层的管道发送一个"T"的消息
        end
        connection[fd] = true
        handler.connect(fd, msg)
        function handler.connect(fd, addr)
            connection[fd] = c
            skynet.send(watchdog, "lua", "socket", "open", fd, addr)
```

watchdog服务会收到"socket", "open"的lua类型的消息
```lua
    function SOCKET.open(fd, addr)
        skynet.call(agent[fd], "lua", "start", { gate = gate, client = fd, watchdog = skynet.self() })
            skynet.call(gate, "lua", "forward", fd)
```

例子中的watchdog收到这个消息后，会将向agent发送一个start消息，agent收到后，会向gate服务发送一个forward消息，gate服务收到后
```lua
    function CMD.forward(source, fd, client, address)
        local c = assert(connection[fd])
        unforward(c)
        c.client = client or 0
        c.agent = address or source
        forwarding[c.agent] = c
        gateserver.openclient(fd)
            if connection[fd] then
                socketdriver.start(fd)
```

gate服务收到forward消息后，由前面的分析知道，它会向管道发送一个 'S'消息，所以`ctrl_cmd`会执行
```c
    static int ctrl_cmd(struct socket_server *ss, struct socket_message *result)
        switch (type) {
        case 'S':	//listen与accept后都会调用'S' 返回:SOCKET_ERROR、SOCKET_OPEN
            return start_socket(ss,(struct request_start *)buffer, result);
                if (s->type == SOCKET_TYPE_PACCEPT || s->type == SOCKET_TYPE_PLISTEN) {
                    if (sp_add(ss->event_fd, s->fd, s)) {
                        s->type = (s->type == SOCKET_TYPE_PACCEPT) ? SOCKET_TYPE_CONNECTED : SOCKET_TYPE_LISTEN;
                        s->opaque = request->opaque;
                        result->data = "start";
                        return SOCKET_OPEN;
```

此时此已连接描述符`s->type = SOCKET_TYPE_CONNECTED`，并且被加入到epoll事件管理器了

### 客户端发送数据过来
此时如果客户端发送数据过来，由于已加入到epoll事件管理器，所以有事件发生，`socket_server_poll`的`epoll_wait`返回。
```c
    ss->event_n = sp_wait(ss->event_fd, ss->ev, MAX_EVENT);		//等待有事情发生， 返回的是需要处理的事件个数
        struct socket *s = e->s;	// 取出自定义数据
        default:
            int type;
            if (s->protocol == PROTOCOL_TCP) {
                type = forward_message_tcp(ss, s, result);		// 正常的话返回 SOCKET_DATA
                    int sz = s->p.size;		// 先接收预设的字节数,即在new_fd函数中设置的 MIN_READ_BUFFER
                    int n = (int)read(s->fd, buffer, sz);
                    if (n == sz) {
                        s->p.size *= 2;
                    } else if (sz > MIN_READ_BUFFER && n*2 < sz) {
                        s->p.size /= 2;
                    }

                    result->opaque = s->opaque;
                    result->id = s->id;
                    result->ud = n;		// 收到了多少个字节数的网络数据
                    result->data = buffer;
                    return SOCKET_DATA;
            return type;
```

`socket_server_poll`返回`SOCKET_DATA`，看看`skynet_socket_poll`的处理
```c
    int skynet_socket_poll()
        case SOCKET_DATA:	//远端有数据发送过来
            forward_message(SKYNET_SOCKET_TYPE_DATA, false, &result);
                sm = (struct skynet_socket_message *)skynet_malloc(sz);
                sm->type = type;
                sm->id = result->id;
                sm->ud = result->ud;
                sm->buffer = result->data;
                struct skynet_message message;
                message.source = 0;
                message.session = 0;
                message.data = sm;
                message.sz = sz | ((size_t)PTYPE_SOCKET << MESSAGE_TYPE_SHIFT);
                
                if (skynet_context_push((uint32_t)result->opaque, &message)) {
```

老规矩，gate服务会收到一个`PTYPE_SOCKET`类型消息，直接转到lfilter
```c
    static int lfilter(lua_State *L)
        switch(message->type) {
            case SKYNET_SOCKET_TYPE_DATA:
                return filter_data(L, message->id, (uint8_t *)buffer, message->ud);
                static inline int filter_data(lua_State *L, int fd, uint8_t * buffer, int size)
                    int ret = filter_data_(L, fd, buffer, size);    // filter_data_就是解protobuf/sproto包的过程
```

这时如果protobuf/sproto包小于64个字节，正常的话会返回5个参数:queue 'data' 'fd' 'msg' 'sz'，所以会执行`MSG.data`
```c
    MSG.data = dispatch_msg
        if connection[fd] then
            handler.message(fd, msg, sz)
                local c = connection[fd]
                local agent = c.agent
                if agent then
                    skynet.redirect(agent, c.client, "client", 0, msg, sz)
                else
                    skynet.send(watchdog, "lua", "socket", "data", fd, netpack.tostring(msg, sz))
                end
```

一般来说服务端在底层调用过accept并顺利返回时，可以选择向gate服务发送一个`accept`或者`forward`消息

如果是前者，当有数据来时，gate服务会将数据转发到首次向它发'open'消息的服务，转发的消息类型为"lua"

如果是后者，当有数据来时，gate服务会将数据转发到最后一次向它发'forward'消息的服务，并且转发的消息类型为"client"

至此socket处理主要流程梳理完毕，细节处还是需要细细品尝。