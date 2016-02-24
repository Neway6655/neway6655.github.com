---
layout: post
title: "Hazelcast split brain"
description: "Hazelcast split brain analysis"
category: Hazelcast
tags: [hazelcast, split brain]
---

前段时间工作上遇到有关Hazelcast Split Brain的问题,对其内部关于网络脑裂部分的代码做了些分析.这里从学习的角度,结合代码分析下Hazelcast脑裂产生的过程以及Hazelcast是如何处理并恢复的[^1].

[^1]: 这里的分析都是基于Hazelcast 2.6.9的版本.

### 什么是脑裂 ###
wikipedia上关于脑裂的定义:

> Split-brain is a term in computer jargon, based on an analogy with the medical Split-brain syndrome. It indicates data or availability inconsistencies originating from the maintenance of two separate data sets with overlap in scope, either because of servers in a network design, or a failure condition based on servers not communicating and synchronizing their data to each other.

在Hazelcast里,简单来说,就是原来有一个10个member的集群,由于某些网络原因,导致其中的4个member和其余的6个member失去网络连接,从而原来10个member的这个集群被分割为两个独立的集群:一个4个member的集群和一个6个member的集群,出现这种现象就被称为Split-Brain Syndrome.

### 脑裂的产生 ###
脑裂的产生往往是因为网络抖动或不稳定等因素造成,意味着脑裂是由于外界因素的变化而产生的.那么在Hazelcast里,出现网络不稳定状况时,脑裂产生的过程是怎样的呢?想知道脑裂怎么产生,就要先知道Hazelcast的集群管理是怎么做的.

那么首先,在Hazelcast的集群里有两种类型的成员:master和普通member.一个集群里只有一个master,由最早加入集群的节点当选,master负责集群的管理.

其次,Hazelcast集群管理方法(比如heartbeart)有哪些呢,我们在```ClusterManager.java```这个类中可以找到他们:

```java
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

registerPeriodicProcessable(splitBrainHandler,
    splitBrainHandler.getFirstRunDelayMillis(), splitBrainHandler.getNextRunDelayMillis());
```

这些方法都是注册在```ClusterManager.java```这个类中,并由独立的线程控制执行.

OK,那我们先看看和产生脑裂相关的三个: 

- ##### heartBeater: ##### 
master和普通member都会发起heartBeater,
    - master定期向集群中的所有普通member发送heartBeater消息,同时检查是否有超过```MAX_NO_HEARTBEAT_MILLIS```时间都没有反应的member[^2],如果有,则认为这个member已经脱离集群,更新由master维护的集群member列表.
    - 普通member也会向cluster中其他member(包括master)发送heartBeater消息,如果master没有响应,那么会从集群的member列表中选举出一个新的master,**集群中每个member通过这个选举算法选出来的master是一致的**(集群成员列表按加入时间排序,找到当前master的下一个member作为新的master).

[^2]: Hazelcast为了避免网络里太多heartbeat的消息包,超过5s(不可配)没响应才会发一次ping,所以```MAX_NO_HEARTBEAT_MILLIS```这个配置不宜太小,否则会对网络太敏感,比如10s,意味着连续检查2次没响应就认为member脱离集群；但也不宜配得太大，否则对网络出问题时的响应太迟钝,比如10min,如果网络真的出了问题,hazelcast需要10分钟的时间才能确定剔除早已不在集群的member.因此,这个配置需根据实际情况谨慎配置,默认值是300s.

- ##### sendMemberListToOthers: #####
master向集群里其他member更新集群成员列表(因为每个member拿到的都是一样的列表,所以说各个member通过master选举算法选出来的master是一致的). 

- ##### sendMasterConfirmation: #####
根据上面的分析,master通过heartBeater检查,可以将master连不通的那些member从集群中移除,**但这个只能单向检查master到各个member的连接,而各个member到master的连接是否连通则无法得知**,于是就有了这个sendMasterConfirmation这个方法,这个方法由集群里的member向master发起MasterConfirmation的消息,master收到后用一个map记录下发起confirmation消息的member及时间,然后在heartBeater的过程中检查这个map,如果有集群的member不在这个map中(master收不到这个member的confirmation消息),或者map里记录的对应某个member的confirmation消息已有一段时间没有更新(master已有段时间都没有再收到这个member的confirmation消息),那么master都会将这样的member从集群里移除.

就这样,通过这三个方法,保证了master和集群里的member之间双向连通性,而将连接有问题的成员从集群中移走,这也就是Hazelcast脑裂产生的原理了.

我们再用一个简单的例子来说明一下整个过程[^3]:假设一个集群ClusterA,有5个member(M1,M2..M5,加入集群的顺序也是这样),M1为master,由于网络原因,M4,M5和其他member之间的网络断开了,M4和M5之间网络仍然正常,于是,master M1,通过heartBeater发现M4,M5没有响应,将他俩从集群里移除,更新cluster member列表,并通过sendMemberListToOthers通知M2,M3更新cluster member列表,而M4和M5的heartBeater检测到master M1连不上了,便重新选出M2作为新的master,但在下一次heartBeater检查时发现M2也连不上,于是再选出M3作为新的master,但也连不上,最后选出M4作为master,最后,脑裂出现了,原来的clusterA分成了两个clusterA': M1,M2,M3和clusterA": M4,M5,M1和M4分别是两个集群的master.

[^3]: 这里举的是一个简单脑裂情况的例子,有兴趣的同学可以自己想想其他更烧脑的情况,比如M1,M2,M3互通;M4,M5互通;M1可以连到M4,但M4连不通M1;M5可以连通M1,但M1连不通M5.

### 脑裂的恢复 ###
接下来再看看Hazelcast是如何从脑裂的状态下恢复正常的,当然前提是网络已经恢复正常.

这就要看```splitBrainHandler```这个方法了,它也是在上面注册到```ClusterManager.java```里的,顾名思义,它就是专门做脑裂处理的:

而它做所的事情就是寻找网络中的是否还有其他的跟自己是同一个group的cluster,如果存在,则进行merge,直到网络中只有一个cluster为止.这里有两个问题:如何发现其他的cluster?如何做merge?

- ##### 如何发现其他的cluster? #####
两种方式:tcp和multicast,在hazelcast的配置里,可以为集群设置network config是tcp的方式(指定所有member的ip address)或者multicast的方式,这里也是用这个配置,如果是tcp,则从配置的ip list里逐个遍历,查看是否有配置指定的member不在自己的cluster里,有则意味着这个member在另一个cluster中,说明网络中存在另一个cluster;如果是multicast,则发一个广播消息,向网络里的每一个member询问cluster的信息,若发现有和自己的cluster info不一致的member(前提是同一个group),也是一样，意味着网络中有其他cluster存在.

- ##### 如何做merge? #####
master先需要判断是否要做merge,因为hazelcast做merge的策略是小集群向大集群merge,所以master先依据在*如何发现其他的cluster*这一步拿到的cluster info和自己对比,如果自己的cluster成员数比人家多,就啥事不做,坐等人家merge,如果比人家小(如果相等,则比较master ip的hash值),就开始做merge了,先给cluster里的其他member发MergeClusters的命令,并带上targeAddress,然后把自己restart重新加入新的集群;对于其他member,收到MergeClusters的命令后,也是将自己restart重新加入新的集群,就这样,集群之间的merge就完成了.对于数据的merge,[官方文档](http://docs.hazelcast.org/docs/2.6/manual/html-single/#NetworkPartitioning)已经说得很清楚了.

我们继续之前的例子,再看看恢复的过程,在网络恢复正常后,clusterA'和clusterA"各自的master(M1和M4)开始干活啦,这里假设是用tcp的Network config方式连接,那么M1会向在Network config的配置列表里但不在自己cluster里member,也就是M4,M5,询问cluster info,拿到对方的cluster info发现比自己的cluster小,就坐等对方merge.而M4也是一样,会向M1,M2,M3询问cluster info,拿到后发现比自己的cluster大,于是告诉M5做merge,targetAddress是M1的地址,然后各自都restart,加入到clusterA'里,于是,网络中只剩下clusterA'这个集群了,脑裂恢复了.

##### Tips: 如何方便查看Hazelcast的集群信息 #####

```bash
curl 'http://{ip}:{port}/hazelcast/rest/cluster' 
```

ip,port分别是hazelcast instance的机器ip和hazelcast的端口.

请求会返回类似下面的一个结果:

```c
Members [5] {
    Member [10.20.17.1:5701]
    Member [10.20.17.2:5701]
    Member [10.20.17.4:5701]
    Member [10.20.17.3:5701]
    Member [10.20.17.5:5701]
 }
```