---
title: Java 动态代理实现
date: 2016-02-28 20:00:39
tags: [源码]
categories: Java

---

## 介绍和使用
在Java中是不能像C/C++那样直接以bit为单位操作数据的，必须想办法替代: 

1. 使用整数数组来模拟bit；
2. 通过已有数据类型，使用位移操作来达到操作bit的效果；

第一种办法虽然简单，但是很耗空间。Java自带的工具类BitSet采用的是第二种方案。

BitSet选取的基础类型是long，内部保存了一个long数组，通过and等操作设置long的某一位的值来模拟bit操作。<!--more-->

BitSet的用法如下:

```java
BitSet set = new BitSet(1024);
set.set(1)
```
以上表示建立一个可以容纳1024个bit的BitSet，并将第2位置为1。BitSet本身自带扩容功能，实现了大量的与bit位相关的操作API，详见[Doc](https://docs.oracle.com/javase/7/docs/api/java/util/BitSet.html)。

## 源码剖析
我们来看一下内部实现。首先看一下实例化过程:

```java
private final static int ADDRESS_BITS_PER_WORD = 6;

public BitSet(int nbits) {
   // nbits can't be negative; size 0 is OK
   if (nbits < 0)
       throw new NegativeArraySizeException("nbits < 0: " + nbits);
 
   initWords(nbits);
   sizeIsSticky = true;
}
 
private void initWords(int nbits) {
   words = new long[wordIndex(nbits-1) + 1];
}

private static int wordIndex(int bitIndex) {
    return bitIndex >> ADDRESS_BITS_PER_WORD;
}
```
这里我把实例化涉及到的相关方法和变量都已经贴出来了，我们看到它实际上就是通过initWords实例化了一个long数组，而数组的大小是通过算式__wordIndex(nbits-1) + 1__算出来的，该算式可以保证计算出来的数组的大小正好可以容纳nbits位数的bit值。

接下来我们讲一些重要的方法。

### `set()`方法
```java
public void set(int bitIndex) {
   if (bitIndex < 0)
       throw new IndexOutOfBoundsException("bitIndex < 0: " + bitIndex);
 
   int wordIndex = wordIndex(bitIndex);
   expandTo(wordIndex);
 
 	//A
   words[wordIndex] |= (1L << bitIndex); // Restores invariants
 
   checkInvariants();
}
```
这个方法就是设置某一个bit为为1。这里分为以下几个步骤:

1. 确定bit位映射到数组中的哪一位，即哪一个long值；
2. 通过expandTo方法(下面会有详细解释)保证long数组的长度可以容纳这一位；
3. 将long的这一位置为1；

其中A处的表达式就是在设置位数，java中的移位操作会模除位数，也就是说，long类型的移位会模除64。例如对long类型的值左移65位，实际是左移了65%64=1位。所以这行代码就等于：

```java
int transderBits = bitIndex % 64;
words[wordsIndex] |= (1L << transferBits);
```

### `clear()`方法
这个方法和set()相对，它的实现是这样的:

```java
public void clear(int bitIndex) {
   if (bitIndex < 0)
       throw new IndexOutOfBoundsException("bitIndex < 0: " + bitIndex);
 
   int wordIndex = wordIndex(bitIndex);
   if (wordIndex >= wordsInUse)
       return;
 
   words[wordIndex] &= ~(1L << bitIndex);
 
   recalculateWordsInUse();
   checkInvariants();
}
```
和set()方法的步骤类似，只有在设置位的时候是不一样的，不多解释。

### `expandTo()`方法
这个方法是用来对BitSet内部的long数组扩容的:

```java
/**
 * The number of words in the logical size of this BitSet.
 */
private transient int wordsInUse = 0;

private void expandTo(int wordIndex) {
    int wordsRequired = wordIndex+1;
    if (wordsInUse < wordsRequired) {
        ensureCapacity(wordsRequired);
        wordsInUse = wordsRequired;
    }
}
```
wordsInUse表示逻辑上words的大小，当我们传进一个wordIndex的时候，首先需要判断这个逻辑大小与wordIndex的大小关系，如果小于它，我们就调用方法ensureCapacity():

```java

private void ensureCapacity(int wordsRequired) {
    if (words.length < wordsRequired) {
		// Allocate larger of doubled size or required size
		int request = Math.max(2 * words.length, wordsRequired);
		words = Arrays.copyOf(words, request);
		sizeIsSticky = false;
	}
}
```
这里判断是否words数组确实小了不够用，不够的话就新建一个两倍大的或者和wordsRequired一样大的数组并copy原来的数据。

### `get()`方法
```java
 public boolean get(int bitIndex) {
    if (bitIndex < 0)
        throw new IndexOutOfBoundsException("bitIndex < 0: " + bitIndex);
 
    checkInvariants();
 
    int wordIndex = wordIndex(bitIndex);
    return (wordIndex < wordsInUse)
        && ((words[wordIndex] & (1L << bitIndex)) != 0);
}
```
只有当wordIndex没有越界，并且wordIndex上的wordIndex上的bit不为0的时候，我们才说这一位是true.

### `size()`方法
```java
private final static int ADDRESS_BITS_PER_WORD = 6;
private final static int BITS_PER_WORD = 1 << ADDRESS_BITS_PER_WORD;

public int size() {
	return words.length * BITS_PER_WORD;
}
```
注意，这里返回的是实际数组可以容纳的bit位数，它永远是64的倍数，而不一定是我们传进来实例化的那个nbits大小，比如你传入1020，这里返回的应该也是1024。

### `length()`方法
这个方法看上去和size()很类似，但是实际的逻辑却不一样:

```java

/**
 * Returns the "logical size" of this <code>BitSet</code>: the index of
 * the highest set bit in the <code>BitSet</code> plus one. Returns zero
 * if the <code>BitSet</code> contains no set bits.
 *
 * @return  the logical size of this <code>BitSet</code>.
 * @since   1.2
 */
public int length() {
	if (wordsInUse == 0)
           return 0;
 
	return BITS_PER_WORD * (wordsInUse - 1) +
		(BITS_PER_WORD - Long.numberOfLeadingZeros(words[wordsInUse - 1]));
   }
```
方法虽然短小，却比较难以理解，细细分析一下：根据注释，这个方法法返回的是BitSet的逻辑大小，比如说你声明了一个129位的BitSet,设置了第23，45，67位，那么其逻辑大小就是67，也就是说逻辑大小其实是的是在你设置的所有位里面最高位的Index。

这里有一个方法，Long.numberOfLeadingZeros，网上没有很好的解释，做实验如下：

```java

long test = 1;
System.out.println(Long.numberOfLeadingZeros(test<<3));
System.out.println(Long.numberOfLeadingZeros(test<<40));
System.out.println(Long.numberOfLeadingZeros(test<<40 | test<<4));
```
打印结果如下:

```java
60
23
23
```
也就是说，这个方法是输出一个64位二进制字符串前面0的个数的。

## 总结
BitSet通过操作long数组来模拟bit操作，虽然可能并没有直接操作bit数组那么快速(毕竟要做一些计算)，但已经是一个很不错的方案了。

