---
title: Java ConcurrentHashMap补充
date: 2015-11-21 15:12:54
tags: [源码]
categories: Java
---

基于[《探索 ConcurrentHashMap 高并发性的实现机制》](https://www.ibm.com/developerworks/cn/java/java-lo-concurrenthashmap/)做总结和补充。

前提: IBM 的这篇文章是基于 JDK 1.6 的，现在JDK已经到 1.8，读者可以分别下载 [1.6](http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/6-b27/java/util/concurrent/ConcurrentHashMap.java#ConcurrentHashMap) 和 [1.8](http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/8u40-b25/java/util/concurrent/ConcurrentHashMap.java#ConcurrentHashMap) 代码阅读。

## 补充一：ConcurrentHashMap 和 Segment的建立
其实文章中已经讲述了该过程，不过条理不清晰。看一下ConcurrentHashMap的构造函数：<!--more-->

```java
public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) {
	if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
	throw new IllegalArgumentException();

	if (concurrencyLevel > MAX_SEGMENTS)
		concurrencyLevel = MAX_SEGMENTS;

	// Find power-of-two sizes best matching arguments
	int sshift = 0;
	int ssize = 1;
	while (ssize < concurrencyLevel) {
		++sshift;
		ssize <<= 1;
	}

	segmentShift = 32 - sshift;
	segmentMask = ssize - 1;
	this.segments = Segment.newArray(ssize);

	if (initialCapacity > MAXIMUM_CAPACITY)
		initialCapacity = MAXIMUM_CAPACITY;
	
	int c = initialCapacity / ssize;
	if (c * ssize < initialCapacity)
		++c;
	int cap = 1;
	while (cap < c)
		cap <<= 1;

	for (int i = 0; i < this.segments.length; ++i)
		this.segments[i] = new Segment<K,V>(cap, loadFactor);
}
```
直接看代码以及文章注释可能都不是很清楚，该构造函数传进来三个参数，分别是什么作用呢？我将构造函数中代码提出来，做了一下实验，赋值和计算结果分别如下：

```java
concurrencyLevel:17
initialCapacity:63
sshift:5
ssize:32
segmentShift:27
segmentMask:31
cap:2
```
这下子就很清楚了，当我们要求concurrencyLevel是17的时候，ssize的计算结果是32，而sssize最终是用于创建Segment数组的，也就是说程序会扩展（这个很常见，扩展成2的幂值），保证实际的Segment数量大于等于开发者设定的并发Level（一个Segment作为一个并发Level看待）。

而initialCapacity值为63的时候，cap的最终计算结果是2。这是因为，当我们分配了ssize数量的Segment之后，我们需要将开发者设定的initialCapacity分配到Segment中去，很明显，至少每个Segment存储2个HashEntry才能满足63的的初始容量设置，因此这个值等于2。

基于以上认识，配合注释，就不难理解代码逻辑了。下面这幅图很重要:

![ConcurrentHashMap结构图](http://7xktd8.com1.z0.glb.clouddn.com/ConcurrentHashMap结构图.jpg)

通过将容器切分为多个Segment，降低锁的粒度，从而提高并发性——现在Segment之间是独立的，读写可以并发，具体的锁操作只有到了单个Segment内部才会发生。

## 补充二：readValueUnderLock
这个方法在两个方法被调用，一个是containsValue，一个是get。都是在获取某个HashEntry的value发现为null之后再去调用。文章中把它的解释和volatile放在一起，容易产生误解，这个方法的注释是：

```java
/**
 * Reads value field of an entry under lock. Called if value
 * field ever appears to be null. This is possible only if a
 * compiler happens to reorder a HashEntry initialization with
 * its table assignment, which is legal under memory model
 * but is not known to ever occur.
 */
V readValueUnderLock(HashEntry<K,V> e) {
	lock();
	try {
		return e.value;
	} finally {
		unlock();
	}
}
```
根据注释，发生这种情况很有可能是因为编译器重排序了一个HashEntry的初始化和赋值过程。即原本应该是先初始化再存入table，而实际上却因为重排序，导致HashEntry没有初始化完成就被存入了table(发生在get方法中)。

而在代码中，readValueUnderLock和put方法都是使用lock()包围的，因此根据happens-before原则使用readValueUnderLock可以强制和put确立先后关系，等待put完成之后再去获取相应的value。

正如文章后面所说:
>在实际的应用中，散列表一般的应用场景是：除了少数插入操作和删除操作外，绝大多数都是读取操作，而且读操作在大多数时候都是成功的。正是基于这个前提，ConcurrentHashMap针对读操作做了大量的优化——只有读到value域的值为null时，读线程才需要加锁后重读，极大的减少了持锁时间。通过HashEntry对象的不变性和用volatile型变量协调线程间的内存可见性，使得 大多数时候，读操作不需要加锁就可以正确获得值。这个特性使得ConcurrentHashMap 的并发性能在分离锁的基础上又有了近一步的提高。

## 补充三：结构性修改的操作
在文章中提到了对散列表做非结构性修改的操作和对散列表做结构性修改的操作。非结构性修改的可见性由volatile字段完成，而结构性修改的操作，实际上作者讨论的并没有太大意义，因为put、remove、clear三个操作在发生对链表的修改时，都是在lock状态下进行的。

## 补充四：为什么性能好？
降低锁的粒度和锁持有时间，并考虑到实际的使用情况，对集合读取的情况进行了优化，非常赞！

//TODO 研究JDK 1.8。JDK1.6中HashMap采用的是位桶+链表的方式，即我们常说的散列链表的方式，而JDK1.8中采用的是位桶+链表/红黑树的方式，也是非线程安全的。当某个位桶的链表的长度达到某个阀值的时候，这个链表就将转换成红黑树。