---
layout: post
title: 'kafka基本概念'
tags: 消息队列
---
kafka是生产-消费模型的消息队列，它是通过分区和副本集这种横向扩展的方式来提升吞吐量，和rabbitmq相比，它通过相对较复杂的分区分布式设计和牺牲极少丢消息的概率来达到极高的吞吐量

## kafka组成部分
* producer，生产消息的成员，由于一个消息可以被选择性的投递到不成的partition，所以producer可以控制投递到哪个partition上
* topic partition，kafka通过很简单的配置就能实现多机器分区，这应该算是核心设计了，每个分区上面可能有同一个主题的消息
* replication，kafka的复制单位是分区，即每个分区都可能有多个副本集
* zookeeper，目前了解到zookeeper在kafka架构里面最主要的功能是在leader选举的时候，因为它提供了强一致性，这样能保证在选举时不会产生多个leader，从而产生脑裂的情况
* consumer group，kafka通过consumer group的设计来消费同一topic下不同分区的消息，它保证了，同一个group下的两个consumer不会消费同一个分区的消息

关于分区和副本，kafka尽量保持它们平均分配在各个broker上，以尽量的分摊压力

## kafka消息保存方式
kafka消息保存是通过接收即保存的方式来实现，即kafka收到一个消息即将它保存到磁盘。只不过由于消息队列的特殊性，它是以只读追加的方式来保存消息，这样消费时只通过读取消息的偏移量来消费，通过这种方式保证不管是消息的保存还是消费都是顺序IO，这样即便是每个操作都操作磁盘，吞吐性能基本上只与网络IO的速度挂钩

消息的删除时机和传统消息队列也有很大不同，传统消息队列是通过没消费一条消息就删除一条消息来实现，但是kafka会通过定期删除老的数据来实现，这样删除消息的效率也非常高，因为要删除的消息基本上不会有消费者在消费了，很久才操作一次的删除一大块磁盘内容也不是很大问题

## kafka的复制模式
kafka的复制模式可选有：同步复制，半同步复制，异步复制，对于异步复制，producer只等待或不等待leader的成功保存消息的结果，kafka的同步复制和半同步复制原理一致，它通过维护一个ISR(in-sync replication)副本列表来保证副本集间副本的一致性

当分区的其中一个机器坏掉了，刚好这个机器上存在leader，那么就需要重新进行leader选举，leader选举通过强一致性的key-value存储系统zookeeper来实现，保证在任何时候都不会发生多个leader的情况

当leader crash时，kafka提供一个`Disable unclean leader election`的选项来选择是否允许一个不在ISR的副本成为leader，这同样是在C 和 A之间做选择(CAP理论)

## kafka的应用模式
kafka可以有很多种应用场景，根据kafka提供的api，主要可以有以下几种应用模式
* 传统消息队列模式，典型的生产者生产消息，消费者消费消息
* stream，从某个topic下读取消息，将处理结果写到另外一个topic下，例子可见：[https://kafka.apache.org/documentation/streams/](https://kafka.apache.org/documentation/streams/)
* connector，让topic下的消息与外部文件系统或者数据库产生联系， 例子可见： [https://kafka.apache.org/quickstart#quickstart_kafkaconnect](https://kafka.apache.org/quickstart#quickstart_kafkaconnect)

## kafka为什么吞吐量这么高
* producer端批量消息发送优化，producer发送消息到kafka不是每条消息都会与kafka broker交互，而是可以通过积累一定的消息量来提升性能
* zero-copy优化，保证消息的发送到tcp连接的速度基本上和网卡速度一致
* 不管是消息的保存还是消息的消费都是顺序IO，且尽量保持批量读取

## kafka消息可靠性
* producer将消息累计在内存中，然后批量发送，意味着在某些极端情况下，我们需要处理消息发送期间的丢失问题，比如重试最大次数上限过后
* 对于broker，kafka本身通过ISR机制来保证消息的一致性与可靠性，这点小心处理不用担心
* 对于consumer，我们只需要尽力保证消费消息是幂等的就够了

## kafka的pull模式
long-poll，由于消费端是主动请求消息，而不是由broker push给消费端，简单实现可能是轮询的方式，但是这种方式有比较打的性能问题，所以kafka采取保持一个long-pull的连接来实现，kafka的pull模式最主要的优点是consumer可以自己控制消费的位置和消费速率，这样避免以下push过多消息把consumer给冲垮

## kafka的消息保证语义
消息的可靠性保证有三种语义：
* 至少一次
* 至多一次
* 正好一次

在说这三种语义之前来说说为什么会出现这三种保证，原因在于：网络的不确定性，如果因为某种原因网络突然断开了，那在这之前producer发送的消息有没有到达kafka broker呢？答案是不确定。这就引出了消息可靠性保证的三种语义，它们代表了开发者需要根据应用情况进行合理的选择

我们分别从producer和consumer来阐述消息可靠性的保证

### producer消息保证语义
producer在发送消息到kafka时，kafka本身会给producer一个回应，告诉producer说：消息我收到了，并且已经保存啦。那么kafka什么在什么时机给producer这个回应呢？它有三种选择方式(由producer控制)：
* 不回应，producer选择无需回应，只要发送出去了，就认为kafka已经接收到这个消息了
* 只等待partition的leader回应，不管followers的状态
* 等待ISR副本回应

* 至少一次，producer选择等待ISR副本回应，这样发送端会多发消息，但是能保证不会丢失消息，但是消息可能被重复发送，因为在网络不稳定的情况下，ISR副本回应了，但是producer不知道，它可能选择再发送一次
* 至多一次，producer选择不回应，这样保证消息不会多发，但是能保证消息最多只发送一次
* 正好一次，从kafka0.11版本开始，支持一个幂等性的消息发送，通过给每个producer分配一个producer ID，并且每个消息分配一个序列号来实现

### consumer消息保证语义
consumer消费时会提交一个已消费的offset给kafka，kafka通过其中的一个coordinator来保存(也可以选择通过zookeeper，选择zk可能会影响性能)，那么就引来一个问题，是先保存这个offset还是先处理消息，这样就引出了至少一次和至多一次的语义保证
* 至多一次，先提交offset再处理消息
* 至少一次，先处理消息再提交offset
* 正好一次，通过将offset与消费的成果保存在一起，比如将它们保存在关系型数据库或者实现一个2PC分布式事务方案，通过unique或原子性等约束来保障，其实就是借助offset设计幂等性的consumer

## 总结
kafka是大数据环境下的消息队列，由于它的模式的特殊性，可以有很多的应用场景，不过由于它天然的分布式设计，我们需要在实际工作中多多权衡它提供的各种选择以使其符合我们的业务需要

一句话来评价kafka：高性能而不易掌控