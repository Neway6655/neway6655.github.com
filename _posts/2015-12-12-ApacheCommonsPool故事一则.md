---
layout: post
title: "Apache Commons Pool 故事一则"
description: "对Java中最常用的对象池Apache Commons Pool的简要说明及分析"
category: commons-pool, java
tags: [commons-pool]
---

最近工作中遇到一个由于对commons-pool的使用不当而引发的一场血案，习得正确的使用姿势后简要总结分析下，希望可以帮到大家对apache commons-pool的理解。

>Apache Commons Pool, Java界无人不知无人不晓的对象池技术, 常用于实现各种连接池, 如数据库连接池, Redis连接池等


下面以租车公司为例子说明这张图，介绍commons pool的基本工作方式:

![Alt text](https://raw.githubusercontent.com/Neway6655/neway6655.github.com/master/images/commons-pool/GenericObjectPool.png)

我们将commons pool看成一个租车公司，那么:

* GenericObjectPool: 租车公司，管理着公司的所有车辆。
* PooledObject: 租车公司的车。
* EvictThread: 租车公司的车辆维修人员。

**GenericObjectPool**

作为租车公司，需要提供租车和收回归还的车辆的两个服务，同时它还要管理着它所有的那些车辆，比如随着业务发展壮大，需要买新车；对于已经不能安全驾驶的车辆，需要将其销毁，同时还要定期对车辆进行安全检测等。

**Borrow Object**

* A1: 世界那么大，年轻人想租辆车出去逛逛
* A2: 老板先看看有没有空闲的车
* A3.1: 如果有，则借出这辆车，并标记为已借出(Active)，如果没有空闲的车了，就买辆，也标记为已借出(这真是一家不差钱的公司)
* A3.2: 老板把标记好的车租给这位年轻人

**Return Object**

* B1: 世界那么大，年轻人终于逛完了，回来还车
* B2: 老板把车放回，并把标记改为空闲(Idle)，可以再被其他人租用。

**TestOnBorrow/TestOnReturn**

这家公司不仅不差钱，它还是一家很负责的租车公司，对于租出去的车，不管是从空闲车辆里取出的车，还是新买回的车，会先检查一遍这车的好坏，总不能坑了那年轻人，如果发现车有问题，立马再换一辆。同时，归还的时候，它也会检查一遍，如果有问题，就扔掉(真土豪)，除此之外，公司还专门请了一位车辆安检员，定期对闲置的车辆进行安全检测，一有问题就扔掉。

有借有还，看上去一切都很美好。

然而现实里总有意外发生：
年轻人借走车后，发现世界越逛越大，久久不愿回家。
安检员发现这车子都借出去大半年了，还没还回来，是不是丢了，掏出手机，啪的按了一下，将车子熄火掉，标记为报废车辆(Abandoned)，当作报废处理了。

**Evict Thread**

* C1: 对于空闲的车辆，安检员定期对它们检查，看是否坏掉，坏了就及时作废掉(C2).
* D1: 对于标记为已借出的对象，安检员定期检查时发现借出很久都未还，直接作废(D2)。

好了，故事讲完了，希望大家都能理解。

有兴趣的同学可以继续往下看看我们遇到的那场血案：

我们使用Jedis作为redis客户端操作，但在压测环境下，时不时发现Jedis有报：
```ClassCastException - [B cannot be cast to java.lang.Long```

网上各种google百度，发现大部分网友们说一般是由于pipeline操作，出现异常时连接没有正确destory掉，而直接放回连接池里，被下个线程拿到后，取到连接中残留的pipeline的操作结果，从而导致类型转换错误。

理解了这个后，反复检查代码，发现对于异常的封装都做好了，而且出先问题时也没有使用pipeline操作，怀疑是不是连接池出了问题，多个线程对同一个连接做了不同的操作，获取错了数据导致，但大名鼎鼎的commons-pool怎么可能出现这么低级的错误呢？翻了几遍commons-pool的代码后，发现可能是上面说的那个安检员捣的鬼，对于借出的对象，我们应用里以前配置了使用超过10秒后不归还则作废，理论上对于redis的操作，10秒确实怎么也做完了，但是，基于我们的业务场景，对JedisPool做了进一步的封装，对于一些特殊情况下，确实会出现持有连接超过10秒的情况(这个就不展开了)，导致程序还是读redis的数据时，被清理线程清理了(```jedis.quit()```)，jedis的quit命令返回的就是一个Byte数组，所以出现了ClassCastException，解决办法就是将作废时间的定义适当加大。