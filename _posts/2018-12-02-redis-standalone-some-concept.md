---
layout: post
title: '单机环境下redis的一些理解'
tags: 数据库 redis
---
这里说下在单机环境下redis一些不那么引人注意的问题，知道这些背景对我们使用redis可能会更得心应手。

## 过期key
* 使用EXPIRE可以为key设置一个超时
* 使用PERSIST可以取消key的超时设置
* TTL命令查看key的还能存活多久，返回三种值：正整数，还有多久key会失效；-1，key没有设置超时；-2，key已经超时了或者key不存在
* 当且仅当key的值被完全替换，那么它的超时设置才会取消，这意味这HSET等部分更改值的操作不会取消超时设置
* EXPIRE、EXPIREAT命令设置在过去的时间会导致key被删除
* RENAME被替换的值如果存在，那么被替换的值会执行一个隐式的DEL操作
* 超时时间是按照机器时间戳的，不管redis运行还是没运行都会检查机器的时间戳
* 超时方式有两种：被动超时与主动超时，被动超时指被访问时才删除这个key；主动超指10秒钟一次从设置了超时的key中抽样20个，直到抽样出来超时的key低于一定比例
* 在复制的情况下，只在master里面进行超时，通过发送DEL操作给SLAVE通知SLAVE这个key超时了，如果master还没来得及针对超时key发送DEL命令给SLAVE，如果client访问SLAVE，SLAVE只会在访问时告诉client这个key超时了，但是不会实际删除这个key

## 内存淘汰策略
redis在设置了`maxmemory`的情况下，如果内存使用达到了`maxmemory`阀值，那么会使用LRU(Less Recent Used)淘汰策略，LRU有以下几种设置：
* noeviction，当达到内存阀值，直接返回一个错误给client
* allkeys-lru，范围为所有key，淘汰最少使用的key
* volatile-lru，范围为设置了超时的key，淘汰最少使用的key
* allkeys-random，范围为所有key，随机淘汰key
* volatile-random，范围为设置了超时的key，随机淘汰key
* volatile-ttl，范围为设置了超时的key，淘汰更短TTL的key

由于LRU是抽样检查的，不是全部数据的LRU，所以redis提供了一个设置样本的参数：maxmemory-samples，这个值越大就越耗费CPU和内存，权衡使用

由于LRU只是大致的LRU，所以redis 4.0提供了一个LFU(Least Frequent Used)的选项，这可以让比较少使用的key更准确的淘汰，而比较经常使用的key更高概率的保留下来，不过LFU仍是一个估算算法，LFU提供的选项：
* volatile-lfu，范围为设置了超时的key，淘汰最少使用的key
* allkeys-lfu，范围为所有key，淘汰最少使用的key

## redis的持久化
redis提供了两种持久化方式，AOF和RDB，一般可以两种方式都启用，AOF可以保证数据的完整性(虽然从AOF启动可能会很慢)，RDB相当于定时快照的方式，不过还是有些要点：
* AOF有个rewrite的过程，而RDB是将当前内存里面的数据完整的存到磁盘中，它们都会fork一个子进程来做这个操作，耗费CPU的同时，内存在最坏情况下可能会翻倍，甚至可能造成提供服务给client的主线程停止工作，这点开发和运维需要清楚，理论上AOF fork子进程操作的周期比RDB的更大，而且redis会在一定程度上保证AOF和RDB的fork子进程不会同时存在
* AOF rewrite会将当前内存中的数据以全新的方式存入磁盘
* 如果AOF和RDB同时启用了，那么redis在重启的时候会优先从AOF把数据加载进内存中，因为AOF意味这数据最完整
* AOF有一定几率数据不完整，一般情况下可能是文件最后一个命令不完整，这时可以简单删除最后一行，或者使用redis-check-aof来检查AOF文件
* RDB和AOF都有一定几率丢失数据，但是AOF可以选择每次接收到请求，都把数据写入到磁盘，这种情况下性能损失会很严重
* AOF可能会有很诡异的bug(虽然几率非常小)，但是RDB则不会有这种情况，只要RDB文件生成完成，那么这个RDB文件几乎不可能有问题

## 分布式锁
分布式锁是一种利用 setnx 命令进行的锁操作，是一种悲观锁，即首先认为这个操作会出现问题，必须要获取操作权限才能进行操作

分布式锁的实现除基本语义外还有有几个基本要点：
* 任何情况下不能造成死锁
* 释放锁的一方必须同时是加锁的一方才有效(可选)

上述第二个要点在单机环境下感觉其实不太重要，但是如果想要设计一个高可用的分布式锁就很重要了，所谓高可用的分布式锁即即使机器挂掉了，分布式锁的操作也不会受多少阻碍

高可用的分布式锁的要点：
* 任何情况下不能造成死锁
* 释放锁的一方必须同时是加锁的一方才有效（这里就是必选了）
* 多个独立的redis实例(必须是奇数个)
* 每次加锁都要大部分的redis实例同意才算成功
* 如果加锁不成功，那么需要给所有的redis实例发送删除key的命令，这里就体现释放锁的一方必须同时是加锁的一方才有效的重要性了

其实大部分情况下，单机环境的分布式锁就能满足要求了，最多是不可用而已；或者我们可以在集群环境下应用这个算法

详细的高可用的分布式算法见：[https://redis.io/topics/distlock#the-redlock-algorithm](https://redis.io/topics/distlock#the-redlock-algorithm)

## redis事务
redis的事务保证两点：
1. 事务中的几个操作不会存在中间执行状态，Isolated
2. 操作的原子性，即事务中的每个操作要么均成功，要么均失败，Atomic

redis事务的上述两个保证是在一个前提下的：如果命令本身是不合法的，但是redis只能在运行时才能检查出来，比如一个对string类型的key进行lpush操作，这种就是命令不合法但redis不能静态检查出来，只能在运行的时候才能知道出错了，在这种情况下，redis会跳过不合法的命令，而继续执行

而且redis不支持回滚操作，官方给出的解释是：redis的命令基本上不会存在执行失败的情况，因为没有关系型数据库那么繁琐的关联操作，如果出现了上述情况，是编程错误，编程错误的情况下就算这里堵住了这个口子，其他地方也会造成非常严重的后果

对于能够静态检查出的语法错误，比如参数不对，命令typo等情况，那么这些命令在queue的时候就会报错，执行`EXEC`命令时整个事务直接失败，已经queue成功的命令也不会执行

从上面保证来看，redis的事务只保证了ACID的A和I，由于redis是单线程的，C也可以算勉强保证（虽然官方文档并没有说明这点）；对于D来说，即使配置了append always也不能保证事务操作的持久化的，因为AOF文件可能写到一半就断电了

## 乐观锁
所谓乐观锁，是一种执行时的检查机制，基本原则是：先入为主的相信某些操作不会有问题或冲突，在提交时检查一下是否有冲突或产生问题，如果是则不执行任何操作，如果否，则正常执行操作，当然这要保证检查和执行操作本身就是原子性，当然对于redis来说，它本来就是单线程的，保证这点不难

redis对于乐观锁的支持在使用层面来说很简单粗爆：
1. 针对要检查的一个或多个key使用WATCH命令
2. 使用MULTI命令启动启动一个事务
3. queue个redis命令
4. 执行EXEC提交事务

在执行EXEC命令之前，redis会检查WATCH命令的监视的key是否有过变化，如果确实有变化，则DISCARD这个事务，这时候事务的发起方只需要简单的重试就可以了

不过对于WATCH命令还有些注意点：
* WATCH命令的作用域是一个client与redis server的TCP连接，当连接断开，WATCH命令WATCH的所有key就变成UNWATCHED了
* 只有改变了key的值才会WATCH失败，即如果SET一个String类型的key相同的值则不会视为已经改变了，说明redis内部对于WATCH命令的实现，可能是值与域长度比较的
* 当执行了EXEC后，不论执行失败还是成功，所有key都会变成UNWATCHED

## 缓存穿透、击穿与雪崩
缓存穿透：是指查询一个根本不存在的数据，攻击者可能会利用这点进行攻击，解决方法是对不存在的key也设置缓存

缓存雪崩：多个key集中失效与redis宕机导致的在非常短的时间内对多个缓存进行重建，甚至同一个缓存会重建很多次，本质上和缓存击穿造成的后果是一样的

缓存击穿：大并发情况下，如果key失效了，那么可能会有非常多的请求同时请求到缓存后面的数据库了，这里介绍一个使用redis锁机制来避免这种情况的方法：
1. 查询缓存，未命中
2. 查询数据库
3. 将数据库的结果存到redis，但是数据域的基础上人工写入一个加上随机几秒的超时时间
4. 从redis取出缓存时，首先判断人工设置的超时时间看看是不是假超时了，如果是，则加上一个redis锁，在加锁的情况下查询数据库，如果加锁失败，则直接返回redis中的数据

## redis的大key
redis大key情况分为两种：
1. 简单key对应的value过大
2. key对应的元素过多

在存在大key的情况下，由于redis是单线程，可能会造成对这个key的操作时间时间过长而阻塞其他操作，所以这种情况尽量避免

对于第1种情况，可以使用hash类型的key对value进行拆分

对于第2种情况，使用分批次读取的方式，比如hsan、lrange等，某些特殊情况可能还是需要进行key的拆分，然后利用二级索引来操作