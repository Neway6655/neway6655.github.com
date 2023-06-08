---
layout: post
title: "如何使用bitmap支持范围查询"
description: "bitmap可以高效的支持与或非操作, 是否也能同时支持范围查询呢？"
category: bitmap
tags: [bitmap, range query]
---

在一些标签系统中，经常需要通过标签进行筛选，例如电商里的圈人，圈货都是根据预先打上的标签进行运算，得到最终的结果，由于用户和商品的量级往往非常大，常见的解决方案是使用bitmap存储，每个标签对应一个bitmap，通过对bitmap的压缩（数据往往非常稀疏）既可以降低存储压力，同时又能支持高效的标签运算。然而，在某些查询场景下，例如所有低于99元的男士T恤这样的圈货条件，使用bitmap处理起来就比较费劲，因为价格低于99是一个范围查询，bitmap如何支持呢？看到外网有一个解决方案:[Pilosa](https://www.pilosa.com/blog/range-encoded-bitmaps/)，这篇文章便是对此方案的理解和翻译。

整体思路: 通过Range-Encoded Bitmaps实现范围查找，再通过Bit-sliced Indexes对索引字段进行降维。

## Range-Encoded Bitmaps

还是以上面的价格查询为例子，假设有一系列的商品和他们的价格，如下面的表格所示:

![bitmap example](https://raw.githubusercontent.com/Neway6655/neway6655.github.com/master/images/bitmap-range-query/bitmap_example.jpg)

这是Equal-Encoded Bitmaps, 即可以找到“价格=99”的商品，但如果想找到“价格<=99”的商品，只能通过对所有价格<=99的所有bitmap求并集，可想而知，这样并不高效，特别是价格的值域范围可能非常大。

但通过对bitmap上做些文章，可以轻松实现范围查找，如下图所示:

![range-encoded bitmap](https://raw.githubusercontent.com/Neway6655/neway6655.github.com/master/images/bitmap-range-query/range-bitmap.jpg)

这就是“Range-Encoded Bitmaps”，顾名思义，将每个bitmap的含义改为“<=价格”，即可轻松实现“价格<=99”的商品查找，如果想找出“39<价格<=99”的商品呢？通过”<=99“的bitmap和“<=39”的bitmap进行异或即可。

## Bit-sliced Indexes

一般范围查找的字段，它的值域空间都特别大，比如“价格”，商品的价格全部枚举出来可能有上千个，如果将每个价格都作为一个bitmap，这样存储成本非常高。于是就有了“Bit-sliced Indexes”，什么意思呢？当一个数字太大时，数学上可以用10进制，2进制来表示，例如99这个数字可以用9\*10+9\*1，同样的，可以将一个bitmap拆成多个，以10进制为例:

![bit-sliced indexes bitmap](https://raw.githubusercontent.com/Neway6655/neway6655.github.com/master/images/bitmap-range-query/bit-sliced-indexed-bitmap.jpg)

价格范围在1-99的所有商品，可以通过20个bitmaps表达，而并非需要99个bitmaps，那么“价格=99”的商品怎么查找呢？将“(comp 1) - 9”和“(comp 0) - 9”两个bitmap进行交集即可。

## Range-Encoded Bit-Slice Indexes

终于，将“Range-Encoded bitmaps”和“Bit-Slice Indexes bitmaps”结合起来，即高效的实现了基于bitmap的范围查询:

![range-ecnoded bit-sliced indexes bitmap](https://raw.githubusercontent.com/Neway6655/neway6655.github.com/master/images/bitmap-range-query/range-bit-sliced-indexes-bitmap.jpg)

例如要查询“价格<=78”的商品，用10进制来表示：<=70 ∪ (<=80 ∩ <=8)，即(comp 1) - 7 ∪ ((comp 1) - 8) ∩ (comp 0) - 8)。

若要查询“价格>78”的商品，留给大家可以思考如何运算？