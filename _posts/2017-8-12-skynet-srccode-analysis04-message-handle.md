---
layout: post
title: 'skynet源码分析04-消息处理(协程)'
tags: skynet
---
skynet的消息处理是每个消息过来都对应一个协程进行处理。以前分析过一次，并简单记录过一些东西，无奈这中间的处理有点绕，再猛然一看有点懵。有可能是自己没有完全理解吧。今天又分析了一遍，索性把以前写的自己都看不懂的东西都删了，这里再详细记录下。

这里只是分析，代码片段在[skynet消息处理相关代码(协程)](http://nulls.cc/2017/08/12/skynet_message_handle_maincode/)，打开两个网页对照着看可能比较好,本文中提到的所有函数都可以找到。

这里主要分析消息处理框架的主要流程。当然分析这些之前最好知道消息是怎么注册的，以及消息到来时服务怎么调用消息处理函数的。

本文配合[2016年下旬最新版skynet源码注释](https://github.com/nullscc/skynet_with_note)更佳

## 约定说明
由于这块的确比较绕，为了叙述清楚，这里约定:
* 消息的发送方为服务A，全文中用A来代替
* 消息的接受方为服务B，全文中用B来代替
* 将服务的一次消息处理分为两个协程，主要逻辑处理的协程称为主协程，消息处理函数(即在co_create函数中执行f(...))时的协程为次协程
* 这里的主协程与次协程是相对于消息来说的，可能对于同一个服务的消息A来说是主协程，但是对于消息B来说是次协程

## 简单说说怎么注册消息处理函数的
1. 服务调用 skynet.start 函数
2. skynet.start 会调用 c.callback(skynet.dispatch_message) ，传递给 c.callback 的参数就是服务的消息处理函数
3. 所以 skynet.dispatch_message 就是lua服务的消息处理函数，skynet.dispatch_message 会调用 raw_dispatch_message 并把自己的所有参数传递给 raw_dispatch_message
4. 所以当服务收到外来服务的消息时，消息的处理函数可以认为就是: raw_dispatch_message

## co_create工作机制
在说消息处理之前有必要说一下co_create(第一次看到这里可能会看不懂，看完全篇后可以再回来看一遍~)，这个函数设计的很巧妙(可能我少见多怪吧)，它将创建新的协程与复用老的协程放在同一个函数中。
* **创建新的协程**
1. 如果`table.remove(coroutine_pool)`返回的是nil，说明已经协程池中已经没有协程了，需要新建
2. 用`coroutine.create`创建一个新的协程，参数为一个函数
3. 协程的行为看起来很简单:执行 co_create 函数的参数(参数为函数)f，f的参数(...)为  raw_dispatch_message 函数的第35行的 suspend 中的 coroutine_resume的除co外的所有参数
4. 当f(...) 返回时，将协程回收，以便下次利用。
5. 最后执行 coroutine_yield "EXIT" 让出此次执行，这个协程就等待着复用，至此新建协程的的流程就完成了。接下来的两行就是复用协程的代码。

* **复用协程**
1. 如果`table.remove(coroutine_pool)`返回的不是nil，说明已经协程池中还有协程可以复用。
2. 这时函数跑到else分支，执行: coroutine_resume(co, f)，由于可复用协程此时等待在 `coroutine_yield "EXIT"`，所以函数代码会执行到 co_create 函数的第9行(注意到这里其实已经有两个协程了，姑且称在raw_dispatch_message中的协程为主协程，称在co_create 函数的第9行的协程为次协程)
3. 跑到 co_create 函数的第9行后 coroutine_yield 就此返回，f就是 coroutine_resume(co, f)的f参数，即 co_create 的参数
4. 接下来函数执行到 f(coroutine_yield()) 又让出执行权，这时代码执行到主协程，即 co_create 函数返回到 raw_dispatch_message
5. 当 raw_dispatch_message 执行到 第35行的 suspend 中的 coroutine_resume 后，代码又切换到次协程
6. 切换到次协程后 f(coroutine_yield()) 中的 coroutine_yield 函数返回，返回的值(就是 coroutine_resume 除了co外的所有参数)作为f的参数，即作为 p.dispatch 的参数
7. f(coroutine_yield()) 执行完后，最终会回收此协程，并调用 coroutine_yield "EXIT"等待下一次的复用

不管是新建协程还是复用协程的流程，调用 `coroutine_yield "EXIT"`后都会返回到 raw_dispatch_message 的第35行，执行 suspend 后此次流程才算真正的完成

## A调用skynet.send发送消息给B
调用skynet.send发送消息的流程相对来说比较简单。

1. A调用skynet.send发送消息给B
2. A就会调用`c.send(addr, p.id, 0 , p.pack(...))`这个核心接口来将消息压入消息队列。第三个参数为session，这里为0是因为skynet.send不需要返回值，所以发送完后这里流程就完全结束了。A的此次发送消息任务已经完全完成了。
3. B从消息队列中取出消息，B找到自身的消息处理函数: raw_dispatch_message ，调用 raw_dispatch_message
4. raw_dispatch_message 发现 prototype 不是1(不是别的服务的返回值)。走到else分支:找到服务注册的此类消息的 dispatch 函数，然后调用 co_create 得到一个协程 co。
5. 流程走到 raw_dispatch_message 的suspend 的 coroutine_resume 跑到 co_create中的主协程中执行消息处理函数(具体怎么执行的见上面的 co_create 工作机制)
6. co_create 中的次协程执行完(即消息处理函数执行完后)， coroutine_resume 返回的参数(true "EXIT")给 suspend 作为参数，此次服务A与B的交互完成

## A调用skynet.call发送消息给B
调用skynet.call与skynet.send虽然表现上只有一个返回值的区别，但是内在实现多饶了好几个圈~

1. A的主协程调用skynet.call发送消息给B:调用`c.send(addr, p.id , nil , p.pack(...))`将消息压入消息队列，第三个参数为nil表示由框架自动分配一个session，这个session用来当B返回消息到A时找到对应的协程
2. 跑到skynet.call函数的第8行，执行 yield_call，到 coroutine_yield("CALL", session) 处让出执行
3. coroutine_yield("CALL", session) 会让出到A的调用它的消息的次协程的 raw_dispatch_message 的第35行(由于skynet中一切皆服务，所以调用skynet.call的函数本身就是一个消息处理函数的一部分，所以总能让出到suspend)，
4. A这时候执行到suspend，所以执行到`if command == "CALL" then`， 然后执行`session_id_coroutine[param] = co`仅仅记录下session对应的协程，以便将来B返回时找到对应的协程恢复执行，到这里A发送消息完成
5. B的主协程从消息队列中取出消息，B找到自身的消息处理函数: raw_dispatch_message ，调用 raw_dispatch_message
6. B的 raw_dispatch_message 发现 prototype 不是1(不是别的服务的返回值)。走到else分支:找到服务注册的此类消息的 dispatch 函数，然后调用 co_create 得到一个协程 co。
7. B的流程走到 raw_dispatch_message 的suspend 的 coroutine_resume 跑到 co_create中的主协程中执行消息处理函数(具体怎么执行的见上面的 co_create 工作机制)
8. B的co_create 中的次协程执行完(即消息处理函数执行完)之前， 会调用skynet.ret函数将A需要的返回值打包进去，然后次协程执行到 return coroutine_yield("RETURN", msg, sz)让出执行权到B的主协程
9. B的主协程得到执行权后，从 raw_dispatch_message 的第35行即 `suspend(co, coroutine_resume(co, session,source, p.unpack(msg,sz)))`处继续执行:`suspend(co, true, "RETURN", msg, sz)`
10. B的主协程会执行到suspend函数的 `elseif command == "RETURN" then`
11. 如果A相对于B来说不是dead_service，那么调用 c.send 将包含返回值的消息压入到消息队列
12. 这时候B执行到其主协程的 suspend 的39行，即 `return suspend(co, coroutine_resume(co, ret))` 唤醒 `skynet.ret`中的 `return coroutine_yield("RETURN", msg, sz)`的执行
13. 至此B的次协程中的 f(...)函数才执行完成，然后是处理函数清空，协程回收，最后调用 `coroutine_yield "EXIT"`结束此次B的消息处理函数的执行(仅仅是次协程执行完成)
14. B调用`coroutine_yield "EXIT"`后会让出到B的主协程，即到 suspend 函数的 `elseif command == "RETURN" then`的分支的 `return suspend(co, coroutine_resume(co, ret))`处
15. 然后B得主协程的suspend函数执行到`elseif command == "EXIT" then`分支做些清理工作，然后完成B主协程的执行。至此B的消息处理才算真正完成了。
16. 然后A的主协程收到B的返回消息，执行 raw_dispatch_message 函数，由于是返回消息类型，所以执行`if prototype == 1 then`，正常的话会执行else分支，即会执行 `suspend(co, coroutine_resume(co, true, msg, sz))`
17. A的主协程执行`suspend(co, coroutine_resume(co, true, msg, sz))`后，会唤醒 skynet.call 中的 yield_call，即`local succ, msg, sz = coroutine_yield("CALL", session)`的执行，这时A的主协程就收到了B的返回消息，并从skynet.call返回。此次调用完成。
18. 巧妙的是调用A进行skynet.call调用的次协程如果让出的话，还是会让出到suspend函数，即 raw_dispatch_message 的第11行处，即`suspend(co, coroutine_resume(co, true, msg, sz))`，这样就接续调用skynet.call的消息处理函数的流程了。