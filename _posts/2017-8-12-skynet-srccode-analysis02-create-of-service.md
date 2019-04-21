---
layout: post
title: 'skynet源码分析02-服务创建流程'
tags: skynet
---
skynet中的服务可以分为两大类:C服务于lua服务。但是创建服务都是通过C模块来进行的，通常来说创建一个C服务就是通过加载一个so模块进行的，一个lua服务是通过 snlua.so 这个模块来启动的。

本文配合[2016年下旬最新版skynet源码注释](https://github.com/nullscc/skynet_with_note)更佳

## C服务的创建
skynet本身提供的并且在用的C服务只有:harbor、logger(snlua在我看来只是一个雀巢)

## harbor服务的创建
一个skynet进程的 harbor 服务的启动是在 cdummy/cslave(单节点模式是cdummy，多节点是cslave)这两个服务中的一个来启动的,harbor服务启动代码为:`skynet.launch("harbor", harbor_id, skynet.self())`。
下面就就具体代码来分析下:
```lua
    skynet.launch("harbor", harbor_id, skynet.self())
        local addr = c.command("LAUNCH", table.concat({...}," "))       -- 调用 lualib-src 下的 lua-skynet 模块
            static int lcommand(lua_State *L)
                skynet_command(context, cmd, parm);
                    struct command_func * method = &cmd_funcs[0];
                    return method->func(context, param);
                        static const char * cmd_launch(struct skynet_context * context, const char * param) {	
                            // param为一串字符串(服务名和参数组成，以空格分开)
                            struct skynet_context * inst = skynet_context_new(mod,args);
```

## logger服务的创建
logger服务的启动很简单: `struct skynet_context *ctx = skynet_context_new(config->logservice, config->logger);`

## lua服务的创建
我们知道一个lua服务一般是通过 `skynet.newservice` 或者 `skynet.uniqueservice` 来启动的，暂且只看 skynet.newservice 的过程，`skynet.uniqueservice`的过程等了解完 master/slave 模式再来看也不迟。
```lua
    function skynet.newservice(name, ...)
        return skynet.call(".launcher", "lua" , "LAUNCH", "snlua", name, ...)   -- launcher服务 就是 launcher.lua
            function command.LAUNCH(_, service, ...)	-- _为发送此消息的源地址 service一般为启动lua服务的中介 一般为 snlua
                launch_service(service, ...)    -- service 为服务模块名，... 为创建服务时传递进去的参数
                    local param = table.concat({...}, " ")
                    local inst = skynet.launch(service, param)	-- service 为要启动的服务名 param 为以空格分开的参数
                        local addr = c.command("LAUNCH", table.concat({...}," "))       -- 调用 lualib-src 下的 lua-skynet 模块
                            static int lcommand(lua_State *L)
                                skynet_command(context, cmd, parm);
                                    struct command_func * method = &cmd_funcs[0];
                                    return method->func(context, param);
                                        static const char * cmd_launch(struct skynet_context * context, const char * param) {	// param为一串字符串(服务名和参数组成，以空格分开)
                                            struct skynet_context * inst = skynet_context_new(mod,args);
```

## skynet 服务创建的核心
从上面我们可以看到不管是lua服务还是C服务，最终都是通过同一个接口:`skynet_context_new`来创建的。所以要弄清楚它的工作，而要弄清楚它的工作只需要知道 logger服务(C服务)和 bootstrap服务(lua服务)是怎么起来的就行了。

### logger服务怎么起来的
logger服务是第一个启动的C服务
```lua
        struct skynet_context *ctx = skynet_context_new(config->logservice, config->logger);    // config.logservice 为 "logger" config->logger为要写入的log的路径(可无)
            struct skynet_module * mod = skynet_module_query(name);     // 查找C模块(如果没找到就找到对应的so文件，打开它，并创建文件句柄以供使用)并返回一个 struct skynet_module 结构
                void *inst = skynet_module_instance_create(mod);        // 调用C模块的 xxx_create 函数
                    struct logger * logger_create(void)
                struct skynet_context * ctx = skynet_malloc(sizeof(*ctx));
                ctx->mod = mod;  //struct skynet_module
                ctx->instance = inst;	//动态库特有的结构体指针
                ctx->ref = 2;
                ctx->cb = NULL;
                ctx->cb_ud = NULL;
                ctx->session_id = 0;
                ctx->logfile = NULL;

                ctx->init = false;
                ctx->endless = false;
                // Should set to 0 first to avoid skynet_handle_retireall get an uninitialized handle
                ctx->handle = 0;	

                ctx->handle = skynet_handle_register(ctx);                                  // 得到服务的地址
                struct message_queue * queue = ctx->queue = skynet_mq_create(ctx->handle);  // 创建服务的消息队列
                int r = skynet_module_instance_init(mod, inst, ctx, param);                 // 调用 xxx_init
                    int logger_init(struct logger * inst, struct skynet_context *ctx, const char * parm)
                        if (parm) {
                            inst->handle = fopen(parm,"w");
                        else {
                            inst->handle = stdout;
                        skynet_callback(ctx, inst, logger_cb);          // 注册 logger 服务的消息处理函数
                        skynet_command(ctx, "REG", ".logger");          // 为 logger 服务注册一个本节点有效的字符串地址
                skynet_globalmq_push(queue);                            // 将服务的消息队列放入全局的消息队列尾
```

### bootstrap服务是怎么起来的
```lua
    bootstrap(ctx, config->bootstrap);
        struct skynet_context *ctx = skynet_context_new(name, args); // name为 "snlua" args为 "bootstrap"
            ... //中间省略，和上面的 logger服务怎么起来的 是一样的
            int r = skynet_module_instance_init(mod, inst, ctx, param);                 // 调用 xxx_init
                int snlua_init(struct snlua *l, struct skynet_context *ctx, const char * args)
                    skynet_callback(ctx, l , launch_cb);					// 注册消息处理函数
                    skynet_send(ctx, 0, handle_id, PTYPE_TAG_DONTCOPY,0, tmp, sz);	// 发消息给自己 以便加载相应的服务模块，这里tmp为 "bootstrap"

    // 从上面我们知道 创建的这个服务的初始消息处理函数为 launch_cb，当worker线程调度到这个消息时便会执行
    static int launch_cb(struct skynet_context * context, void *ud, int type, int session, uint32_t source , const void * msg, size_t sz)
        skynet_callback(context, NULL, NULL);					// 将消息处理函数置空，以免别的消息发过来
            int err = init_cb(l, context, msg, sz);					// 消息处理:通过 loader.lua 加载 lua 代码块
                const char * loader = optstring(ctx, "lualoader", "./lualib/loader.lua");
                int r = luaL_loadfile(L,loader);	// 把 loader.lua 作为一个lua函数压到栈顶
                r = lua_pcall(L,1,0,1);				// 调用 loader.lua 生成的代码块
```

## 总结
从上面分析可以看出，服务创建都是通过 `skynet_context_new`来进行的，`skynet_context_new` 必然会创建一个C模块(或者从已有C模块中找到已创建的模块)。

C服务的创建可以简单的说是加载这个模块，执行一些初始化工作，然后注册消息处理函数就完毕了。

lua服务的创建则有不同，它是通过snlua来创建的，每个lua服务的第一消息必然是发给自己的，而且这个消息的处理函数是 snlua 模块在执行 snlua_init 时注册的，当一个lua服务收到第一个消息时，是通过 loader.lua 来加载自己，并重新注册处理函数的

注:并不存在什么 snlua 服务，它只是创建lua服务的一个中间过程而已。