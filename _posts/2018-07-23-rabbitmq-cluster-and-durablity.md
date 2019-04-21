---
layout: post
title: 'rabbitmq“集群”与消息可靠性保证'
tags: 消息队列
---
标题的集群是打引号的，为什么呢？因为rabbitmq本身的cluster并不算真正的集群，增加机器并不能保证吞吐量基本成比例上升，因为存在queue的单点问题，不过还是有办法支持真正的广域网集群模式，不过这里主要说说适用于局域网的cluster模式与镜像队列

## rabbitmq的可用集群模式
* 使用cluster，我们在rabbitmq下说的集群基本上就是通过这种形式，它通过将除queue与message的部分分布到多台机器上来实现，但是没办法将queue的压力分担到多台机器上(即便启用镜像队列)，适用于局域网
* 使用federation或shovel，这两种可以放在广域网中，本文不具体讨论这两种形式

## 镜像队列
镜像队列，是将rabbitmq的queue组件配置成主从模式，从而让队列和消息分布在多台机器上。它的特点：
1. 一个master，任意多个slave
2. 同步复制，消息如果发送到master上，那么必须等待复制到slave成功
3. 由于是同步复制，所以镜像队列也不太适合分布在广域网中
4. slave不提供读写功能，所有的读写必须通过master

### master crash
如果由于某种原因，master不能提供工作了，那么需要从slave中选举出一个新的master来供读写，一般情况下，新选举的master不会丢失消息，但是在非常少见的情况下，新选举的master可能没有同步到所有消息，可以视配置而定是否允许这种情况(分布式环境下的可用性与一致性二选一问题)

## 镜像队列情况下的消息可靠性
rabbitmq被设计成一个cp系统，即使在cluster+镜像队列+`publisher confirm`+`consumer ack`的情况下，rabbitmq仍然可以视为是一个CP(consitency and partition)系统

由于一般情况下，slave是与master同步的，所以可以认为即使在配置镜像队列的情况下，消息在rabbitmq server下是可靠的

其次，如果启用了`publisher confirm`，那么如果producer发送一条消息到master，那么必须等到slave的所有镜像队列都确认将消息写入到磁盘后才会想producer返回一个确认收到的消息

## 为什么要有镜像队列与cluster
既然镜像队列与cluster都不能分担queue的压力到多台机器上，为什么还需要？

首先镜像队列保证即使在master机器磁盘坏掉的情况下也可以保证消息基本不丢失，而且这在分布式环境下的高可用很重要

其次，cluster虽然不能分担queue的压力，但是exchange的路由选择等也是需要耗费一定的cpu，所以还是能在一定程度上分担压力的

即cluster+镜像队列相当于冗余备份的同时，可以有限的分担机器压力，并且在crash情况下恢复可以做到用户基本无感知，即不保证非常高的吞吐量，但是保证高可用与消息高度可靠性
