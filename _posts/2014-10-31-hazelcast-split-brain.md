---
layout: post
title: "Hazelcast split brain"
description: "Hazelcast split brain analysis"
category: Hazelcast
tags: [hazelcast, split brain]
---
{% include JB/setup %}

最近遇到Hazelcast Split Brain的问题，纠缠了几天，网上相关资源非常稀缺，无奈之下只能分析源码，趁热打铁记录下Hazelcast关于Split Brain的产生和处理方法。

注: 以下分析都是基于Hazelcast 2.6.9的版本.

### 什么是脑裂 ###
wikipedia上关于脑裂的定义：

> Split-brain is a term in computer jargon, based on an analogy with the medical Split-brain syndrome. It indicates data or availability inconsistencies originating from the maintenance of two separate data sets with overlap in scope, either because of servers in a network design, or a failure condition based on servers not communicating and synchronizing their data to each other.

简单来说，就是应该在一个集群里的成员，由于某些原因，从集群中分离出来，形成两个或多个集群的情况。

wikipedia上也说明了脑裂的两种解决方案:
- 乐观方案：认为脑裂的问题在短时间内是可以恢复的，集群的暂时不一致性是可以容忍的，集群最终会恢复正常而达到一致性。集群恢复可以是自动或人为手工干预恢复的。
- 悲观方案：认为脑裂的问题在短时间内是无法恢复的，集群宁愿牺牲一部分的可用性，也要保证集群的一致性。

### 脑裂的产生 ###
脑裂的产生往往是因为网络抖动不稳定等因素造成，导致集群的分裂，在Hazelcast里集群是由一个内部选举出来的Master管理的，Master会定期检查成员状态，更新所有集群成员信息并通知到各个成员，Master的这几个职责对应到代码里就是ClusterManager.java里的这么几个Processables:

        registerPeriodicProcessable(new Processable() {
            public void process() {
                heartBeater();
            }
        }, heartbeatIntervalMillis, heartbeatIntervalMillis);

        registerPeriodicProcessable(new Processable() {
            public void process() {
                sendMasterConfirmation();
            }
        }, masterConfirmationIntervalMillis, masterConfirmationIntervalMillis);

        registerPeriodicProcessable(new Processable() {
            public void process() {
                sendMemberListToOthers();
            }
        }, memberListPublishIntervalMillis, memberListPublishIntervalMillis);

下面会着重分析下这几个是干什么事情的:

首先看看heartBeater, Master和普通Member都会做heartbeat:


- Master的heartBeat，Master会定期主动给集群里每个成员发送heartbeat来检查他们的状态，**并且还会查看由各个集群成员发起的Master确认检查**，这里多解释一下(这也是Hazelcast 2.×早期几个版本的bug)，可以通俗理解成：Master会定期主动问每个成员“你是我的成员吗？”，成员会回答说“是啊”，然后Master就一个一个记录下来，把没有响应的从成员列表里移除掉，同时，每个成员也会主动问Master“你确认下我还在不在你的成员列表里？”，Master收到这种请求后，会核对下看看这个发起者还是不是我的成员，如果不在列表里，那我也需要把他移除掉。因为有这么一种暗恋的情况存在：Master(MA)每次给成员C发heartbeat，C都回应了，但在A的心目中，Master却是另一个(MB)，这样MA在收不到C的master确认消息后(因为C只会给MB发送master确认消息)，会果断把C踢掉，就可以避免这种单相思的情况出现了。
- 普通Member的heartBeat，每个成员也会定期给Master发送heartbeat，来检查Master是否存活，否则就会另选一个Master出来。

再看看sendMasterConfirmation:

- 在上面Heartbeat的时候已经提到过，这是Hazelcast2.×后面版本新加进去的一个东东，目的就是为了解决单相思的问题，而单相思的现象存在，会导致“整个大集群”(脑裂导致分裂出来的多个集群)的数据不一致，比如前面例子中的成员C被MasterA认为是在集群A里，但C又认为自己是在MasterB的集群B里，这样的集群数据不一致会导致后面说道的脑裂集群合并做不下去。

最后看看这个sendMemberListToOthers:

- 这个其实挺容易理解，就是Master会定期给集群里的每个成员更新所有成员的信息，来保证集群里所有Member拿到的集群信息一致。

OK，了解了这三个方法之后，再看看脑裂是怎么产生的，关键就看heartbeat：


- Master发送的heartbeat请求，如果超过MAX_NO_HEARTBEAT_MILLIS时间都没有响应，那么Master会把不响应的成员移除掉，那么这个成员就会游离在集群外，而当这个成员发送heartbeat给Master而Master也没有响应的话，这个成员会寻找新的master而组成新的集群，若Master的heartbeat仍能响应，但在发送Master Confirmation的时候，由于Master已经把他移除集群了，Master会回应说我已不再是你的Master，请重新再找个Master吧，然后这个成员也会寻找新的master而组成新的集群
- 同样，普通Member发送给Master的heartbeat请求，超过MAX_NO_HEARTBEAT_MILLIS时间没有响应，会寻找新的master而组成新的集群。

就这样，脑裂的现象就出现了，那么可以看到，这里关键的一点是MAX_NO_HEARTBEAT_MILLIS这个时间长度，因为heartbeat是每5秒(hardcode)做一次，也就意味着如果这个时间设得太短，比如10s，那么两次heartbeat没反应就会出现脑裂，这样轻微的网络抖动都可能导致脑裂的出现，但设置得太长，比如10分钟，那么网络真的出现问题的时候，中间会有10分钟的时间段出现请求出错异常等情况(因为有一些集群成员已经网络不通，但由于仍在集群中，请求仍会被发送到这些成员上)。所以对于这个配置值，需要谨慎配置。

