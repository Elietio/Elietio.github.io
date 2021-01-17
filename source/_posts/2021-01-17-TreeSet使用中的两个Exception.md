---
title: TreeSet使用中的两个Exception
categories: Java
tags:
  - TreeSet
  - IllegalArgumentException
  - ConcurrentModificationException
copyright: true
comment: true
abbrlink: 264bd07b
date: 2021-01-17 12:48:26
description:
---

最近的一个业务场景中需要在内存中换成一些数据,并且需要根据时间戳有序排列,因此使用了TreeSet,但是在使用过程中确出现了IllegalArgumentException和ConcurrentModificationException,因此记录一下这两个问题.

<!-- more -->

## IllegalArgumentException

首先是

```java
java.lang.IllegalArgumentException: fromKey > toKey
```

我们的一个业务场景是需要对内存中的一些数据根据指定区间筛选出对应数据排列好之后返回给前端,因此我们选用了有序集合TreeSet,当前端传入一个区间范围[A,B]之后,可以使用`E ceiling(E e)`返回大于等于A的最小元素,使用`E floor(E e)`返回小于等于B的最大元素,这样就可以使用subSet就可以返回指定区间的set集合

```java
NavigableSet<E> subSet(E fromElement, boolean fromInclusive,
                 E toElement,   boolean toInclusive) 
```

这样看来,这个思路并没有什么问题.但是代码部署之后,测试环境缺偶尔抛出了`java.lang.IllegalArgumentException: fromKey > toKey`异常,简而言之就是fromElement大于toElement,无法返回指定范围的set,由于入参的时候已经对于A、B大小已经进行了校验,那么只能是使用ceiling和floor方法导致返回的元素出现了问题,查看了原数据和传入的区间参数之后发现了这样一个问题,例如TreeSet存储以下元素

```java
[1,3,7,8,9]
```

而传入的区间为[4,6],`ceiling(4)`返回7,而`floor(6)`返回3,那么`subSet(7,3)`会抛出IllegalArgumentException也就不奇怪了,因此,当使用subSet方法时一定要确保fromElement小于toElement,加上这个二次校验之后,这个问题就再也没出现过了.但是过了不久之后又出现了另外一个问题

## ConcurrentModificationException

```java
java.util.ConcurrentModificationException
```

这个异常也很清晰,由于并发安全问题导致的,多线程同时修改或者同时读取和修改都会导致这个问题.

我们知道,iterator遍历元素时通过源列表直接删除元素会导致ConcurrentModificationException,必须使用iterator的remove方法才能安全删除元素,使用Iterator会返回集合自身的一个迭代器,这个机制与这两个字段有关

- expectedModCount 预期被修改的次数,属于迭代器`私有`,初始时和modCount相等
- modCount 集合被结构性修改(新增或删除)的次数,它是属于集合的

当进行遍历/删除时都会判断modCount和expectedModCount是否相等,不等就会抛出ConcurrentModificationException,而只有使用迭代器的remove方法才会更新expectedModCount值确保二者相等.以下相关源码

```java
final Entry<K,V> nextEntry() {
		 Entry<K,V> e = next;
     if (e == null)
         throw new NoSuchElementException();
     if (modCount != expectedModCount)
         throw new ConcurrentModificationException();
     next = successor(e);
     lastReturned = e;
     return e;
           
}

public void remove() {
    if (lastReturned == null)
        throw new IllegalStateException();
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
    // deleted entries are replaced by their successors
    if (lastReturned.left != null && lastReturned.right != null)
        next = lastReturned;
    deleteEntry(lastReturned);
    expectedModCount = modCount;
    lastReturned = null;
}
```

所以必须要求Iterator迭代过程中必须使用Iterator的remove方法,但是这仅限于单线程,当多线程情况下,即使使用Iterator的remove方法仍然会有线程安全问题,因为迭代器是线程私有的,所以expectedModCount也是线程私有的,而modCount是线程共享的.如果有一个线程对集合进行了修改,那么modCount和此线程的expectedModCount会更新,但是其他线程的expectedModCount都不会更新,expectedModCount!=modCount,最终抛出ConcurrentModificationException.

我们正是犯了了这个问题,多线程同时读取和修改,导致产生了线程安全问题.所以需要将TreeSet换成线程安全的有序Set集合SynchronizedSortedSet.

 java.util 包中的集合类都返回 fail-fast 迭代器,具有强一致性,当迭代器检测到迭代过程中元素进行了更改,就会抛出ConcurrentModificationException.而java.util.concurrent包中的集合类返回的是weakly consistent迭代器,即弱一致迭代器,当迭代开始,如果元素在迭代到达前被删除或者修改,这些更改会返回给调用者,但是对于插入元素则无法保证,并且不会抛出ConcurrentModificationException.

