---
layout: post
title: "性能测试及调优"
description: ""
category: performance test, performance tuning
tags: [performance test, performance tuning]
---

### 性能测试

##### 明确性能测试的目标及场景

##### 准备性能测试环境，测试case

##### 执行性能测试并做好监控

### 性能调优

##### 关于调优的几点经验
> 调优的手段有很多，从性价比最高的方式开始

比如: 

* 有没办法可以快速重现问题
* 先看看系统大体的情况，比如CPU，内存，IO(磁盘/网络)，再检查细项
* 多用简单方便易用的工具/方法初步定位，比如JDK提供的那些基本工具: jstack, jstate, jmap

举个栗子，给一个人看病，如果看看喉咙，用下听诊器就可以判定病症了就没有要验血化验了。

> 一个问题可能是由多个原因导致的，系统越复杂，问题越多

如果是个老者过来看病，可能会发现一个症状背后还有很多隐藏的问题，需要找出这些问题之间的千丝万缕的关系，再一一治疗。

> 多点怀疑，是好事，但不要乱猜

如果一个男的看肚子疼，可以怀疑是不是吃错东西，但总不能怀疑人家怀孕了吧。

> 把问题抛出来，和其他同事一起讨论下，开拓一下思路，避免走到死胡同

> 还是没头绪，就翻代码吧，祝好运

> 最后还没招？那就先休息一下吧


##### 调优例子
> 现象1: 单机应用服务器CPU很高(80%), QPS达到1.3w 
这个仅从现象来看似乎也没有问题，单机能跑到1.3w已经很高了，所以CPU很忙可能是正常的。但结合具体的应用来看，这个应用并不是CPU密集型的，所以还是不要偷懒，看看thread dump有没有什么发现:

```
"OspThreadPool-236" prio=10 tid=0x00007f9c880dfea0 nid=0x617 waiting for monitor entry [0x00007f9b248c7000]
   java.lang.Thread.State: BLOCKED (on object monitor)
	at java.util.Hashtable.writeObject(Hashtable.java:852)
	- waiting to lock <0x0000000760028920> (a java.util.Properties)
	at sun.reflect.GeneratedMethodAccessor125.invoke(Unknown Source)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:601)
	at java.io.ObjectStreamClass.invokeWriteObject(ObjectStreamClass.java:962)
	at java.io.ObjectOutputStream.writeSerialData(ObjectOutputStream.java:1480)
	at java.io.ObjectOutputStream.writeOrdinaryObject(ObjectOutputStream.java:1416)
	at java.io.ObjectOutputStream.writeObject0(ObjectOutputStream.java:1174)
	at java.io.ObjectOutputStream.writeObject(ObjectOutputStream.java:346)
	at com.vip.venus.core.context.util.Utils.deepClone(Utils.java:48)
	at com.vip.venus.core.context.property.PropertyHolder.getProperties(PropertyHolder.java:37)
	at com.vip.venus.core.context.property.impl.PropertyManagerImpl.getProperty(PropertyManagerImpl.java:42)
	at com.vip.usp.usprtsapi.service.impl.RealTimeStableUserServiceImpl.getRealTimeUserProperty(RealTimeStableUserServiceImpl.java:80)
```

搜一下usp, vip等关键字，很容易发现这么一段，很显然不正常，从PropertyManager里取一个property怎么会有等锁，为什么会用到deepClone？谜底揭开了，大量的线程做深度克隆，不断的序列化反序列化当然就导致了CPU飙高。

> 现象2: Jmeter开100个线程压测，刚压一会就报错说服务端线程不够用
这个问题藏的比较深，先说说服务端的线程池创建:

```
executor = new ThreadPoolExecutor(256, 512, 60L, TimeUnit.SECONDS, new SynchronousQueue<Runnable>(), new OspThreadFactory("Osp-Business-Thread"));
```

coreSize=256, maxPoolSize=512, 采用SynchronousQueue, 也就是说如果线程数用到256后，再来请求，如果没有core threads可用，就马上创建一个临时线程来服务，当core threads+临时线程超过maxPoolSize就报错说拒绝服务了。

那么问题来了，明明Jmeter只开了100个线程，服务端为什么会用到超过512条线程?

为什么说问题隐藏的比较深，主要是不好解释上面这个现象，另外thread dump因为不知道具体什么时候会出现问题，等看到错误再thread dump只能抓到一堆线程都在积极地等Queue取任务干活，一片正常，但为什么开了这么多线程，仍然是个迷...

于是，只要程序的问题用程序来解决，在程序里报错之后，程序自动触发一次thread dump，记录下当场的景象，再来分析，于是再次重现后抓到了非常宝贵的thread dump(丢失了)，发现有很多线程堵塞在logback的AsyncAppender的ArrayBlockingQueue.put()方法上，代码上分析得出是因为这个异步写日志的queue满了(size=1024)导致线程阻塞了，但明明线程已经阻塞了，为什么还会有线程请求过来呢？

翻代码，发现代码里虽然是用了异步写日志，但对于客户端的请求，服务端线程是先flush回去结果，再异步写日志。所以就出现了虽然Jmeter只开100个线程，但服务端可能由于日志写得满(并且服务端的处理过快)导致服务端线程不断增加，最终堵在写日志的路上...





