---
layout: post
title: "Elasticsearch 学习笔记"
description: "Elasticsearch的学习笔记"
category: elasticsearch
tags: [search]
---

##介绍
>Elasticsearch 是一个分布式可扩展的实时搜索和分析引擎.

Elasticsearch 是一个建立在全文搜索引擎 Apache Lucene(TM) 基础上的搜索引擎.
当然 Elasticsearch 并不仅仅是 Lucene 那么简单，它不仅包括了全文搜索功能，还可以进行以下工作:
* 分布式实时文件存储，并将每一个字段都编入索引，使其可以被搜索。
* 实时分析的分布式搜索引擎。
* 可以扩展到上百台服务器，处理PB级别的结构化或非结构化数据。

###基本概念

先说Elasticsearch的文件存储，Elasticsearch是面向文档型数据库，一条数据在这里就是一个文档，用JSON作为文档序列化的格式，比如下面这条用户数据：

{% highlight json %}
{
    "name" :     "John",
    "sex" :      "Male",
    "age" :      25,
    "birthDate": "1990/05/01",
    "about" :    "I love to go rock climbing",
    "interests": [ "sports", "music" ]
}
{% endhighlight %}

用Mysql这样的数据库存储就会容易想到建立一张User表，有balabala的字段等，在Elasticsearch里这就是一个*文档*，当然这个文档会属于一个User的*类型*，各种各样的类型存在于一个*索引*当中。这里有一份简易的将Elasticsearch和关系型数据术语对照表:

关系数据库     ⇒ 数据库 ⇒ 表    ⇒ 行    ⇒ 列(Columns)
Elasticsearch  ⇒ 索引   ⇒ 类型  ⇒ 文档  ⇒ 字段(Fields)

一个 Elasticsearch 集群可以包含多个 索引（数据库），也就是说其中包含了很多 类型（表）。这些类型中包含了很多的 文档（行），然后每个文档中又包含了很多的 字段（列）。

Elasticsearch的交互，可以使用Java API，也可以直接使用HTTP的Restful API方式，比如我们打算插入一条记录，可以简单发送一个HTTP的请求：

{% highlight json %}
PUT /megacorp/employee/1
{
    "name" :     "John",
    "sex" :      "Male",
    "age" :      25,
    "about" :    "I love to go rock climbing",
    "interests": [ "sports", "music" ]
}
{% endhighlight %}

更新，查询操作类似这样操作就可以了

----------

##索引

Elasticsearch最关键的就是提供强大的索引能力了，其实InfoQ的这篇[时间序列数据库的秘密(2)——索引](http://www.infoq.com/cn/articles/database-timestamp-02?utm_source=infoq&utm_medium=related_content_link&utm_campaign=relatedContent_articles_clk)经讲的非常详细了，我只是再结合自己的理解梳理下。

Elasticsearch索引的精髓：

>一切设计都是为了提高搜索的性能

另一层意思：为了提高搜索的性能，难免会牺牲某些其他方面，比如插入/更新，否则其他数据库不用混了:)

前面看到往Elasticsearch里插入一条记录，其实就是直接PUT一个json的对象，这个对象有多个fields，比如上面例子中的*name, sex, age, about, interests*，那么在插入这些数据到Elasticsearch的同时，Elasticsearch还默默[^1]的为这些字段建立索引--倒排索引，因为Elasticsearch最核心的是提供搜索功能。

###Elasticsearch是如何做到快速索引的

InfoQ那篇文章里提到Elasticsearch使用的倒排索引比关系型数据库的B-Tree索引快，为什么呢？

####什么是B-Tree索引?

二叉树查找效率是logN，同时插入新的节点不必移动全部节点，所以用树型结构存储索引，能同时兼顾插入和查询的性能。

在这个基础上，再结合磁盘读取的特性(顺序读/随机读)，传统关系型数据库采用了B-Tree的数据结构：

![Alt text](https://raw.githubusercontent.com/Neway6655/neway6655.github.com/master/img/elasticsearch-study/b-tree.png)

为了提高查询的效率，减少磁盘寻道次数，将多个值作为一个数组通过连续区间存放，一次寻道读取多个数据，降低树的高度。

####什么是倒排索引?

![Alt text](https://raw.githubusercontent.com/Neway6655/neway6655.github.com/master/img/elasticsearch-study/inverted-index.png)

继续上面的例子，假设有这么几条数据(为了简单，去掉about, interests这两个field):
| ID | Name | Age  |  Sex     |
| -- |:------------:| -----:| -----:| 
| 1  | Kate         | 24 | Female
| 2  | John         | 24 | Male
| 3  | Bill         | 29 | Male

ID是Elasticsearch自建的文档id，那么Elasticsearch建立的索引如下:

**Name:** 

| Term | Posting List |
| -- |:----:|
| Kate | 1 |
| John | 2 |
| Bill | 3 |

**Age:**

| Term | Posting List |
| -- |:----:|
| 24 | [1,2] |
| 29 | 3 |

**Sex:**

| Term | Posting List |
| -- |:----:|
| Female | 1 |
| Male | [2,3] |

#####Posting List

可以看到Elasticsearch会分别为每个field建立一个倒排索引，Kate, John, 24, Female这些叫做 term，而[1,2]就是**Posting List**。Posting list就是一个int的数组，存储了所有符合某个term的文档id。

看到这里，不要认为就结束了，精彩的部分才刚开始...

通过posting list这种索引方式似乎可以很快进行查找，比如要找age=24的同学，爱回答问题的小明马上就举手回答：我知道，id是1，2的同学。但是，如果这里有上千万的记录呢？如果是想通过name来查找呢？

#####Term Dictionary

Elasticsearch为了能快速找到某个term，将所有的term排个序，二分法查找term，logN的查找效率，就像通过字典查找一样，这就是**Term Dictionary**。现在再看起来，似乎和传统数据库通过B-Tree的方式类似啊，为什么说比B-Tree的查询快呢？

#####Term Index

B-Tree通过减少磁盘寻道次数来提高查询性能，Elasticsearch也是通过这样的思路，Elasticsearch直接通过内存查找term，这样就不用扫磁盘了，但是如果term太多，term dictionary也会很大，放内存不现实，于是有了**Term Index**，就像字典里的索引页一样，A开头的有哪些term，分别在哪页，可以理解term index是一颗树：
![Alt text](https://raw.githubusercontent.com/Neway6655/neway6655.github.com/master/img/elasticsearch-study/term-index.png)

这棵树不会包含所有的term，它包含的是term的一些前缀。通过term index可以快速地定位到term dictionary的某个offset，然后从这个位置再往后顺序查找。
![Alt text](https://raw.githubusercontent.com/Neway6655/neway6655.github.com/master/img/elasticsearch-study/index.png)

所以term index不需要存下所有的term，而仅仅是他们的一些前缀，再结合通过FST(Finite State Transducers)的压缩技术，可以使term index以树的形式缓存在内存中。从term index查到对应的term dictionary的block位置之后，再去磁盘上找term，大大减少了磁盘随机读的次数。

这时候爱提问的小明又举手了:"那个FST是神马东东啊?"

一看就知道小明是一个上大学读书的时候跟我一样不认真听课的孩子，数据结构老师一定讲过什么是FST。但没办法，我也忘了，这里再补下课：
>FSTs are finite-state machines that **map** a **term (byte sequence)** to an arbitrary **output**.

假设我们现在要将mop, moth, pop, star, stop and top映射到他们的字典排序的序号：0，1，2，3，4，5。最简单的做法就是定义个Map<String, Integer>，大家找到自己的位置入座就好了，但从内存占用少的角度再想想，有没有更优的办法呢？答案就是：**FST**([理论依据在此，但我相信你不会想去看的](http://www.cs.nyu.edu/~mohri/pub/fla.pdf))
![Alt text](https://raw.githubusercontent.com/Neway6655/neway6655.github.com/master/img/elasticsearch-study/fst.png)

⭕️表示一种状态
➡️表示状态的变化过程，上面的字母/数字表示状态变化和权重

将单词分成单个字母通过⭕️和➡️表示出来，0权重不显示。如果⭕️后面出现分支，就标记权重，最后整条路径上的权重加起来就是这个单词对应的序号。

>FSTs are finite-state machines that map a term (**byte sequence**) to an arbitrary output.

FST以字节的方式存储所有的term，这种压缩方式可以有效的缩减存储空间，使得term index足以放进内存，但这种方式也会导致查找时需要更多的CPU资源。

![Alt text](https://raw.githubusercontent.com/Neway6655/neway6655.github.com/master/img/elasticsearch-study/fst-compress.png)

后面的更精彩，看累了的同学可以喝杯咖啡……

----------

####压缩技巧

Elasticsearch里除了上面说到用FST压缩term index外，对posting list也有压缩技巧。
小明喝完咖啡又举手了:"posting list不是已经只存储文档id了吗？还需要压缩？" 
嗯，我们再看回最开始的例子，如果Elasticsearch需要对同学的性别进行索引(这时传统关系型数据库已经哭晕在厕所……)，会怎样？如果有上千万个同学，而世界上只有男/女这样两个性别，每个posting list都会有至少百万个文档id。
Elasticsearch如何有效的对这些文档id压缩的呢？

#####Frame Of Reference

>增量编码压缩，将大数变小数，按字节存储

首先，Elasticsearch要求posting list是有序的（为了提高搜索的性能，再任性的要求也得满足），这样做的一个好处是方便压缩，看下面这个图例：
![Alt text](https://raw.githubusercontent.com/Neway6655/neway6655.github.com/master/img/elasticsearch-study/frameOfReference.png)

如果数学不是体育老师教的话，还是比较容易看出来这种压缩技巧的吧。

原理就是通过增量，将原来的大数变成小数存储，再精打细算按bit排好队，最后通过字节存储，而不是大大咧咧的尽管是2也是用int(4个字节)来存储。

#####Roaring bitmaps

说到Roaring bitmaps，就必须先从bitmap说起。Bitmap是一种数据结构，假设某个posting list如下：

[1,3,4,7,10]

对应的bitmap就是：

[1,0,1,1,0,0,1,0,0,1]

用0/1表示某个值是否存在，比如10这个值对应第10位的bit值就是1，这样用一个字节就可以代表8个文档id，旧版本(5.0之前)的Lucene就是用这样的方式来压缩的，但这样的压缩方式仍然不够高效，如果有1亿个文档，那么需要12.5MB的存储空间，这仅仅是对应一个索引字段(我们往往会有很多个索引字段)。于是就有人想出了Roaring bitmaps这样更高效的数据结构。

Bitmap的缺点是存储空间随着文档个数线性增长，Roaring bitmaps需要打破这个魔咒就一定要用到指数的特性：

将posting list按照65535为界限切分，比如第一块所包含的文档id范围在0~65535之间，第二块的id范围是65536~131071，以此类推。
![Alt text](https://raw.githubusercontent.com/Neway6655/neway6655.github.com/master/img/elasticsearch-study/Roaringbitmaps.png)

细心的小明这时候又举手了:"为什么是以65535为界限?"

程序员的世界里除了1024外，65535也是一个经典值，因为它=2^16-1，正好是用2个字节能表示的最大数，一个short的存储单位，注意到上图里的最后一行“If a block has more than 4096 values, encode as a bit set, and otherwise as a simple array using 2 bytes per value”，如果是大块，用节省点用bitset存，小块就豪爽点，2个字节我也不计较了，用一个short[]存着方便。

那为什么用4096来区分大块还是小块呢？

个人理解：都说程序员的世界是二进制的，4096*2bytes ＝ 8192bytes < 1KB, 磁盘一次寻道可以顺序把一个小块的内容都读出来，再大一位就超过1KB了，需要两次读。

----------

####联合索引

上面说了半天都是单field索引，如果是多个field索引联合查询，倒排索引如何做到呢？

* 利用跳表(Skip list)的数据结构快速做“与”运算，或者
* 利用上面提到的bitset按位“与”

先看看跳表的数据结构：

![Alt text](https://raw.githubusercontent.com/Neway6655/neway6655.github.com/master/img/elasticsearch-study/skiplist.png)

将一个有序链表level0，挑出其中几个元素到level1及level2，每个level越往上，选出来的指针元素越少，查找时依次从高level往低查找，比如55，先找到level2的31，再找到level1的47，最后找到55，一共3次查找，查找效率和2叉树的效率相当，但也是用了一定的空间冗余来换取的。

假设有下面三个posting list需要联合索引：

![Alt text](https://raw.githubusercontent.com/Neway6655/neway6655.github.com/master/img/elasticsearch-study/combineIndex.png)

如果使用跳表，对最短的posting list中的每个id，逐个在另外两个posting list中查找看是否存在，最后得到交集的结果。

如果使用bitset，就很直观了，直接按位与，得到的结果就是最后的交集。

----------

##存储

##聚合运算

##集群

##性能优化/实战

----------

##Reference

[Elasticsearch权威指南](http://www.learnes.net/index.html)

[时间序列数据库的秘密(2)——索引](http://www.infoq.com/cn/articles/database-timestamp-02?utm_source=infoq&utm_medium=related_content_link&utm_campaign=relatedContent_articles_clk)

[^1]: Elasticsearch默认会为每个字段根据value的类型分别建立索引，如果不想为某些字段建立索引或者不做分词分析的话，需要通过FieldMapping注明