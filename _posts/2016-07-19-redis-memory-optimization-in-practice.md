---
layout: post
title: "Redis内存优化实践"
description: "近做的一个系统大量使用redis，我们将大量的用户信息存放在redis中，内存一申请就是几百G，体量也是相当庞大。所以我们也在不断的想方法优化减少redis的内存使用，把我们的优化实践也分享出来"
category: redis
tags: [redis, memory optimization]
---

最近做的一个系统大量使用redis，我们将大量的用户信息存放在redis中，内存一申请就是几百G，体量也是相当庞大。所以我们也在不断的想方法优化减少redis的内存使用，把我们的优化实践也分享出来。

##### 采用Hash代替<K,V>键值对存储
因为是存放用户维度的数据，用户id(uid)往往会作为key，而一个用户会有多个信息，比如年龄，生日等等，比较容易想到的存储结构会采用Hash，将一个用户的多个信息作为hash里的不同field来存放

##### 善用Hash，List，ZSet的ziplist压缩特性
Redis针对Hash，List，ZSet都实现了ziplist的压缩存储，可以通过配置最大元素不超过512，每个元素大小不超过64bytes，来判断是否要采用[ziplist压缩格式](http://redisbook1e-gallery.readthedocs.io/en/latest/7-ziplist.html)存储。

注意:虽然这个ziplist是否启用做成了配置参数，但对这个配置参数的修改要谨慎，因为ziplist是一个连续的数组空间，查找效率O(n)，如果设置元素超过512太多，可能导致查找效率降低，反而影响性能。那为什么Redis会采用512*64bytes这样的默认配置呢？据说是这个大小可以被加载进CPU的Cache里，所以即使不是O(1)，查找效率也是很快的。

如果List里元素太多，超过了512的限制，无法采用ziplist的压缩方式存储，会非常消耗内存，有个做法是在应用层将list里多个元素，比如多个string，并成一个‘大’的元素，这样可以减少list里的元素数量，由于list里每个元素都有前后指针等额外的内存开销，这样也可以节省大量的内存空间。当然前提是需要应用层自己对list里的元素做合并和拆解，应用本身的复杂度也会升高不少。

##### 优先使用数字类型，比String类型省空间
在Redis的内部，不管是数字类型，String类型，都会统一用一个叫redisObject的对象做一层封装:

```
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* lru time (relative to server.lruclock) */
    int refcount;
    void *ptr;
} robj;

```
可见，一个简简单单的"hello world"在redis里都不是直接11个bytes就搞定的，还有很多附加的属性，比如引用计数(内存回收)refcount，lru清理等信息。

但如果使用了上面提到的ziplist，redis在ziplist里对数字类型，做了一些优化。

我们先看看ziplist的大致结构：

```
<zlbytes><zltail><zllen><entry><entry>...<entry><zlend>
```

ziplist是一个连续的内存数组，每个entry就是里面的数据内容，针对hash结构，每个field和value都分别是一个entry，而每个entry又分为2个header和content，一个header是前一个entry的长度，另一个header是这个entry的encoding及string content的长度(如果是string的话)。

为什么需要知道长度？因为这里对每个entry的查找都是通过计算数组的下标位置来查找的，而string是变长的，所以想获取一个string的entry的内容，必须知道这个string的长度。

而int数字就不一样了，数字都是固定长度的，所以，redis在ziplist里对数字类型做了特殊处理：

```
* |11000000| - 1 byte
* Integer encoded as int16_t (2 bytes).
* |11010000| - 1 byte
* Integer encoded as int32_t (4 bytes).
* |11100000| - 1 byte
* Integer encoded as int64_t (8 bytes).
* |11110000| - 1 byte
* Integer encoded as 24 bit signed (3 bytes).
* |11111110| - 1 byte
* Integer encoded as 8 bit signed (1 byte).
* |1111xxxx| - (with xxxx between 0000 and 1101) immediate 4 bit integer.
* Unsigned integer from 0 to 12. The encoded value is actually from
* 1 to 13 because 0000 and 1111 can not be used, so 1 should be
* subtracted from the encoded 4 bit value to obtain the right value.
```
先用1byte来表示不同的encoding。针对大小不同的数字，分别采用不一样的内存空间来存储，比如0-127就是2个字节，128-32768就是4个字节等等。所以算下来，和String相比，使用int类型相对会更省内存。

另外，如果不是采用ziplist的存储方式，而是直接用redisObject这样相对庞大的对象存储呢？

如果能用数字，还是尽量使用数字类型，并且是小于10000的数字最好，因为：

```
#define OBJ_SHARED_INTEGERS 10000
```
redis考虑到redisObject这个庞大的对象占用过多内存的因素，将10000以下数字的redisObject做了一个对象池，其他地方都通过指针(4/8bytes)引用这个池里的redisObject，而不是各自存一份。

注: 以上都是针对Redis 3.2之前版本的分析，因为Redis 3.2对内存优化这部分做了很多改进，具体的改进点还未了解清楚。

最后，对坚持看完的同学送上一个非常有用的Redis内存分析工具: [redis-rdb-tools](https://github.com/sripathikrishnan/redis-rdb-tools)，结合bgsave的dump文件，分析redis里的数据，可以看到底层存储是用的什么数据结构，占用了多少空间等信息。






