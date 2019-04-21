---
layout: post
title: 'rabbitmq基本概念'
tags: 消息队列
---
rabbitmq是相对较传统的消息队列，主要特色是Erlang实现的AMQP(Advanced Message Queue Protocol)，带消息确认机制的消息队列。相对于kafka，理解和维护起来比较容易，实时性较高，不过由于扩展不方便，所以吞吐量没有那么好。

## 基本组成部分
* producer
* connection
* channel
* exchange
* queue
* consumer

一个消息从producer发送到consumer的过程为: 
1. producer 与 rabbitmq server(实现了channel, exchange, queue等) 建立一条tcp连接，在此连接的基础上创建一个多路channel
2. producer 发送消息到channel，rabbitmq server 接收到这个消息，将它放到exchange里面去
3. exchange 根据设置的参数投递到不同的queue上
4. rabbitmq server 将 queue 里面的消息 push 到 consumer

上面的组成部分其实是AMQP的组成部分，rabbitmq 只是AMQP的实现，从上面的描述中，可以知道AMQP的部分特点是：
* channel的存在可以让单个connection就可以存在多个不同的producer和consumer
* producer和consumer可以配置channel, exchange, queue的参数，这给了消息队列极大的灵活性
* 由 exchange 控制消息投递到那个queue上面去
* 消息从 rabbitmq server 到 consumer 默认是 push 模式的，而非 pull 模式，虽然AMQP也有个pull模式的方法(basic.get)

## exchange的类型
从之前的描述中我们知道：消息被投递到那个 queue 是由 exchange 决定的，所以 exchange 有4种类型， 在说明这4种类型之前，有必要先说些背景知识，以便更好的理解这4种类型
* 每个exchange 有两个参数：`exchange name`与`exchange type` 
* 消息在producer端发送时有两个参数: `exchange name` 与 `routing key`， 其中`exchange name`就是 exchange 的参数的`exchange name`
* 消息发送到最终的队列有队列名: `queue name` 指定
* `routing key`一般分段划分，比如`product.notify`，由`.`分割的我们称为word

上面说了几个名词`exchange name`, `exchange type`, `routing key`, `queue name`是不是有点云里雾里？说完exchange的类型(`exchange type`)就清楚啦
* direct，指定`exchange type`为`direct`，消息发送时指定`routing key`，消息会被投递到`queue name`与`routing key`一样的queue上
* topic，指定指定`exchange type`为`topic`，消息发送时指定`routing key`，它可以有两个特殊字符`*`与`#`，它们是通配符，`*`对应任意一个word，`#`对应任意多个word，只要`routing key`和`queue name`能匹配上就会发送到对应的queue上
* head，和topic类似，只不过是基于header数据来匹配，header数据和字典差不多，我们需要在设置header中key为`x-match`的value为`any`或`all`，当设置为`any`时只要其中有一个key符合就可以，设置为`all`时需要全部key都符合才能路由到对应的 queue上
* fanout，只要在消费时绑定了对应的exchange就可以，这种情况下`routing key`可以被忽略
* default，默认如果不提供exchange，那么 rabbitmq server 会给一个默认的 direct 类型的exchange

## 消息的持久化
消息的持久化需要将`exchange`, `queue`和`message`都配置成可持久化的

rabbitmq server 为了减轻fsync的压力在以下两个条件之一满足时会将持久化的信息存到磁盘：
1. 从上次持久化的时间点累计有一定数量的持久化消息没有持久化
2. 每隔几百毫秒

所以即使配置了持久化，如果rabbitmq server 机器crash了，消息也会存在一定程度的丢失

## 消息确认
rabbitmq 支持的消息确认模式：
* 自动确认，默认情况下，消息从producer发送出去的一瞬间就被认为是消息已经发送到rabbitmq server了
* `publisher confirm`，消息已经被rabbitmq server接收了，并且已经被存到磁盘了，意味着在这种情况下，消息基本上不可能丢失，除非磁盘坏掉了；需要在channel上将其设置为`confirm`模式
* `consumer ack`，消息被consumer确认成功消费，由于网络的复杂性，即便在双端确认启用的情况下，同一个消息有可能被重复发送两次，所以最好能保证consumer的幂等性

消息的确认在AMQP层面有三个方法：
* basic.Ack，肯定的消息确认，支持批量与单个消息的确认
* basic.Reject，否定的消息确认，只支持单个消息的确认
* basic.Nack，否定的消息确认，只支持批量消息的确认

需要注意的是，在批量消息确认的情况下，意味着之前的所有消息都被确认了，因为rabbitmq的消息确认机制是通过`channel`层面的`delivery tag`来控制的，它是一个自增的整数

还有一点需要注意的是消息确认是异步的，对于`pushlisher confirm`来说，需要注册一个回调来处理消息确认

## prefetch count
`prefetch count`的作用是为了平衡consumer的负载，因为比较简单的负载均衡策略会导致consumer消费不均衡的问题，所以需要这样一个参数来控制。比如如果在启用`consumer ack`的情况下其值设置为2，那么意味着在这个channel下，最多只能有2个未确认的消息

prefetch count有两个范围的含义：
1. connection范围，意味着在这个tcp连接下最多可以有`prefetch count`个未确认的消息
2. channel范围，意味着在这个channel下最多可以有`prefetch count`个未确认的消息

更多细节参考： https://www.rabbitmq.com/consumer-prefetch.html

## 死信队列
如果配置了私信队列，那么在以下几种情况消息会被重新投递到私信队列：
* 消息被拒绝 (basic.reject or basic.nack) 且带 requeue=false 参数
* 消息的TTL-存活时间已经过期
* 队列长度限制被超越（队列满）

私信队列的channel默认处于confirm模式

更多的可以参考：https://www.rabbitmq.com/dlx.html

## mandatory和immediate参数
消息发送到rabbitmq servver可以设置两个bool值的参数：mandatory, immediate

当mandatory标志位设置为true时，如果exchange根据自身类型和消息routeKey无法找到一个符合条件的queue，那么会调用basic.return方法将消息返回给生产者

当immediate标志位设置为true时，如果exchange在将消息路由到queue时发现对于的queue上没有消费者，那么这条消息不会放入队列中。当与消息`routing key`关联的所有queue都没有消费者时，该消息会通过basic.Return方法返还给生产者。

需要注意，如果采用消息`pushlisher confirm`模式，那么`basic.Return`和`basic.Nack`可能会同时返回给producer，所以需要注意这种情况

## 单机下的rabbitmq的消息可靠性保证
在单机模式下，如果想要保证消息是可靠的，在保证`publisher confirm`和`consumer ack`都启用的情况下，还需要对producer端做补偿，才有可能保证消息基本不会丢失，在`consumer ack`情况下，如果消息没有被确认，那么消息不会丢失，但是即便在`publisher confirm`情况下，如果消息发生失败，不在应用层做处理，还是不能避免消息丢失的可能性，可以采取将在消息发送之前将消息存放到一个可持久化的数据库中，只有接收到肯定的`confirm`消息，才将其中的已发送标识为置为True，这样可以在异常情况下，进行重新发送的补偿
