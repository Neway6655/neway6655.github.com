---
layout: post
title: "Redis的hmget操作复杂度问题"
description: "hmget的操作复杂度真的是O(N)吗？"
category: redis
tags: [redis]
---

目前负责的系统，强依赖于Redis，高并的接口压力和瓶颈都主要集中在Redis上，对Redis的操作基本都是hmget，之前遇到了一个性能瓶颈，花了些时间对hmget进行更深入的学习，以此记录下来，供大家参考。

##### 遇到的性能问题
先简单介绍下背景，我们对hmget的fields之前没有加控制，正常情况下都是请求10个不到的fields，但有些特殊场景会出现一次hmget100多个fields，甚至严重时会出现200个fields，由于当时Redis集群的压力不大，并没有把这个问题凸现出来。所以我们立刻在测试环境做了一次验证，本以为100个fields的性能应该比10个fields的性能差1/10左右，但测试的结果却远不止。

##### 难道hmget的操作复杂度不是O(N)?
[Redis官网](http://redis.io/commands/hmget)对于hmget的操作复杂度白纸黑字写得很清楚就是O(N)，N为请求的fields个数。

于是抱着怀疑的精神翻出Redis关于hmget操作的源码(Redis 3.0分支，t_hash.c)：

```
void hmgetCommand(redisClient *c) {
    robj *o;
    int i;

    /* Don't abort when the key cannot be found. Non-existing keys are empty
     * hashes, where HMGET should respond with a series of null bulks. */
    o = lookupKeyRead(c->db, c->argv[1]);
    if (o != NULL && o->type != REDIS_HASH) {
        addReply(c, shared.wrongtypeerr);
        return;
    }

    addReplyMultiBulkLen(c, c->argc-2);
    for (i = 2; i < c->argc; i++) {
        addHashFieldToReply(c, o, c->argv[i]);
    }
}
```
看到代码里最后其实是遍历各个请求的fields，调用```addHashFieldToReply```方法：

```
...
if (o->encoding == REDIS_ENCODING_ZIPLIST) {
	...
} else if (o->encoding == REDIS_ENCODING_HT) {
	...
}
```
看到这段```if/else```似乎明白了什么，之前的文章[Redis内存优化实践](http://neway6655.github.io/redis/2016/07/19/redis-memory-optimization-in-practice.html)里说到的关于hash底层的ziplist数据结构，其实就是一个连续空间的数组，而数组的查找是需要遍历里面的元素的，和hashtable的O(1)不同。而我们采用的也正是ziplist的压缩结构 (由于数据量太大，ziplist结构可以省大量内存，节省成本)。所以，对于ziplist的hmget操作复杂度应该不是O(N)。

抱着这个想法进一步看进第一个if里的代码：

```
if (o->encoding == REDIS_ENCODING_ZIPLIST) {
        unsigned char *vstr = NULL;
        unsigned int vlen = UINT_MAX;
        long long vll = LLONG_MAX;

        ret = hashTypeGetFromZiplist(o, field, &vstr, &vlen, &vll);
        if (ret < 0) {
            addReply(c, shared.nullbulk);
        } else {
            if (vstr) {
                addReplyBulkCBuffer(c, vstr, vlen);
            } else {
                addReplyBulkLongLong(c, vll);
            }
        }
} 
```
看到这里，我们还是先回顾一下ziplist的数据结构吧：

```
<zlbytes><zltail><zllen><entry><entry>...<entry><zlend>
```
除掉头尾，中间都是各个entry直接拼起来，hash里的每个field和value都是一个entry，field的entry后面跟着的就是它的value的entry，而每个entry又分为2个header和content，一个header是前一个entry的长度，另一个header是这个entry的encoding及string content的长度(如果是string的话)，简单来说在ziplist里识别entry是通过长度来判断的。

再回到上面的代码里，针对ziplist的情况，对一个field的value查找分成了两步：一、先通过field找到对应ziplist的位置，从而判断出value的位置```vstr```；二、再通过value entry的header里记录的value的长度，找出对应的value值。

第一步，看看```hashTypeGetFromZiplist```里是如何查找field的:

```
...
fptr = ziplistIndex(zl, ZIPLIST_HEAD);
if (fptr != NULL) {
        fptr = ziplistFind(fptr, field->ptr, sdslen(field->ptr), 1);
        if (fptr != NULL) {
            /* Grab pointer to the value (fptr points to the field) */
            vptr = ziplistNext(zl, fptr);
            redisAssert(vptr != NULL);
        }
}
...
```
大致可以看出，查找的过程是从ziplist的第一个entry开始，通过```ziplistFind```方法查找field，然后推算出field的value在ziplist的位置。```ziplistFind```方法源码(ziplist.c):

```
...
while (p[0] != ZIP_END) {
	...
	if (len == vlen && memcmp(q, vstr, vlen) == 0) {
        	return p;
    }
}
...
```
终于，猜测是对的，ziplist对field的查找就是遍历比较的方式。所以整个hmget里的field的查找是O(M)复杂度，M为hash的field总数[1]，再根据field找出对应的value值就只是一个根据长度取出具体值的过程。

所以，对于ziplist结构的hmget操作复杂度应该是O(N*M)，N为请求的fields数量，M为这个hash里一共有的fields数量。而对于hashtable结构的hmget操作自然是O(N)了。

##### 结论
对于Hash，Set等数据结构，由于Redis所采用的底层的存储结构可能出现不同，对一些操作的使用需要格外小心。而且这个底层存储结构在数据发生变化后可能还会自我调整，比如hash的entry个数超过512(默认值)后，会从ziplist变成hashtable结构等，这些变化，不仅对存储有影响，对一些操作的效率也一样有影响，所以开发同学必须非常了解自己的应用所存放在redis的数据变化情况，避免掉坑。

正所谓“知己知彼，方能百战百胜”，也是这个道理吧。

------------


![image](https://raw.githubusercontent.com/Neway6655/neway6655.github.com/master/images/redis-hmget/hmget.png)







