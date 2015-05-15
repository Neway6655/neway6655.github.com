---
layout: post
title: "A Story of Concurrent Problem"
description: "A concurrent problem's anaylsis and fix solution"
category: performance, concurrent
tags: [concurrent]
---

近期同事发现一处代码的并发问题，分析了下，觉得挺有意思，顺带记录下来。

闲话少说，上代码(片段):

{% highlight java %}
private List<Long> measurements = new LinkedList<Long>();
...
public void update(long value) {
    measurements.add(value);
}
...
public HistogramMetric calculateMetric() {
    // keep snapshot not to block new metrics update
    List<Long> snapshotList = measurements;
    measurements = new LinkedList<Long>();
    ......
    for (Long value : snapshotList) {
        ...
    }
    ...
}
{% endhighlight %}

简单解释下代码: traffic线程调用update方法不断的往measurements这个list里加值, 每隔段时间(10s)会有另一个线程调用calculateMetric方法计算这个时间段内的最大，最小值，总和，平均值等统计数据。

>问题：程序日志打印```for (Long value : snapshotList) ``` 这行报ConcurrentModificationException错

{% highlight java %}
java.util.ConcurrentModificationException: null
        at java.util.LinkedList$ListItr.checkForComodification(LinkedList.java:761) ~[na:1.6.0_85]
        at java.util.LinkedList$ListItr.next(LinkedList.java:696) ~[na:1.6.0_85]
        at com.***.Histogram.calculateMetric(Histogram.java:70)
{% endhighlight %}

这里第70行就是for那段.

>分析(一): 为什么会在for这里报ConcurrentModificationException的错误?

因为在Java5里对"for/in loop"做的优化, 这个遍历是通过iterator实现的, 而使用iterator遍历时会对集合是否有改动做检查，一旦发现有，则报ConcurrentModificationException的错误。[^1]

>分析(二): 整个calculateMetric方法都没有找到有对snapshotList做修改的操作, 只有一些get, size, isEmpty这样的非修改类的操作, 为什么会报错?

看贴出来部分的代码，容易发现上面的update方法里有对measurements的add操作, 再结合calculateMetric里的这两句:

{% highlight java %}
List<Long> snapshotList = measurements;
measurements = new LinkedList<Long>();
{% endhighlight %}

其实snapshotList是直接从measurements赋值过来的, 问题在这?
..
思考..
..
问题的确是在这里, 这个取snapshot的过程并没有上锁,如果在这两句执行过程中，其他线程执行了前面那个measurements.add()的方法会怎样? 而这个时候的measurements.add就等同于snapshotList.add(因为measurements还没有被重新分配新的list,和snapshotList指向同一个list),所以就有可能出现measurements.add和snapshotList遍历的并发修改问题.

这里其实还有另一个问题,LinkedList.add()操作本身并不是原子的(包括链表指针调整, size加1等操作),所以在calculate的时候取size经常是不准的.

>如何解决?

比较容易想到的做法有:

+ 对取snapshot那段和measurements.add那句通过对同一个object加synchronized,变成两个同步块,相互之间互斥。
+ 不使用LinkedList,而是使用LinkedBlockingQueue替代,通过drainTo方法实现取snapshot的过程。

但注意到最开始对代码的说明:#update方法是由traffic线程调用#,不断更新值,这个并发量非常大,如果每次更新都加锁,而被锁住的代码块本身执行是很快的,这样锁带来的额外开销相对太大,所以好的方案是可以无锁实现.

所以,通过加synchronized得到的性能会比较差,而使用LinkedBlockingQueue,依靠它本身的并发支持,确实把并发的问题都解决了,但个人觉得仍然有再提升的空间(LinkedBlockingQueue内部也是使用ReentrantLock实现的,依然有引入锁的额外开销问题, 但用CAS实现也存在竞争激烈而产生无谓的多次CAS的浪费[^2]).

想到的一个方法是通过CountDownLatch和CAS来解决, 思路:

+ 在update里不引入锁,
+ 尽量减少update里的为了不引入锁而带来的额外操作.(update调用频率远高于calculateMetrics调用频率)

Fix后的代码:[^3]

{% highlight java %}
private List<Long> measurements = new LinkedList<Long>();

private AtomicBoolean mutex = new AtomicBoolean(false);

private CountDownLatch snapshotCountDownLatch;

public void update(long value) {
    try {
        if (snapshotCountDownLatch != null) {
            snapshotCountDownLatch.await();
        }
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    while (mutex.getAndSet(true)) {
        // do nothing.
    }
    measurements.add(value);
    mutex.set(false);
}

public HistogramMetric calculateMetric() {
    // keep snapshot not to block new metrics update
    snapshotCountDownLatch = new CountDownLatch(1);
    while (mutex.getAndSet(true)) {
        // do nothing.
    }
    List<Long> snapshotList = measurements;
    measurements = new LinkedList<Long>();
    mutex.set(false);
    snapshotCountDownLatch.countDown();
    for (Long value : snapshotList) {
        ...
    }
{% endhighlight %}

待续补充:对不同方案的进行压力测试.

若大家有任何comments,欢迎留言讨论:)

[^1]: 对"for/in loop"的详细说明, 请参见[这里](http://www.ibm.com/developerworks/library/j-forin/index.html)
[^2]: 对CAS的理解,[这篇文章](http://www.cnblogs.com/Mainz/p/3546347.html)讲的很详细
[^3]: 此处Fix的代码可能会再更新,优化无止尽.