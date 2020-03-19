---
title: HashMap探究以及ConcurrentHashMap
tags:
  - HashMap
categories:
  - Java
mathjax: true
copyright: true
comment: true
abbrlink: 582417c6
date: 2017-11-02 15:48:18
photo:
---

HashMap是一种结构非常巧妙的集合容器。HashMap是存储键值对的结构，key和value一一对应，key必须保证唯一，存储无序，线程不安全，这是它的一些基本特点。在java中，最基本的结构就是两种，一个是数组，另外一个是模拟指针（引用），所有的数据结构都可以用这两个基本结构来构造的，hashmap也不例外。
<!-- more -->

Hashmap实际上是一个数组和链表的结合体（在数据结构中，一般称之为“链表散列“），如下图所示 ![Hashmap构成](http://dl2.iteye.com/upload/attachment/0017/7479/3f05dd61-955e-3eb2-bf8e-31da8a361148.jpg)

当新建一个HashMap的时候，会初始化一个数组。

```java
    //The table, resized as necessary. Length MUST Always be a power of two. 
    //FIXME 这里需要注意这句话，至于原因后面会讲到  
    transient Entry[ ] table;  
    static class Entry<K,V> implements Map.Entry<K,V> {  
            final K key;  
            V value;  
            final int hash;  
            Entry<K,V> next;   
    }
```

Entry<K,V>就是存储在数组中的元素，它包含了一个指向下个元素的引用，构成了一个单向链表。 当我们往hashmap中put元素的时候，先根据key的hash值得到这个元素在数组中的位置（即下标），然后就可以把这个元素放到对应的位置中了。如果这个元素所在的位子上已经存放有其他元素了，根据equals方法进行比较，有相等的则覆盖掉，没有相当的那么将以链表的形式存放，新加入的放在链头，最先加入的放在链尾。从hashmap中get元素时，首先计算key的hashcode，找到数组中对应位置的某一元素，然后通过key的equals方法在对应位置的链表中找到需要的元素。

### 1.Hash算法

我们可以看到在hashmap中要找到某个元素，需要根据key的hash值来求得对应数组中的位置。如何计算这个位置就是hash算法。前面说过hashmap的数据结构是数组和链表的结合，所以如果hashmap里面的元素位置尽量的分布均匀些，尽量使得每个位置上的元素数量只有一个，那么查找的时候无需遍历链表，速度就会很快。Java中是这样实现的

```java
    static int indexFor(int h, int length) {  
           return h & (length-1);  
    }
```

首先算得key得hashcode值，然后跟数组的长度-1做一次“与”运算（&）。HashMap默认的初始化数组长度是16，并且无论创建多大的HashMap，它的数组长度都是2的n次方，这是为何呢？ 看下图，左边两组是数组长度为16（2的4次方），右边两组是数组长度为15。两组的hashcode均为8和9，但是很明显，当它们和1110“与”的时候，产生了相同的结果，也就是说它们会定位到数组中的同一个位置上去，这就产生了碰撞，8和9会被放到同一个链表上，那么查询的时候就需要遍历这个链表，得到8或者9，这样就降低了查询的效率。同时，我们也可以发现，当数组长度为15的时候，hashcode的值会与14（1110）进行“与”，那么最后一位永远是0，而0001，0011，0101，1001，1011，0111，1101这几个位置永远都不能存放元素了，空间浪费相当大，更糟的是这种情况中，数组可以使用的位置比数组长度小了很多，这意味着进一步增加了碰撞的几率，减慢了查询的效率！ ![](http://dl2.iteye.com/upload/attachment/0017/7481/4b3732d6-fb5f-369b-b50d-e8b8325c69d4.jpg) 

因此，当数组的长度为2的n次方的时候，能够尽量保证元素分布均匀。那么我们想要存放1000个元素，new HashMap(1000),java会自动将HashMap的数组长度设为1024，这样到底合适吗？实际上并不合适，HashMap中还有一个装载因子LoadFactor。

### HashMap的扩容

HashMap含有一个阈值，当达到这个阈值，HashMap就会扩容，重新计算每一个元素的Hash，放到新的位置，默认扩容一倍。扩容也是为了减少元素碰撞的机率，阈值=数组长度xLoadFactor，默认LoadFactor为0.75。因此，回到上面，当我们想要存储1000个元素的时候，如果设置HashMap大小为1024，那么阈值为1024x0.75=768，即超过768就会发生扩容，会将数组长度扩大到2048，然后重新计算每个元素在数组中的位置，这是一个非常消耗资源的操作，因此正确的做法是应该new HashMap(2048)比较合适。我们说道HashMap是不安全的，它的不安全也主要体现在扩容的时候。

### HashMap的安全问题

在多线程操作时，扩容的时候如果多线程都进行reHash操作，那么有可能造成链表中的元素相互引用，即A的下一个指向B，B的下一个指向A，造成了死循环，从而导致线程不安全。如何让HashMap变成安全的？有三种方法。 
1. 替换成Hashtable，Hashtable通过对整个表上锁实现线程安全，因此效率比较低 
2. 使用Collections类的synchronizedMap方法包装一下。方法如下：
```java
public static <K,V> Map<K,V> synchronizedMap(Map<K,V> m)
```
3. 使用ConcurrentHashMap，它使用分段锁来保证线程安全 通过前两种方式获得的线程安全的HashMap在读写数据的时候会对整个容器上锁，而ConcurrentHashMap并不需要对整个容器上锁，它只需要锁住要修改的部分就行了。 CHM引入了分割，并提供了HashTable支持的所有的功能。在CHM中，支持多线程对Map做读操作，并且不需要任何的[blocking](http://javarevisited.blogspot.com/2012/02/what-is-blocking-methods-in-java-and.html)。这得益于CHM将Map分割成了不同的部分，在执行更新操作时只锁住一部分。根据默认的并发级别(`concurrency level`)，Map被分割成16个部分，并且由不同的锁控制。这意味着，同时最多可以有16个写线程操作Map。试想一下，由只能一个线程进入变成同时可由16个写线程同时进入(读线程几乎不受限制)，性能的提升是显而易见的。但由于一些更新操作，如put(),remove(),putAll(),clear()只锁住操作的部分，所以在检索操作不能保证返回的是最新的结果。 另一个重要点是在迭代遍历CHM时，keySet返回的iterator是弱一致和[fail-safe](http://javarevisited.blogspot.in/2012/02/fail-safe-vs-fail-fast-iterator-in-java.html)的，可能不会返回某些最近的改变，并且在遍历过程中，如果已经遍历的数组上的内容变化了，不会抛出ConcurrentModificationExceptoin的异常。 CHM默认的并发级别是16，但可以在创建CHM时通过构造函数改变。毫无疑问，并发级别代表着并发执行更新操作的数目，所以如果只有很少的线程会更新Map，那么建议设置一个低的并发级别。另外，CHM还使用了ReentrantLock来对segments加锁。

### 什么时候使用ConcurrentHashMap

CHM适用于读者数量超过写者时，当写者数量大于等于读者时，CHM的性能是低于Hashtable和synchronized Map的。这是因为当锁住了整个Map时，读操作要等待对同一部分执行写操作的线程结束。CHM适用于做cache,在程序启动时初始化，之后可以被多个请求线程访问。正如Javadoc说明的那样，CHM是HashTable一个很好的替代，但要记住，CHM的比HashTable的同步性稍弱。

*   CHM允许并发的读和线程安全的更新操作
*   在执行写操作时，CHM只锁住部分的Map
*   并发的更新是通过内部根据并发级别将Map分割成小部分实现的
*   高的并发级别会造成时间和空间的浪费，低的并发级别在写线程多时会引起线程间的竞争
*   CHM的所有操作都是线程安全
*   CHM返回的迭代器是弱一致性，fail-safe并且不会抛出ConcurrentModificationException异常
*   CHM不允许null的键值
*   可以使用CHM代替HashTable，但要记住CHM不会锁住整个Map

参考：http://www.importnew.com/21388.html http://www.importnew.com/21388.html