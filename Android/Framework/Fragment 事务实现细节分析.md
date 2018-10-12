---
title: Fragment的事务实现细节分析
date: 2015-10-17 10:58:26
tags: [源码]
categories: Android
---

>__Note:__ 本文适合对 Fragment 的用法比较熟悉的读者。

```java
getSupportFragmentManager().beginTransaction()
	                .add(R.id.container, LCFragment.newInstance("I am new!")).addToBackStack(null).commit();
```
对于稍有经验的Android开发者而言，上面的代码都应该很熟悉，通过这段代码我们可以向某个ViewGroup中添加一个Fragment。对于这段代码，我有以下个好奇点: 

1. beginTransaction()应该是开启一个事务，这个事务是什么含义?
2. addToBackStack是将这个变化添加到堆栈中去，我们知道Fragment还可以Pop堆栈，从而恢复回到commit之前的状态，这是如何实现的？

本文通过源码分析，解析Fragment的工作原理，顺便解释清楚以上三个问题。<!--more-->

>__PS:__ Fragment是3.0之后才添加的，为了支持3.0之前的应用开发，google提供了support包，本文的源码分析是基于support-v4版本进行的。

## 入口函数getSupportFragmentManager()
我们操作Fragment都是通过FragmentManager来执行的，在FragmentActivity中找到这个方法的实现，发现它返回的是FragmentManagerImpl的一个实例，我们直接定位到FragmentManager类，FragmentManagerImpl作为它的一个内部类存在。这里就是我们的切入点。

### 那些带有疑问的函数以及不常见的API
1. __public abstract Fragment findFragmentById(@IdRes int id);__  
   读者应该知道两点:1)如果一个Fragment在添加的适合没有指定id，则他的id就等于它的containerId；2）一个ViewGroup可以添加多个Fragment。那么问题来了，这个方法只返回了一个Fragment，在多个Fragment拥有同样的Id的时候，返回的是哪一个？
2. __POP\_BACK\_STACK\_INCLUSIVE__   
   这个参数读者可以阅读注释，或者自行查阅相关资料先进行了解，文章后面会讲述。
3. __public abstract void putFragment(Bundle bundle, String key, Fragment fragment);__  
   这个函数很少用，其注释中说"Put a reference to a fragment in a Bundle."，查看Fragment的实现，很容易发现Fragment本身并没有实现Parcelable接口，name它是如何实现将Fragment保存到Bundle的呢?

这些问题将在后面的分析中一一解答。

## 事务函数beginTransaction()
从FragmentManagerImpl找到这个函数的实现，返回的是一个BackStackRecord实例。这个类实现了接口FragmentTransaction，查看接口声明，很容易就发现这个接口中的方法就是我们常见的操作Fragment的方法，比如add, remove, 还有很重要的commit()和commitAllowingStateLoss()方法。这个类还实现了Runnable接口，具体作用后面讲述。

### BackStackRecord类
粗略的看一下BackStackRecord类代码的实现，可以很容易发现，它定义了8种操作以及一个Op内部类。

#### Op类实现
这个类用于将Add，Remove等这样的方法转换成一个个操作命令：
```java
static final class Op {
	Op next;
	Op prev;
	int cmd;
	Fragment fragment;
	int enterAnim;
	int exitAnim;
	int popEnterAnim;
	int popExitAnim;
	ArrayList<Fragment> removed;
}
```
从类属性看，可以知道这个类记录了命令类型，以及执行命令时的动画(分为push和pop动画)，以及涉及到的相关的Fragment(比如replace方法，就涉及两个Fragment)，另外从最前面两个属性看，Op将成为一个双向链表的节点：实际上就在这个类声明的下面几行，BackStackRecord就声明了mHead和mTail两个变量，佐证了链表的猜测。

#### 操作方法示例解释
从定义的命令来看，FragmentManager支持8种操作，我们选择一种简单的看(主要是代码简短，比较好贴)——remove。
```java
public FragmentTransaction remove(Fragment fragment) {
	Op op = new Op();
	op.cmd = OP_REMOVE;
	op.fragment = fragment;
	addOp(op);
	return this;
}
```
代码很简单，无需过多解释，我们看一下addOp方法的实现:
```java
void addOp(Op op) {
	if (mHead == null) {
		mHead = mTail = op;
	} else {
		op.prev = mTail;
		mTail.next = op;
		mTail = op;
	}
	op.enterAnim = mEnterAnim;
	op.exitAnim = mExitAnim;
	op.popEnterAnim = mPopEnterAnim;
	op.popExitAnim = mPopExitAnim;
	mNumOp++;
}
```
这段代码彻底证明了链表的猜想——新添加的Op被加入了一个链表中。其余的操作也类似，会被转换成一个Op命令，添加到这个列表中。

#### `Commit()`方法
commit和commitAllowingStateLoss()两个方法最终都会调用commitInternal方法，只是传入参数不一样，其实现如下:
```java
int commitInternal(boolean allowStateLoss) {
	if (mCommitted) throw new IllegalStateException("commit already called");
	mCommitted = true;
	if (mAddToBackStack) {
		mIndex = mManager.allocBackStackIndex(this);
	} else {
		mIndex = -1;
	}
	mManager.enqueueAction(this, allowStateLoss);
	return mIndex;
}
```
代码也很简单，commit之后返回的参数是调用FragmentManager的allocBackStackIndex后返回的值(先不探索，稍等转到FragmentManager后再解释)，之后调用FragmentManager的enqueueAction方法，将自己作为参数传入。从enqueueAction这个方法名字看，很容易就能猜到意思:将自己压入一个队列，做什么呢？等待执行。

#### `run()`方法
前面说到BackStackRecord实现了Runnable()接口，作用什么呢？其实这里和线程没有关系，只是为了实现调用类的统一方法来执行一个动作，这个用法和Handler的post(Runnable)方法一样，读者可以先查阅一下FragmentManager的enqueueAction方法签名，它接受的第一个参数就是Runnable，而非BackStackRecord类型。

看一下run方法实现的核心:
```java
Op op = mHead;
while (op != null) {
	int enterAnim = state != null ? 0 : op.enterAnim;
	int exitAnim = state != null ? 0 : op.exitAnim;
	switch (op.cmd) {
		case OP_ADD: {
			Fragment f = op.fragment;
			f.mNextAnim = enterAnim;
			mManager.addFragment(f, false);
			} break;
		//....其余命令
		default: {
			throw new IllegalArgumentException("Unknown cmd: " + op.cmd);
			}
	}
	op = op.next;
}

if (mAddToBackStack) {//加入堆栈
	mManager.addBackStackState(this);
}
```
这是一个典型的链表扫描，这里以OP_ADD命令为例，我们可以看到它最终实现的时候，是调用了FragmentManager的addFragment方法，读者可以查看别的方法，很容易就发现其余的命令实现也是类似。再看方法整体，就是一个遍历扫描链表，执行节点Op的过程。

因此总结如下： __一次beginTransaction()方法会开启一个BackStackRecord实例，后续对Fragment的操作(链式编程)会转化成Op节点以链表形式存储在BackStackRecord中，commit()之后就交给FragmentManager调度执行，而执行的时候，就是遍历链表的Op命令，将命令映射到FragmentManager的函数上，操作Fragment。__

读者注意到这个函数的最后三行代码，这里根据mAddToBackStack判断是否将自己添加到BackStack中去，mAddToBackStack什么时候会为true呢？搜索一下代码就会发现:调用addToBackStack之后就会为true。而且很容易得出结论：只要在commit()之前调用就可以，不一定在操作最后调用。但是为了易读，在所有操作之后、commit之前调用比较好。(还有个类似的方法disallowAddToBackStack，一旦调用了这个方法，再掉用addToBackStack就会抛异常)

FragmentManagerImpl中的addBackStackState方法实现如下:
```java
void addBackStackState(BackStackRecord state) {
	if (mBackStack == null) {
		mBackStack = new ArrayList<BackStackRecord>();
	}
	mBackStack.add(state);
	reportBackStackChanged();
}
```
可以看到，所有的事务都被封装为一个BackStackRecord对象存储到"栈"中(此处栈是用List模拟的)。

### BackStackRecord的执行
前面讲到，commit之后，会把BackStackRecord压入到一个队列中去，现在我们来看一下具体的实现方法：
```java
public void enqueueAction(Runnable action, boolean allowStateLoss) {
	if (!allowStateLoss) {
		checkStateLoss();
	}
	synchronized (this) {
		if (mDestroyed || mActivity == null) {
			throw new IllegalStateException("Activity has been destroyed");
		}
		if (mPendingActions == null) {
			mPendingActions = new ArrayList<Runnable>();
		}
		mPendingActions.add(action);
		if (mPendingActions.size() == 1) {
			mActivity.mHandler.removeCallbacks(mExecCommit);
			mActivity.mHandler.post(mExecCommit);
		}
	}
}
```
在FragmentManager中有一个List维护着所有待执行的Runnable对象，当然也包含BackStackRecord对象，每次执行enqueueAction，都会往List中加入一个enqueueAction，如果添加之后List的大小为1(表明之前List中是空的)，那么会在主Activity的Handler中发一条全新的mExecCommit命令，这个命令是一个Runnable对象，只执行一个函数：execPendingActions。

这个函数会做什么呢？读者看一下代码(比较长，不贴了)，可以看到以下几个步骤：

1. 将List中的Runnable对象都移入mTmpActions数组中，清空List，清空Handler中的mExecCommit命令——表示主线程上完整的对待执行的Runnable对象进行了一次执行操作；
2. 调用每个Runnable对象的run()方法执行(还记得BackStackRecord的run()方法吗？)。

逻辑非常简单。这样一次执行完成，就可以完成一次事务提交了。一些方法比如popBackStack都被打上了asynchronous的标记，就是这个原因：它们不是立刻执行的，都是向队列中添加一个Action，等待主线程去调度执行(试想主线程在执行其余的一些操作，则这些事务必须等待它们执行完成之后才有机会执行)。

在FragmentManager中也有一组方法，它们后面跟着Immediate，表明是立即执行的。那是如何做的呢？这涉及到另外一个函数:executePendingTransactions。这个函数其实是直接调用了execPendingActions，而没有经过Handler，因此可以立即执行，开发人员在外部也可以直接调用该函数。正如方法注释中所写的：这个方法只能从主线程调用。

## 回退——事务出栈
前面讲述了FragmentManager如何执行事务以及存储事务的。那具体的回退呢？

### 事务出栈
FragmentActiivty中写了一个方法，细节如下：
```java
public void onBackPressed() {
	if(!this.mFragments.popBackStackImmediate()) {
		this.finish();
	}
}
```
原来回退键默认的行为就是关闭当前界面，这里做了一层拦截：查看popBackStackImmediate方法的返回值。

### popBackStackState方法
popBackStackState方法最终调用了popBackStackState方法，这个方法极为重要，可以从中得出对之前一些参数详细的解释，下面分片段解释：
```java
if (name == null && id < 0 && (flags & POP_BACK_STACK_INCLUSIVE) == 0) {
	int last = mBackStack.size() - 1;
	if (last < 0) {
        return false;
    }
	final BackStackRecord bss = mBackStack.remove(last);
	SparseArray<Fragment> firstOutFragments = new SparseArray<Fragment>();
	SparseArray<Fragment> lastInFragments = new SparseArray<Fragment>();
	bss.calculateBackFragments(firstOutFragments, lastInFragments);
	bss.popFromBackStack(true, null, firstOutFragments, lastInFragments);
	reportBackStackChanged();
}
```
首先，执行这个If的条件是name和id值都非法并且Flag POP\_BACK\_STACK\_INCLUSIVE没有被设置才会去执行，即默认行为，是弹出最上层一个。从方法片段可以看出，取出mBackStack中的最后一个事务操作记录，调用calculateBackFragments方法。

calculateBackFragments方法的目的是查看哪些Fragment需要被移除、哪些需要被添加(与之前的Op命令相反，比如之前是Add，现在就应该被计入移除队列)，传入参数是两个SparseArray，执行中涉及到两个函数setLastIn和setFirstOut，这两个方法没有特别的逻辑，就是以containerId为键值将需要添加和移除的fragment保存到SparseArray中。

>__Note:__ 以containerId为键值会有问题，原因同样是来自于一个container中会存在多个Fragment。比如setFirstOut实现的时时候，会判断f移除队列中是否存在以container为键值的fragment，没有的话才会去移除。这意味着：一次事务回退是不能从一个container中移除两个Fragment的。但是在添加的时候却可以，两相矛盾。  
>不过不用担心回退栈会出现问题，因为实际上并不是通过这两个SparseArray来进行回退的，这个计算结果最终是为了得到transitionStyle和transition，这边是另外一条线，不做细讲。

#### popFromBackStack方法实现
计算好之后，就会调用BackStackRecord的popFromBackStack方法。这个方法前面小半段是计算transitionStyle和transition，后面半段就比较有意思了:
```java
public TransitionState popFromBackStack(boolean doStateMove, TransitionState state,
	SparseArray<Fragment> firstOutFragments, SparseArray<Fragment> lastInFragments) {
    Op op = mTail;
	while (op != null) {
		int popEnterAnim = state != null ? 0 : op.popEnterAnim;
		int popExitAnim= state != null ? 0 : op.popExitAnim;
		switch (op.cmd) {
		case OP_ADD: {
			Fragment f = op.fragment;
			f.mNextAnim = popExitAnim;
			mManager.removeFragment(f,
			FragmentManagerImpl.reverseTransit(transition), transitionStyle);
			} break;
		case OP_REMOVE: {
			Fragment f = op.fragment;
			f.mNextAnim = popEnterAnim;
			mManager.addFragment(f, false);
			} break;
		//....
		default: {
			throw new IllegalArgumentException("Unknown cmd: " + op.cmd);
			}
		}
		op = op.prev;
	}
	if (mIndex >= 0) {
		mManager.freeBackStackIndex(mIndex);
		mIndex = -1;
	}
	return state;
}
```
这段代码倒叙遍历之前的Op链表，以相反的方式执行Op链表。之前说到BackStackRecord中定义了8中Fragment操作，这8种操作都有互逆性，比如OP_ADD和OP_REMOVE就是相反的，OP_HIDE和OP_SHOW也是相反的，而OP_REPLACE则是一个OP_ADD和一组OP_REMOVE的组合。因此从这个方法的实现看，其实就是一个逆过程，比如之前的Op命令执行的是OP_ADD，这里只需要将这个Op中记录的Fragment移除就好了。通过这种方式，总能顺利的回退栈。

#### popBackStackState方法后半段
继续回到popBackStackState方法，前面只讲了一半，还有一个If分支没有讲述，执行条件是：name或者id不为null其中一个有效或者设置了Flag POP\_BACK\_STACK\_INCLUSIVE。这里可以看到程序是如何解释POP_BACK_STACK_INCLUSIVE这个Flag的，具体细节不解释，总结一下就是：对于popBackStack方法，如果int(flag)为POP\_BACK\_STACK\_INCLUSIVE，则在这个name或者id标识的BackStackRecord以上的BackStackRecord包括自身都会被弹出，否则，自身不会被弹出，只有该Fragment以上的Fragemnt会被弹出。

name的含义很清楚，因为addToBackStack就要求传入一个String，但是id呢？这里可以看到一行代码:  
__if (id >= 0 && id == bss.mIndex)__  
是的，mIndex其实是BackStackRecord在栈中的位置！那么这个mIndex是在什么时候生成的呢？我们回退到commitInternal中，可以在这个方法实现中找到，在提交一个事务的时候，如果需要添加到事务回退栈，则会调用FragmentManager的allocBackStackIndex方法去分配一个Index。这个方法的实现也比较有意思。为了防止读者分心，此处不拓展讲述，读者只需要知道mIndex是BackStackRecord在栈中的位置就好。

因此，我们向事务栈中添加事务的时候，可以指定一个String作为此次添加的标记，当我们回退的时候，我们可以通过这个标记指定回退到某个历史状态，或者，直接通过位置去指定。

id是唯一的，一个位置上只能存储一个BackStackRecord，但是name却不唯一，多个BackStackRecord可以指定同一个BackStackRecord，如果我指定一个name去弹出，会发生什么呢？弹出栈的时候，首先找到name或者id匹配的那个BackStackRecord，如果没有找到，就直接返回，不进行退出操作。如果找到了，并且设置了POP_BACK_STACK_INCLUSIVE，会继续向下搜索，直到最后一个name或者id匹配的BackStackRecord才会结束。接下去后面的处理就和之前的类似了。

以上，是事务回退时的逻辑。

## 总结
综上所述，现在可以解答一开始提出的问题。第3个问题则在前面已经详细描述。

第1个问题：Fragment的事务和数据库中的事务是两个概念。虽然实际上它也是多个原子操作(Op)的组合，但是并不存在其中一个Op失败之后，回退其余Op的控制保证，这个事务目的是为了组合一堆零散的操作，以便于在一次提交中完成复杂的Fragment切换。而对于数据库而言，事务失败之后是需要回退到执行之前的样子的，Fragment的事务失败可以认为界面退出，它没有持久化保存任何数据，也没有后续的操作会受此次事务失败的影响，因此不需要做出保证。

关于Fragment的事务，还有一些细节没有去解读，但是大致的逻辑在此。读者如有补充，欢迎回复，如发现疑问，欢迎斧正。
