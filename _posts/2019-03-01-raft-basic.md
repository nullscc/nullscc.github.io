---
layout: post
title: 'raft简单理解'
tags: 分布式
---
raft算是一致性的算法，这里的一致性指的是：在任意时刻，连接到集群的client得到的结果和连接一台机器的效果是基本一样的

## 基本概念
一个没有crash的node在运行时有三个角色：
* Follower，作用是按Leader的执行顺序执行操作更改，刚开始所有的节点都是Follower
* Candidate，Leader crash时的Follwer一个中间状态，不会持续多久
* Leader，提供读写能力，所有的数据改变操作都需要经过Leader

Term：任期，针对Leader的变迁过程来说，只要发生了一次Leader的变换，那么系统的任期就+1，当Candidate或Leader发现自己的任期比收到的任期小，那么它就会回滚一些操作，并将自己的一些操作回滚到与当前Leader一致

Election timeout：每个Follower都会维护一个定时器

heartbeat timeout：每个Follower每隔`heartbeat timeout`时间发送一个hearbeat包，然后Follower会重置它的`Election timeout`

所有操作都要通过Leader，即便是读操作，如果读操作经过Follower，那么它还是有可能收到过期的数据

## 日志复制
日志复制是记录Leader节点的变化，当Leader有更新操作后，它会：
1. 数据变化到Leader后，Leader将数据变化以UNCOMMITTED的方式将变更日志保存起来
2. Leader将数据变化以日志的形式发送到Follwer节点
3. Follower节点收到这个数据变化后，将数据变化保存起来
4. Leader收到多数Follwer节点的数据变化成功提交的确认
5. Leader将数据变化以COMMITTED的方式将数据变化存到实际的存储中
6. 返回写入成功的Response给客户端

这里需要注意的是，Follower在确认数据的更改之前都需要确认上一个log的commit点是不是和Leader能对应上

## 选举
raft算法的选举过程如下：
1. 集群所有节点刚启动，所有节点处于Follower状态，并维护一个`Election Timeout`的定时器
2. 第一个Follower的定时器到时间了，Follower就成为Candidate
3. Candidate向所有节点请求自己想要成为Leader的投票
4. 当Candidate得到大多数节点的肯定投票，即成为新的Leader，记任期为：TERM 1，否则节点中的所有节点可以继续进行申请成为Leader的流程
5. 成为Leader以后，Leader需要定期发送一个心跳包，当Follower接收到这种心跳包时，就重置它的`Election Timeout`定时器，再次等待心跳包的到来
6. 如果Leader crash了，那么又重新进行第2～5步的流程，只不过任期会一步步增加

从这里可以看到，集群需要知道目前总共有多少个节点的，不然没法知道自己得到的是不是多数票

多数票选举形式保证只有一个Leader能选举成功

## 问题
从上面的情况我们可以看到，Follower crash了，系统的运作不会出现太大问题，只要系统中还存活的节点数还是占总节点的大多数就可以了

那么如果Leader crash了呢？我们分两种情况来看看：
1. 在收到client的操作请求后，Leader将操作修改提交，并且成功告知client后崩溃了，当client下次请求读数据，这时候会怎么样呢？因为Leader崩溃了，client不得不去另外一个节点上请求数据，然而另外一个节点会告诉client，我们在选举，你等一会，选举完成以后，告诉client去Leader节点取数据
2. 在收到client的操作请求后，Leader将操作修改提交，但是在告知client之前就崩溃了，这时候怎么样呢？这时候client能做的就只有重新提交这次操作啦，不过需要给每一个指令赋予一个序列号，当集群重新选举完以后，对于相同的已经提交的序列号，不做任何操作，这样数据不会重复提交了
3. 读操作可不可以经过Follower达到增加负载的目的呢？从一致性的角度来说，不能，因为数据的操作必须经过Leader，即使是读操作，即使client请求Follower，那么Follower也应该拒绝，反之给其一个应该去请求的地址。不然client可能读到脏数据，因为如果client A请求写一个数据到Leader，这个写因为网络波动，没有得到绝大多数的Follower的认可，这次写是不成功的，但是这时部分Follower上可能已经有client A写的数据了，这时client B去这部分的Follower上读数据就会读到client A没有写成功的数据

