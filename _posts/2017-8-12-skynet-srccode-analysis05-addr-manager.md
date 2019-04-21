---
layout: post
title: 'skynet源码分析05-地址管理'
tags: skynet
---
skynet消息发送是通过地址来投递的。

地址按形式分:数字地址，与字符串地址

地址按作用范围分:单节点有效("."开头)，整个skynet网络有效(字母开头)，这里只说单节点有效的地址，多节点有效的留到后面的 master/slave 比较好点

字符串地址是不允许冒号开头的

在[skynet源码分析02-服务创建流程](skynet_srccode_analysis02_create_of_service.html)中已经提过消息创建的时候最终都会通过`skynet_context_new`进行创建服务的结构体

本文配合[2016年下旬最新版skynet源码注释](https://github.com/nullscc/skynet_with_note)更佳

## 数字地址的来源
```lua
    struct skynet_context * inst = skynet_context_new(mod,args);
        ctx->handle = skynet_handle_register(ctx); // handle就是数字地址
            for (;;) {
                int i;
                for (i=0;i<s->slot_size;i++) {
                    uint32_t handle = (i+s->handle_index) & HANDLE_MASK; //将高八位置为0
                    int hash = handle & (s->slot_size-1);	//从1开始增长，到0终止，如果hash为0了，说明slot_size已经用尽了
                    if (s->slot[hash] == NULL) {
                        s->slot[hash] = ctx;
                        s->handle_index = handle + 1;

                        rwlock_wunlock(&s->lock);

                        handle |= s->harbor; 				//将服务的高八位置为 harbor 编号
                        return handle;
                    }
                }

                //如果不够分配新的slot，成倍扩充，将老的slot复制过来
                assert((s->slot_size*2 - 1) <= HANDLE_MASK);	// 一个节点最多可拥有的服务数为: 0xffffff
                struct skynet_context ** new_slot = skynet_malloc(s->slot_size * 2 * sizeof(struct skynet_context *));
                memset(new_slot, 0, s->slot_size * 2 * sizeof(struct skynet_context *));
                for (i=0;i<s->slot_size;i++) {
                    int hash = skynet_context_handle(s->slot[i]) & (s->slot_size * 2 - 1);
                    //复制时hash值为:1-> s->slot_size， 即将老的全部复制到新的数组的起始处
                    assert(new_slot[hash] == NULL);
                    new_slot[hash] = s->slot[i];
                }
                skynet_free(s->slot);	// 复制完后将老的slot删除
                s->slot = new_slot;
                s->slot_size *= 2;
            }
```

从上面可以看到所有的地址都挂在一个`struct handle_storage`下进行统一管理，数字地址的高八位为节点id，低24位为此节点内的地址，刚开始slot_size只有4个，发现不够了就成倍扩充(将老的拷贝到新的，然后再丢弃老的)。

## 字符串地址的来源
skynet框架上层提供两个接口来注册一个字符串地址:`skynet.register`、`skynet.name`
```lua
    skynet.register(name)
        -- 如果以"."开头,则注册一个本节点内有效的字符串地址
        c.command("REG", name)
            { "REG", cmd_reg },
                static const char * cmd_reg(struct skynet_context * context, const char * param)
                    return skynet_handle_namehandle(context->handle, param + 1); //注册本节点有效的全局名字
                            const char * ret = _insert_name(H, name, handle);
                                {
                                    int begin = 0;
                                    int end = s->name_count - 1;
                                    while (begin<=end) {
                                        int mid = (begin+end)/2;
                                        struct handle_name *n = &s->name[mid];
                                        int c = strcmp(n->name, name);
                                        if (c==0) {
                                            return NULL;
                                        }
                                        if (c<0) {
                                            begin = mid + 1;
                                        } else {
                                            end = mid - 1;
                                        }
                                    }
                                    char * result = skynet_strdup(name);

                                    _insert_name_before(s, result, handle, begin);

                                    return result;
                                }

    skynet.name(name, handle)
        -- 如果以"."开头,则注册一个本节点内有效的字符串地址
        c.command("NAME", name .. " " .. skynet.address(handle)) -- skynet.address返回地址的字符串形式,以冒号开头
            static int lcommand(lua_State *L)
                result = skynet_command(context, cmd, parm);
                    struct command_func * method = &amp;cmd_funcs[0];
                        return method->func(context, param);
                            { "NAME", cmd_name },
                                static const char * cmd_name(struct skynet_context * context, const char * param)
                                    sscanf(param,"%s %s",name,handle);
                                    if (name[0] == '.') {
                                        //注册全局名字
                                        return skynet_handle_namehandle(handle_id, name + 1);
```

从上面代码来看，所有字符串地址储存在一个数组里面，当要注册新的字符串地址时，先折半查找到要插入的点，并且将入点之后的所有元素后移，然后将在对应的地方插入。
需要注意的是:如果你仅仅需要注册一个本节点有效的地址的话，最好在字符串前加上一个 '.',不然就会注册一个全skynet网络有效的地址。

## 查询服务地址:
```lua
    skynet.localname
        local addr = c.command("QUERY", name)   --返回一个带冒号的16进制的数字的字符串
            static int lcommand(lua_State *L)
                result = skynet_command(context, cmd, parm);
                    struct command_func * method = &amp;cmd_funcs[0];
                    return method->func(context, param);
                        static const char * cmd_query(struct skynet_context * context, const char * param)
                                if (param[0] == '.') {
                                    uint32_t handle = skynet_handle_findname(param+1);  //折半查找全局名字服务对应的地址，与 skynet_handle_namehandle 差不多
                                        
    function skynet.self()
        self_handle = string_to_handle(c.command("REG"))
            { "REG", cmd_reg },
                static const char * cmd_reg(struct skynet_context * context, const char * param)
                        if (param == NULL || param[0] == '\0') {    //如果不带参数，返回自身的地址
                            sprintf(context->result, ":%x", context->handle);
                            return context->result;

    // 我们知道调用 skynet.send/skynet.call发送消息时，第一个参数是同时支持数字地址与字符串地址的，那么是怎么实现的呢?
    function skynet.send(addr, typename, ...)
        return c.send(addr, p.id, 0 , p.pack(...))
            static int lsend(lua_State *L)
                if (dest == 0) { 			//如果是字符串
                    if (lua_type(L,1) == LUA_TNUMBER) {
                        return luaL_error(L, "Invalid service address 0");
                    }
                    dest_string = get_dest_string(L, 1);	//得到字符串地址
                }
                if (dest_string) {	//如果是字符串地址
                    //skynet_sendname 最终还是会调用 skynet_send
                    session = skynet_sendname(context, 0, dest_string, type, session , msg, len);
                        } else if (addr[0] == '.') {	//本节点有效的字符串地址
                            des = skynet_handle_findname(addr + 1);
                } else {	//如果是数字地址
                    session = skynet_send(context, 0, dest, type, session , msg, len);	
```

可见查找字符串地址时，最终都会调用`skynet_handle_findname`进行折半查找(或者直接由skynet服务的结构体得到)