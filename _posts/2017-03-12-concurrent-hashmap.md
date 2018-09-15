---
layout: post
title:  "ConcurrentHashMap 1.7和1.8的不同实现"
date:   2016-07-10 23:06:05
categories: Java
tags: java HashMap ConcurrentHashMap jdk8 集合 基础
author: sqp
---

* content
{:toc}

>在多线程环境下，使用HashMap进行put操作时存在丢失数据的情况，为了避免这种bug的隐患，JDK5中添加了新的concurrent包,相对同步容器而言，并发容器通过一些机制改进了并发性能
因为同步容器将所有对容器状态的访问都串行化了，这样保证了线程的安全性，所以这种方法的代价就是严重降低了并发性，当多个线程竞争容器时，吞吐量严重降低。因此JDK5开始针对多线程并发访问设计，提供了并发性能较好的并发容器。  

>因此并发情况下强烈建议使用ConcurrentHashMap代替HashMap，为了对ConcurrentHashMap有更深入的了解，本文将对ConcurrentHashMap1.7和1.8的不同实现进行分析。  

## 1.7实现

### 数据结构
jdk1.7中采用Segment + HashEntry的方式进行实现，结构如下：  
![avator](http://pf1gfkwtz.bkt.clouddn.com/18-9-15/76199483.jpg)
Segment在实现上继承了ReentrantLock，这样就自带了锁的功能,ConcurrentHashMap初始化时，计算出Segment数组的大小ssize和每个Segment中HashEntry数组的大小cap，并初始化Segment数组的第一个元素；其中ssize大小为2的幂次方，默认为16，cap大小也是2的幂次方，最小值为2，最终结果根据根据初始化容量initialCapacity进行计算，计算过程如下：  

``` java
if (c * ssize < initialCapacity)
    ++c;
int cap = MIN_SEGMENT_TABLE_CAPACITY;
while (cap < c)
    cap <<= 1;
```

### put实现

当执行put方法插入数据时，根据key的hash值，在Segment数组中找到相应的位置，如果相应位置的Segment还未初始化，则通过CAS进行赋值，接着执行Segment对象的put方法通过加锁机制插入数据，实现如下：  

场景：线程A和线程B同时执行相同Segment对象的put方法  
1. 线程A执行tryLock()方法成功获取锁，则把HashEntry对象插入到相应的位置；  
2. 线程B获取锁失败，则执行scanAndLockForPut()方法，在scanAndLockForPut方法中，会通过重复执行tryLock()方法尝试获取锁，在多处理器环境下，重复次数为64，单处理器重复次数为1，当执行tryLock()方法的次数超过上限时，则执行lock()方法挂起线程B；  
3. 当线程A执行完插入操作时，会通过unlock()方法释放锁，接着唤醒线程B继续执行；

### size实现

因为ConcurrentHashMap是可以并发插入数据的，所以在准确计算元素时存在一定的难度，一般的思路是统计每个Segment对象中的元素个数，然后进行累加，但是这种方式计算出来的结果并不一样的准确的，因为在计算后面几个Segment的元素个数时，已经计算过的Segment同时可能有数据的插入或则删除，在1.7的实现中，采用了如下方式：

先采用不加锁的方式，连续计算元素的个数，最多计算3次：  
1. 如果前后两次计算结果相同，则说明计算出来的元素个数是准确的；  
2. 如果前后两次计算结果都不同，则给每个Segment进行加锁，再计算一次元素的个数；

具体代码如下：  
``` java
try {
    for (;;) {
        if (retries++ == RETRIES_BEFORE_LOCK) {
            for (int j = 0; j < segments.length; ++j)
                ensureSegment(j).lock(); // force creation
        }
        sum = 0L;
        size = 0;
        overflow = false;
        for (int j = 0; j < segments.length; ++j) {
            Segment<K,V> seg = segmentAt(segments, j);
            if (seg != null) {
                sum += seg.modCount;
                int c = seg.count;
                if (c < 0 || (size += c) < 0)
                    overflow = true;
            }
        }
        if (sum == last)
            break;
        last = sum;
    }
} finally {
    if (retries > RETRIES_BEFORE_LOCK) {
        for (int j = 0; j < segments.length; ++j)
            segmentAt(segments, j).unlock();
    }
}
```

## 1.8实现

### 数据结构
1.8中放弃了Segment臃肿的设计，而是直接采用Node + CAS + Synchronized来保证并发安全进行实现。对链表的查询进行了优化，当链表过长时采用红黑树的结构代替链表，将查询的时间复杂度由O(n)降到O(log<sub>2</sub>n)。其结构如下：
![avator](http://pf1gfkwtz.bkt.clouddn.com/18-9-15/69028987.jpg)


### put实现  
当执行put方法插入数据时，如果key hash计算后该位置的Node数组不存在则会调用initTable()初始化Node数组，实现如下：

``` java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

根据key的hash值，在Node数组中找到相应的位置，实现如下：  
1. 如果相应位置的Node还未初始化，则通过CAS插入相应的数据；  
2. 如果相应位置的Node不为空，且当前该节点不处于移动状态，则对该节点加synchronized锁，如果该节点的hash不小于0，则遍历链表更新节点或插入新节点；  
3. 如果该节点是TreeBin类型的节点，说明是红黑树结构，则通过putTreeVal方法往红黑树中插入节点；  
4. 如果binCount不为0，说明put操作对数据产生了影响，如果当前链表的个数达到8个，则通过treeifyBin方法转化为红黑树，如果oldVal不为空，说明是一次更新操作，没有对元素个数产生影响，则直接返回旧值；
5. 如果插入的是一个新节点，则执行addCount()方法尝试更新元素个数baseCount；

### size实现  

1.8中使用一个volatile类型的变量baseCount记录元素的个数，当插入新数据或则删除数据时，会通过addCount()方法更新baseCount，实现如下：

``` java
if ((as = counterCells) != null ||
    !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
    CounterCell a; long v; int m;
    boolean uncontended = true;
    if (as == null || (m = as.length - 1) < 0 ||
        (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
        !(uncontended =
          U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
        fullAddCount(x, uncontended);
        return;
    }
    if (check <= 1)
        return;
    s = sumCount();
}
```
1. 初始化时counterCells为空，在并发量很高时，如果存在两个线程同时执行CAS修改baseCount值，则失败的线程会继续执行方法体中的逻辑，使用CounterCell记录元素个数的变化；  
2. 如果CounterCell数组counterCells为空，调用fullAddCount()方法进行初始化，并插入对应的记录数，通过CAS设置cellsBusy字段，只有设置成功的线程才能初始化CounterCell数组。  
3. 如果通过CAS设置cellsBusy字段失败的话，则继续尝试通过CAS修改baseCount字段，如果修改baseCount字段成功的话，就退出循环，否则继续循环插入CounterCell对象，所以在1.8中的size实现比1.7简单多，因为元素个数保存baseCount中，部分元素的变化个数保存在CounterCell数组中，
通过累加baseCount和CounterCell数组中的数量，即可得到元素的总个数；