---
title: ConcurrentModificationException
date: 2015-10-26 22:29:27
tags: [源码]
categories: Java
---

Java中的 ConcurrentModificationException 是 Fail-Fast 的一种表现，这里有[一篇文章](http://javapapers.com/core-java/fail-fast-vs-fail-safe/)探讨 Fail-Fast 和 Fail-Safe 谁更好。

这里我们以 Java 中 ArrayList 的 Iterator (它有好几个 Iterator，这个最简单，但能概括问题)实现为例，看看什么时候会发生 ConcurrentModificationException 。以下为 ArrayList 的 Iterator 实现源码：<!--more-->

```java
private class Itr implements Iterator<E> {
	int cursor;       // index of next element to return
	int lastRet = -1; // index of last element returned; -1 if no such
	int expectedModCount = modCount;

	public boolean hasNext() {
		return cursor != size;
	}

	@SuppressWarnings("unchecked")
	public E next() {
		checkForComodification();//第一种
		int i = cursor;
		if (i >= size)
			throw new NoSuchElementException();
		Object[] elementData = ArrayList.this.elementData;
		if (i >= elementData.length)
			throw new ConcurrentModificationException();
		cursor = i + 1;
		return (E) elementData[lastRet = i];
	}

	public void remove() {
		if (lastRet < 0)
			throw new IllegalStateException();
		checkForComodification();//第一种

		try {
			ArrayList.this.remove(lastRet);
			cursor = lastRet;
			lastRet = -1;
			expectedModCount = modCount;
		} catch (IndexOutOfBoundsException ex) {
			throw new ConcurrentModificationException();
		}
	}

	@Override
	@SuppressWarnings("unchecked")
	public void forEachRemaining(Consumer<? super E> consumer) {
		Objects.requireNonNull(consumer);
		final int size = ArrayList.this.size;
		int i = cursor;
		if (i >= size) {
			return;
		}

		final Object[] elementData = ArrayList.this.elementData;
		if (i >= elementData.length) {
			throw new ConcurrentModificationException();//第二种
		}
		
		while (i != size && modCount == expectedModCount) {
			consumer.accept((E) elementData[i++]);
		}
		// update once at end of iteration to reduce heap write traffic
		cursor = i;
		lastRet = i - 1;
		checkForComodification();//第一种
	}

	final void checkForComodification() {
		if (modCount != expectedModCount)
			throw new ConcurrentModificationException();
	}
}
```
可以看到有如下几种情况会抛出这个异常：

1. next()和remove()方法一开始会调用checkForComodification()方法，这个方法主要是比较modCount和expectedModCount，这两个参数分别代表什么呢？前者代表集合本身的方法修改集合的次数：读者可以自行查看ArrayList的源码，在remove、add、trim等方法中都可以看到modCount被++。而expectedModCount则在一开始就赋值为modCount，而在Iterator中的remove方法中，又被赋值为modCount，由此可见，如果使用集合的remove、add等方法修改集合元素，则必然引起modCount增加，导致modCount != expectedModCount成立，在下次调用next()或者remove()方法的时候，就会抛出ConcurrentModificationException，这里要注意是下次：如果修改之后不调用这两个方法，就不会checkForComodification，也就不会引发异常。而Iterator本身的remove()方法在执行完毕后因为会重新赋值expectedModCount，因此后续是不会出问题的。
2. 第二类是一种判断，从代码看，可以简单的认为是一种集合越界：cursor(指向下一个返回值)指向的元素不存在，需要remove的元素不存在等等，我们可以猜测一下为什么会发生这种情况？比较明显的一点是：所有的remove、add方法都没有同步措施，因此在多线程中很容易发生这种情况：一个线程在按序遍历，另外一个线程在随机删除数据，导致越界错误；

以上，会导致我们在使用ArrayList的Iterator的时候抛出ConcurrentModificationException，可以认为Iterator在检测到任何可能发生的错误或者不一致的情况的时候，都应该抛出这个异常。

但是正如Java API中所提到的:
>“Note that the fail-fast behavior of an iterator cannot be guaranteed as it is, generally speaking, impossible to make any hard guarantees in the presence of unsynchronized concurrent modification. Fail-fast iterators throw ConcurrentModificationException on a best-effort basis. Therefore, it would be wrong to write a program that depended on this exception for its correctness: the fail-fast behavior of iterators should be used only to detect bugs.”

这些方法并没有做好同步工作，只能说是一种辅助预防，即使是Iterator本身的方法也是没有做同步的，这就埋下一些隐患。举个简单的例子：  
Iterator的remove方法中有一行代码__expectedModCount = modCount;__这行代码是重新赋值expectedModCount，假设这个Iterator被一个线程A获取，并执行到这一行，线程A被切换，则无论别的线程调用什么方法添加集合元素，当执行回到线程A的时候，都不会导致后续发生问题，因为expectedModCount会被重新赋值。