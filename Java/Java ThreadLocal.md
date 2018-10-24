---
title: ThreadLocal源码剖析
date: 2016-02-27 16:28:03
tags: [源码]
categories: Java
---

## 介绍
当一个资源可能被多个线程同时访问的时候，我们就需要对资源加以保护，以确定在该资源上的操作是线程安全的。常见的方法就是加锁（synchronized 或者 Lock），或者将操作原子化（比如 AtomicInteger）。

ThreadLocal 是解决这类问题的另外一种方案: 既然资源在线程间共享会有安全问题，那么就不用共享了，每一个线程有一份资源副本就好了。

## 使用
下面是一个🌰:

```java
package com.footprint.test;

public class ThreadLocalTest {
    public static ThreadLocal<Integer> intLocal = new ThreadLocal<Integer>() {
        @Override
        protected Integer initialValue() {
            return 0;
        }

        @Override
        public Integer get() {
            return super.get();
        }

        @Override
        public void set(Integer value) {
            super.set(value);
        }

        @Override
        public void remove() {
            super.remove();
        }
    };

    public static void main(String[] args) {
        for(int index = 0; index < 3; index ++)
            new MyThread(index).start();
    }
}

class MyThread extends Thread{
    int id;

    public MyThread(int id){
        this.id = id;
    }

    @Override
    public void run() {
        for(int index = 0; index < 5; index ++){
            int value = ThreadLocalTest.intLocal.get();
            System.out.println("Thread-" + id + " : " + value);
            ThreadLocalTest.intLocal.set(++value);
            try {
                Thread.sleep((int)(100 * Math.random()));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```
这个例子基于一个很典型的问题: 银行存钱取钱问题。我们知道当多线程并发操作一个 int 值的加减操作的时候，最后的数值会产生很大的不确定性，得不到最终正确的结果。这里我们将操作的“存款”对象由普通的 int 值转变为 ThreadLocal<Integer> 对象，我们看一下运行结果:

```java
Thread-0 : 0
Thread-2 : 0
Thread-1 : 0
Thread-0 : 1
Thread-1 : 1
Thread-1 : 2
Thread-2 : 1
Thread-1 : 3
Thread-0 : 2
Thread-1 : 4
Thread-2 : 2
Thread-0 : 3
Thread-2 : 3
Thread-2 : 4
Thread-0 : 4
```
虽然 5 个线程操作的是同一个 ThreadLocal 对象，但每一个线程的数字都是在自我递增，线程之间互不干扰，这就是 ThreadLocal 的作用。

ThreadLocal类中可重载的方法只有四个：

1. set()：设置值，也就是说，我们选择将某个值设置为ThreadLocal类型的；
2. get()：将设置进去的值取出来；
3. remove()：我们不想将某个值设置为 ThreadLocal 了，移除掉；
4. initialValue()：如果 get 的时候还没有设置值，就使用这个方法进行初始化；

一般重载 initialValue() 提供一个初始值就可以了，其余方法不需要重载。

## ThreadLocal 源码剖析
本节从源码角度看一下四个方法是如何配合运作的，又是如何做到为每一个线程分配一个资源副本的。

我们首先从`get()`方法入手：

```java
public T get() {
	//获取当前线程
	Thread t = Thread.currentThread();
	ThreadLocalMap map = getMap(t);
	if (map != null) {
		ThreadLocalMap.Entry e = map.getEntry(this);
		if (e != null)
			return (T)e.value;
	}
	return setInitialValue();
}

ThreadLocalMap getMap(Thread t) {
	return t.threadLocals;
}
```
getMap 方法返回的是线程的 threadLocals 对象，这个对象是什么含义呢？

```java
/* ThreadLocal values pertaining to this thread. This map is maintained
 * by the ThreadLocal class. */
ThreadLocal.ThreadLocalMap threadLocals = null;
```
如上所示，它是一个 ThreadLocalMap 对象，专门为 ThreadLocal 而生且实现和维护都在 ThreadLocal 中进行，只不过 Thread 对它有一个引用。它的注释如下:

```java
/**
 * ThreadLocalMap is a customized hash map suitable only for
 * maintaining thread local values. No operations are exported
 * outside of the ThreadLocal class. The class is package private to
 * allow declaration of fields in class Thread.  To help deal with
 * very large and long-lived usages, the hash table entries use
 * WeakReferences for keys. However, since reference queues are not
 * used, stale entries are guaranteed to be removed only when
 * the table starts running out of space.
 */
```
它是一个定制的 HashMap 容器，它只能在 ThreadLocal 内部使用，所有的 Key 都是WeakReferences，我们来看一下这个 Map 的 Entry：

```java
static class Entry extends WeakReference<ThreadLocal> {
	/** The value associated with this ThreadLocal. */
	Object value;

	Entry(ThreadLocal k, Object v) {
		super(k);
		value = v;
	}
}
```

通过 Entry 可以很清楚的了解到 ThreadLocalMap 维护的是一个从 ThreadLocal 到对应值的映射，每一个线程内部都有一个这样的变量，这意味着一个 Thread 可以维护很多的 ThreadLocal，也就是说可以维护很多资源的副本。

我们回到`get()`方法。`getMap()`方法就是获取当前线程的 ThreadLocalMap 对象，剩下的步骤如下:

```java
if (map != null) {
	ThreadLocalMap.Entry e = map.getEntry(this);
	if (e != null)
	return (T)e.value;
}
```
只有当当前线程的 Map 中维护着当前 ThreadLocal 映射的值的时候，才会正常返回结果，否则就会去调用setInitialValue() 方法:

```java
/**
 * Variant of set() to establish initialValue. Used instead
 * of set() in case user has overridden the set() method.
 *
 * @return the initial value
 */
private T setInitialValue() {
	T value = initialValue();// 调用方法初始化值
	Thread t = Thread.currentThread();
	ThreadLocalMap map = getMap(t);
	if (map != null)// Map 已经存在直接设置值
		map.set(this, value);
	else// 否则创建 Map
		createMap(t, value);
	return value;
}

// 为当前线程创建 ThreadLocalMap
void createMap(Thread t, T firstValue) {
	t.threadLocals = new ThreadLocalMap(this, firstValue);
}

/**
 * Construct a new map initially containing (firstKey, firstValue).
 * ThreadLocalMaps are constructed lazily, so we only create
 * one when we have at least one entry to put in it.
 */
ThreadLocalMap(ThreadLocal firstKey, Object firstValue) {
	table = new Entry[INITIAL_CAPACITY];
	int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
	//传入的 Key-Value 会被立刻创建为 Entry 保存起来
	table[i] = new Entry(firstKey, firstValue);
	size = 1;
	setThreshold(INITIAL_CAPACITY);
}
```
理解了 ThreadLocalMap 的含义以及它与 Thread 的关系，这里就很好理解了，详见注释。

到这里，`get()`和`initialValue()`方法都已经涉及到，剩下的`set()`和`remove()`也很好理解，不再赘述。

## 问题
分析至此，我们来讨论一个问题: ThreadLocal 会不会造成内存泄露？举个🌰: Thead 通过 ThreadLocalMap 引用着 TheadLocal 到一个变量值的映射，假设如果一个线程有A、B两个方法，A方法调用的时候初始化了一个 ThreadLocal 资源，然后线程一直在执行 B 方法，并且 B 不会再去操作 ThreadLocal 资源了，那么会不会引起这个 ThreadLocal 资源不能释放呢？

ThreadLocal 的实现很容易让人觉得不会，比如 ThreadLocalMap 的 Entry 实现。读者应该注意到 Entry 的 Key 也就是对 ThreadLocal 的引用是 WeakReference 的！这样保证了当外部类（ThreadLocalTest）被销毁后，一旦 ThreadLocal 变量（intLocal）不再有强引用，ThreadLocal 就会立刻被回收掉，即 Entry 中的 Key 会被回收掉。但 TheadlLocal 仅仅是一个桥梁，真正的资源是 Value，它还是被 Entry 引用着，保存在 Thread 的ThreadLocalMap 中。

所以，类似于代码🌰中给出的实现，是有风险的。__当我们确认线程中不会再使用一个资源的时候，我们应当主动去 remove__，比如在前面的例子中，A 方法就应该调用 remove 方法，这样可以使得不使用的资源尽早释放掉。否则就如注释中所说:

```java
/* Each thread holds an implicit reference to its copy of a thread-local
 * variable as long as the thread is alive and the <tt>ThreadLocal</tt>
 * instance is accessible; after a thread goes away, all of its copies of
 * thread-local instances are subject to garbage collection (unless other
 * references to these copies exist).
*/
```
等到 Thread 死亡之后才能把资源副本全部释放掉。

## 总结
日常开发中并不常用到这个类，但是它提供了解决资源安全的一种方案，便利且优雅，对于其使用，Android 开发者可以去研习一下 [Handler的实现](../Android/Framework/Android Handler.md)。