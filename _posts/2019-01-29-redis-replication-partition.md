---
layout: post
title: 'redis复制、分区、哨兵与分布式简单理解'
tags: 数据库 redis
---
这篇文章主要说说在复制、分区的情况下，redis的能力与限制，开发业务逻辑时脑海里要有相应概念，根据这些概念做出一个相应的权衡选择，才算是一个合格的程序员

这里不会太过详细的介绍一些基本概念，只是从开发业务的角度来看看redis有什么能力和限制，如果想要非常详细的了解，可以移步[https://redis.io/documentation](https://redis.io/documentation)，博客很多文章都是这个原则

## 分区
分区依据：
* range，根据key的范围
* hash，根据key的hash值，可以带上取模运算
* 一致性hash

既然说到一致性hash，那么就简单说下一致性hash算法的原理：
* 一个圆环，圆环上有2^32个点
* 服务器的分布会对应圆环上一个或多个点，多个点就是虚拟节点，一个或多个虚拟节点对应至少对应一个实际的物理节点，虚拟节点能有效避免数据的倾斜分布
* 数据对应的key也会对应圆环上一个或多个点
* 当访问key对应的数据时，顺时针旋转遇到的第一个数据库节点就是key要存放的节点或者要请求数据的节点

分区访问形式：
* 客户端分区，redis server不需要知道哪些key被分配到那个节点上了，一切通过客户端的操作来进行key的映射，这种方式当然局限性比较大
* 通过代理访问，client与server访问不直接连接，而是通过一个proxy，所有请求经过proxy，然后proxy再向真正的server请求，server返回的数据再通过proxy返回
* 查询路由，通过某种形式将client的访问动态路由到节点上

redis可以通过以下形式来实现分区：
* redis集群
* Twemproxy，Twitter开发的
* 支持一致性hash的客户端

缺点：
* 事务操作不能在分区上进行
* 不能操作多个key

上述两个缺点不是一定的，比如集群的情况下可以通过hash tag的形式来访问多个key或事务操作，不过可能会造成数据的倾斜分布，形成热点

## 复制
* redis下主要是异步复制，看官方文档几乎已经把同步复制给忽略了，可以看到redis主要想让用户将redis视为一个AP系统
* master关闭持久化而slave开启持久化，这种情况下需要避免master的自动重启，因为如果master挂了，并且其没有启用持久化，那么当master自动restart，slave的所有数据也会丢失
* 单纯的手动复制是不支持自动fail over的
* 每个节点都持有一个复制ID与偏移量，redis根据这个唯一的复制ID与对应的偏移量来进行流式复制，但是如果某些情况下redis认为数据已经对不上了，可能会选择进行全量同步
* 新master可能同时持有新的复制ID与老的复制ID，因为这样在slave刚刚成为master的那一小段时间避免其他slave进行全量同步
* 多个slave的同步可能被master的同一个线程处理，通过这种方式可以优化性能
* 全量无盘复制，一般情况下，全量复制需要一个RDB文件作为中转，但是也可以选用直接启用子进程从内存中发送这个RDB
* 复制的slave默认情况下是read-only的，但是也可以对slave配置可写，但是slave配置可写的情况这些数据只会存在与于slave上，那么什么情况下可以启用slave写呢？一般来说，如果有临时计算需要用到redis的功能，但是也不在乎这些临时计算出来的值是否能长久保存的情况可以启用slave写
* 4.0版本之前，启用了写功能的slave，不兼容EXPIRE命令，在这种情况下，会看不见key的值，但是仍然占内存
* 从redis 2.8开始，可以配置当且仅当至少有N个slave连接到master才能接受写
* 在复制的情况下，只在master里面进行超时，通过发送DEL操作给SLAVE通知SLAVE这个key超时了，如果master还没来得及针对超时key发送DEL命令给SLAVE，如果client访问SLAVE，SLAVE只会在访问时告诉client这个key超时了，但是不会实际删除这个key
* 在启用AOF的情况下重启不太可能进行部分偏移复制，不过可以通过先关闭AOF，开启RDB然后再重启，这样就可以实现部分偏移复制啦

我们知道redis主要是异步复制，异步复制代表可能会丢失数据，调整异步复制造成的数据丢失损失量的原理和方式：
* slave每秒都会ping master，以便让master知晓已复制流的数量
* master会记住最后一次slave ping的时间
* 所以用户可以配置两个参数来降低损失：`min-slaves-to-write <number of slaves>`和`min-slaves-max-lag <number of seconds>`

在docker环境下配置复制需要进行特殊设置:
```bash
slave-announce-ip 5.5.5.5
slave-announce-port 1234
```
因为docker环境下，redis拿到的ip不是自己的真实ip(除非启用host网络模式)

## 哨兵
默认情况下，如果只启用复制，那么当master crash了，需要手动将其中的一个slave给配置成master，那么有没有可能这个过程是自动的，这就是复制情况下的高可用了，redis的哨兵就是为了这种情况服务的

* 需要至少3个并且总数为奇数个redis实例才能启用哨兵，因为哨兵的自动fail over机制需要一个少数服从多数的机制
* 默认使用异步复制
* 客户端需要支持哨兵
* master关闭持久化而slave开启持久化，这种情况下也需要避免master的自动重启，因为哨兵可能来不及检测到master crash就重启完成了，这时slave的数据可能会全部丢失
* 采用gossip协议
* 只需要配置一个master就可以了，其他slave会自动发现，并且哨兵的配置文件会被哨兵模式更新，自动发现与哨兵的配置文件更新是最终一致性的
* 客户端在选举的master fail over的过程中可能会仍然向老的master写数据，这样会造成数据丢失，所以可以配置这两个参数来降低这个数据丢失的损失：`min-slaves-to-write 1`, `min-slaves-max-lag 10`
* 当配置的quorum个哨兵认为一个master已经不可用了(从SDOWN状态变为ODOWN状态)，那么其中一个哨兵被选择成为leader来主导选举的过程
* 可以配置slave被选举成master的优先级
* 哨兵使用Pub/Sub的方式来相互发现

## 集群
redis集群是一个无proxy访问的可启用分区可启用复制的模式，并且当故障发生时，可以通过一定机制来让系统继续正常运作(只有很短一段时间的不可用)
* 用少量的数据的丢失可能性来保证高性能
* 客户端需要支持集群，客户端的内存里面会维护一个集群的map表
* 集群的通信端口在常规端口上面+10000
* 集群不使用一致性hash，而是可以使用最多16384个hash slot，key的选择通过`HASH_SLOT = CRC16(key) mod 16384`，意味这可能发生分布倾斜的情况，不过可以redis支持手动调整slot，所以这种情况可以通过手动分配slot来调整
* 
* redis集群是支持事务或多个key的操作的，不过需要通过hash tag的方式来仅仅访问其中一个hash slot，但是如果被操作的节点在进行resharding操作，那么多key和事务操作在这个期间可能不支持的，因为在resharding的过程中，被操作的多个key可能不在同一个节点，在这种情况下，client会收到一个`TRYAGAIN`错误，客户端收到这个错误可以在几秒后进行重试或者直接返回一个错误
* 在docker中部署需要使用host网络模式
* `cluster-node-timeout`控制如果一个master node和集群失去联系多久就会导致cluster启动一个failover
* 要让集群启动，必须最少有三个master节点
* 集群中的每个实例都有个永久不变的集群ID
* 支持cluster的client启动时一般只需要知道部分节点，因为cluster会通过`-MOVED`回复其他的节点给client
* 手动failover会有更少的几率丢失数据，因为这一过程可以被程序控制，发生一些临界情况的几率小很多
* 可以通过`CLUSTER REPLICATE <master-node-id>`命令手动将slave的master换成另外一个分区的master，不过如果redis cluster检测到有master没有节点并且有些master有多个slave，那么会自动将多的slave移到master下([Replicas migration](https://redis.io/topics/cluster-tutorial#replicas-migration))，通过这种方式进一步提高redis集群的高可用性
* 集群只能用db 0
* 和哨兵一样，使用gossip协议来沟通集群的变化，同时也因为gossip协议，即使集群中有非常多的节点，那么心跳包的频率也不会太高
* cluster不使用proxy，它通过重定向的方式告诉client直接连接到正确的节点

NODE_TIMEOUT意义：
* 如果master至少有NODE_TIMEOUT时间与集群的绝大部分节点失去联系了，那么就会进行自动fail over
* 如果失联的master超过了NODE_TIMEOUT，那么它会拒绝写入写操作，所以NODE_TIMEOUT是最大的写丢失的时间

cluster client尝试接收全部slot地址映射关系的时机：
* 启动时
* 当接收到一个MOVED重定向命令

redis在增加或删除节点时，slot会在节点之间迁移，那么在迁移过程中怎么回事儿呢(假设是从A迁移到B)？
* 一个key一个key的进行原子性的迁移，所以如果key很大的话，那么可能会阻塞很久，所以大key就要注意啦
* client每次请求都先请求A，如果A发现存在请求的key，那么直接返回相应的值
* 如果A发现不存在请求的key，那么给client一个ASK重定向到B
* 然后client会先发送ASKING命令(带上请求)给B，告诉B，无论如何这一个请求都得服务于我，不能推磨
* B收到带请求的ASKING命令，就直接返回操作结果给client

ASK和MOVED重定向的区别？
* MOVED是告诉client某个slot永久换节点了，这时候client需要更新自己内存中的cluster节点的map表
* ASK告诉client，这次请求去另外一个节点访问吧，不过下次请求你还是可以来访问我
* 如果client发送了一个ASKING请求，那么收到这个请求的节点就必须返回对相应key的相应，而不管这个key对应的slot现在是不是在这个节点上