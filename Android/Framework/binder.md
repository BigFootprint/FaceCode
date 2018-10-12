---
title: Binder总纲
date: 2016-05-16 15:24:14
tags: [源码, Binder]
categories: Android
---

## 写在前面
关于 Binder 的文章，网上已经有非常多的资料，下面推荐几篇：

| 编号   | 文章                                       | 描述                                       |
| ---- | ---------------------------------------- | ---------------------------------------- |
| ①    | [Android Bander设计与实现 - 设计篇](http://blog.csdn.net/universus/article/details/6211589) | 被很多博客推荐的博客，只讲设计不讲代码。介绍了 Binder 机制中的很多概念以及很多关键点的逻辑，非常值得一读，最主要的是里面有一张神图，清楚的展示了相关概念之间的关系。 |
| ②    | [Binder学习指南](http://weishu.me/2016/01/12/binder-index-for-newer/) | 如标题所言，有学习指南，也推荐了第一篇博客，目前还只是停留在 Java 层，有待深入。但是开头的 Linux 系统相关知识的讲解比较有用，另外比喻也非常恰当。 |
| ③    | [红茶一杯话Binder](http://blog.csdn.net/codefly/article/category/1054447) | 这篇文章真正的做到了有图有真相！虽然是讲代码，但是解析图画的太棒了，一目了然，一定要坚持看完上中下三篇，讲解非常精彩。 |
| ④    | [Android进程间通信（IPC）机制Binder简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6618363) | 老罗的文章。首先是分工明确，四篇文章讲述Binder机制的四个要点；其次是讲解的非常细致深入，可以当做字典查看。 |
| ⑤    | [Android深入浅出之Binder机制](http://www.cnblogs.com/innost/archive/2011/01/09/1931456.html) | 邓凡平大神的博客，也是老罗推荐的博客。这篇文章直接从 MediaPlayerService 切入，讲解细致，娓娓道来。建议在看完前面几篇文章之后再来看这篇文章。 |

<!--more-->__因为有大神的文章，才能入得 Binder 的大门一探究竟，非常感谢！__

说实话，几篇文章加起来该讲的也都讲得差不多了。本文只是试着按照自己的思路去梳理一下 Binder 的机制，对上面几篇文章的有引用的地方会标记出来。

这里也省去一些背景介绍，包括为什么选择 Binder、Binder 的设计思想等等，这些在文章①中都有阐述。

## Binder结构
老罗的文章中对此的解析最为清晰，因为他在[学习计划](http://www.cnblogs.com/innost/archive/2011/01/09/1931456.html)中画了个图。图中可以清晰的看到 Binder 分为四个部分:

1. Client —— 服务使用方
2. Server —— 服务提供方
3. Binder Driver —— 底层驱动
4. Service Manager —— 服务管理方

下面这个图是 Android 的系统架构图:

<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/Android-Arc.png" alt="Android系统架构"/></div>

事实上老罗的图是分布在 Libraries 和 Linux Kernel 层的，即 Client、Server以及 Service Manager 三部分属于 Libraries，而Binder Driver属于 Linux Kernel 层。

大部分文章分析 Binder 的时候也是这样，脱离了我们最常接触的 Application Framework 层在讲述，导致不容易理解。但 Binder 的核心逻辑确实不在 Java 层，Java层只是为了方便开发者使用而进行的一层包装，因此我们也先搞清楚以上四个概念，然后再去琢磨Java层。

#### 结构解释
看完文章②关于Linux知识的解释再来看这四个部分的关系，可能理解起来还不是很容易。这里用一种通俗一点的方式来解释一下为什么会形成这四个部分。

首先要知道的一点是：进程不像线程，进程之间的内存是不共享的，这也是为什么我们创造多种多样的方式来实现进程间通讯的原因。以 Android 开发为例，可以想象两个 Activity，有时候在跳转的时候，Intent 需要携带的数据过于复杂，序列化已经不不可能，那该怎么办呢？

一种办法就是将数据设置为静态变量，还有一种办法就是存到 Application 类中（当然还有其他办法）。以上两种办法总结一下无非是找一个两个 Activity 都能访问到的第三方变量来中转数据。对于进程来说，采取的办法也类似：虽然进程之间直接通讯是被禁止的，但是通过内核是可以的。内核是所有进程运行的环境，可以把数据交给内核，让内核发送给对应的进程，对方进程也以同样的方式将结果返回回来。
>内核理解起来太抽象的话，可以想象一个文件，一个文件可以被任何的应用读取，因此我可以让一个应用在文件中写入一段话，另外一个应用去读取这段话，这样也可以完成一次通信。

这就是 Client，Server，Binder Driver 之间的关系，只不过 Client/Server 和 Binder Driver 之间的调用不是一般的方法调用，而是 __系统调用__ 。

那么 Service Manager 又是用来做什么的呢？这一点文章②描述的很清晰。简单来说，一个 Service 启动起来之后，给自己取一个名字，注册到 ServiceManager 中去，告诉 Service Manager：如果有人找"XXXX"，你让它来找我好了，Service Manager 就记下了，这就是__服务的注册过程__。这个时候有个 Client 听说有个叫"XXXX"的 Service 能干某事儿，就问 Service Manager ，有这个叫"XXXX"的 Service 没有？ Service Manager 一查自己的记录表，就找到了这个 Service ，欢快地把它交给了 Client，这就是__服务的查询过程__。Client 拿到 Service 之后就可以享受 Service 提供的服务了。没有 Service Manager，Clients 就不知道去哪里寻找想要的服务。

这么一来，我们就搞清楚了 Binder 机制每个部分的职责以及大体的交互是什么样子的。实际的问题要复杂很多，接下来就是要深究__三个核心问题__:

1. Service Manager 是如何启动并成为服务管理者的？
2. Service 是如何将自己注册到 Service Manager 中去的？
3. Client 是如何寻找对应的 Service 的？

这三个点是 Binder 机制交互的核心，搞清楚这三个点，就可以比较好的理解 Binder 机制了。

### 建议
在开始介绍之前，建议读者先根据文章[Linux 驱动开发](http://www.muzileecoding.com/framework/linux-driver-dev.html)自己动手写个驱动，对Linux系统不陌生的童鞋花不了多少时间，文章中给出的几篇文章链接也建议花时间看一下。了解一个简单驱动是如何编写的，对于后面讲解 Binder 驱动大有裨益。

另外既然要分析源码，代码文件自然少不了，这里推荐一个网站:[AndroidXref](http://androidxref.com/)。可以方便查看多个 Android 版本的系统源代码。

###v声明
在阅读[@悠然红茶](http://blog.csdn.net/codefly/article/category/1054447)的博客时，深感画图的重要性，尤其是函数的调用关系图，在分析复杂系统的时候显得特别有用：可以清晰的定位到目前正在分析哪一块。因此接下去的文章会借用这种方式进行分析。

## Binder 结构体字典
因为 binder 的实现中涉及到非常多的数据结构，本来分析起来方法调用链就很长，临时再去分析数据结构会让分析变得不清楚，因此这里会列出遇到的所有的数据结构以便于读者查询。__在后续的文章中，如果一个结构体列在这里，会标注出来，表明在字典中可查。__

__【注意】__ 这些结构体有的非常复杂，包含别的结构体；有的有些字段含义明确，有的则非常模糊；有的甚至只是与另外一个结构体保持同构。因此在字典中解释全部的字段不现实，更何况有些需要结合代码上下文进行分析，因此这里很多结构体只是简单的枚举，不存在分析。

### `binder_state`
```C++
struct binder_state
{
    int fd;
    void *mapped;
    unsigned mapsize;
};
```
这个结构体有三个属性：fd 代表打开的 binder 驱动的文件描述符，mapped 是内存映射的其实地址，mapsize 是内存映射的大小。

### `binder_proc`
```C++
struct binder_proc {
    struct hlist_node proc_node;
    struct rb_root threads;
    struct rb_root nodes;
    struct rb_root refs_by_desc;
    struct rb_root refs_by_node;
    int pid;
    struct vm_area_struct *vma;
    struct mm_struct *vma_vm_mm;
    struct task_struct *tsk;
    struct files_struct *files;
    struct hlist_node deferred_work_node;
    int deferred_work;
    void *buffer;
    ptrdiff_t user_buffer_offset;
    struct list_head buffers;
    struct rb_root free_buffers;
    struct rb_root allocated_buffers;
    size_t free_async_space;
    struct page **pages;
    size_t buffer_size;
    uint32_t buffer_free;
    struct list_head todo;
    wait_queue_head_t wait; // Linux中很重要的数据结构
    struct binder_stats stats;
    struct list_head delivered_death;
    int max_threads;
    int requested_threads;
    int requested_threads_started;
    int ready_threads;
    long default_priority;
    struct dentry *debugfs_entry;
};
```
这个结构体非常重要，它实际和 Libraries 层的进程一一对应，除了记录进程相关的信息，还创建了很多的属性用于实现 binder 机制相关的功能，这个结构体将被保存在`filp->private_data`中，以后也可以从这个变量中取出 binder_proc 变量。它核心的一组属性是:

```C++
struct rb_root threads; // 记录属于该进程的所有线程
struct rb_root nodes; // 记录属于该进程的所有 binder_node，表示该进程提供的服务
struct rb_root refs_by_desc; // 记录属于该进程引用的服务，节点是 binder_ref，根据 handle 进行查找
struct rb_root refs_by_node; // 同上，但是根据 binder_node 进行查找
```
这是四棵红黑树，后面有注释。

### `binder_thread`
```C++
struct binder_thread {
	struct binder_proc *proc;
	struct rb_node rb_node;
	int pid;
	int looper;
	struct binder_transaction *transaction_stack;
	struct list_head todo;
	uint32_t return_error; /* Write failed, return error code in read buf */
	uint32_t return_error2; /* Write failed, return error code in read */
		/* buffer. Used when sending a reply to a dead process that */
		/* we are also waiting on */
	wait_queue_head_t wait;
	struct binder_stats stats;
};
```
顾名思义，和`binder_proc`类似，这个结构体记录的其实是映射到一个 Libraries 线程。这里可以看到`binder_proc`结构属性，表明这个线程是属于哪个进程的；还有一个属性是 rb_node，还记得前面介绍`binder_proc`时说到它有四棵重要的红黑树么？其中一棵就叫做`threads`，记录的是属于这个进程的所有线程，这里的 rb_node 属性就是指向其中的一个节点；另外 looper 属性记录的是当前线程的状态。

### `binder_node`
```C++
struct binder_node
{
    int debug_id;
    struct binder_work work;
    union {
        struct rb_node rb_node;
        struct hlist_node dead_node;
    };
    struct binder_proc *proc;
    struct hlist_head refs;
    int internal_strong_refs;
    int local_weak_refs;
    int local_strong_refs;
    void __user *ptr;
    void __user *cookie;
    unsigned has_strong_ref:1;
    unsigned pending_strong_ref:1;
    unsigned has_weak_ref:1;
    unsigned pending_weak_ref:1;
    unsigned has_async_transaction:1;
    unsigned accept_fds:1;
    unsigned min_priority:8;
    struct list_head async_todo;
};
```
代表着一个 Service 服务在驱动层的实体节点，驱动层内部通过它便可以找到 Libraries 层的服务实体。它的 cookie 字段记录着服务在 Libraries 层的引用位置。proc则表示该节点所代表的服务属于那个进程。其余的属性主要是引用计数。

### `binder_ref`
```C++
struct binder_ref {
    /* Lookups needed: */
    /*   node + proc => ref (transaction) */
    /*   desc + proc => ref (transaction, inc/dec ref) */
    /*   node => refs + procs (proc exit) */
    int debug_id;
    struct rb_node rb_node_desc;
    struct rb_node rb_node_node;
    struct hlist_node node_entry;
    struct binder_proc *proc;
    struct binder_node *node;
    uint32_t desc;
    int strong;
    int weak;
    struct binder_ref_death *death;
};
```

### `binder_write_read`
```C++
struct binder_write_read {  
    signed long write_size; /* bytes to write */  
    signed long write_consumed; /* bytes consumed by driver */  
    unsigned long   write_buffer;  
    signed long read_size;  /* bytes to read */  
    signed long read_consumed;  /* bytes consumed by driver */  
    unsigned long   read_buffer;  
};  
```
这个结构体并不复杂，只有六个对称的字段：既可以支持读的数据也可以支持写的数据，并且看注释，这个结构体最终应该会传递给 binder 驱动。实际上 __这个结构体只负责数据传输，不负责数据语义，具体 write_buffer 和 read_buffer 里面承载什么数据，代表什么含义，是由上下文环境确定的__。

### `flat_binder_object`
```
/* 
 * This is the flattened representation of a Binder object for transfer 
 * between processes.  The 'offsets' supplied as part of a binder transaction 
 * contains offsets into the data where these structures occur.  The Binder 
 * driver takes care of re-writing the structure type and data as it moves 
 * between processes. 
 */  
struct flat_binder_object {  
    /* 8 bytes for large_flat_header. */  
    unsigned long       type;  
    unsigned long       flags;  
  
    /* 8 bytes of data. */  
    union {  
        void        *binder;    /* local object */  
        signed long handle;     /* remote object */  
    };  
  
    /* extra data associated with local object */  
    void            *cookie;  
}; 
```
注释里面说的很清楚，这个对象就是代表一个 binder 对象，用于在进程之间传输的，在binder transaction中有一个"offsets"字段表明哪些地方存储着该结构体，binder 驱动负责重写结构体中的 type 字段和数据。另外，这个结构体中有一个很重要的 union，这个 union 中有两个字段，分别可以指向一个本地的binder和一个远程的binder，远程binder是通过handle字段来指向的。
>这段话现在听上去不大好理解，因为首先不知道binder transaction代表什么，其次也不明白 binder 为什么要重写这个结构内容。这个在具体分析 binder 驱动的时候会逐个解释。

### `binder_transaction_data`
```C++
struct binder_transaction_data {
	/* The first two are only used for bcTRANSACTION and brTRANSACTION,
	 * identifying the target and contents of the transaction.
	 */
	union {
		size_t	handle;	/* target descriptor of command transaction */
		void	*ptr;	/* target descriptor of return transaction */
	} target;
	
	void		*cookie;	/* target object cookie */
	unsigned int	code;		/* transaction command */

	/* General information about the transaction. */
	unsigned int	flags;
	pid_t		sender_pid;
	uid_t		sender_euid;
	size_t		data_size;	/* number of bytes of data */
	size_t		offsets_size;	/* number of bytes of offsets */

	/* If this transaction is inline, the data immediately
	 * follows here; otherwise, it ends with a pointer to
	 * the data buffer.
	 */
	union {
		struct {
			/* transaction data */
			const void	*buffer;
			/* offsets from buffer to flat_binder_object structs */
			const void	*offsets;
		} ptr;
		uint8_t	buf[8];
	} data;
};
```
有些字段是有注释的，这里最重要的一个字段赋值是`tr.target.handle = handle;`，这个 handle 就是我们需要调用的服务的句柄，当这个服务为 SM 的时候，handle 就为0__。

### `binder_transaction`
```C++
struct binder_transaction {
    int debug_id;
    struct binder_work work;
    struct binder_thread *from;
    struct binder_transaction *from_parent;
    struct binder_proc *to_proc;
    struct binder_thread *to_thread;
    struct binder_transaction *to_parent;
    unsigned need_reply:1;
    /* unsigned is_dead:1; */   /* not used at the moment */
    struct binder_buffer *buffer; /* 该结构体见下面所列 */
    unsigned int    code;
    unsigned int    flags;
    long    priority;
    long    saved_priority;
    uid_t   sender_euid;
};
```

### `binder_buffer`
```C++
struct binder_buffer {
    struct list_head entry; /* free and allocated entries by address */
    struct rb_node rb_node; /* free entry by size or allocated entry */
                /* by address */
    unsigned free:1;
    unsigned allow_user_free:1;
    unsigned async_transaction:1;
    unsigned debug_id:29;
    struct binder_transaction *transaction;
    struct binder_node *target_node;
    size_t data_size;
    size_t offsets_size;
    uint8_t data[0];
};
```

### `binder_work`
```C++
struct binder_work {
    struct list_head entry;
    enum {
        BINDER_WORK_TRANSACTION = 1,
        BINDER_WORK_TRANSACTION_COMPLETE,
        BINDER_WORK_NODE,
        BINDER_WORK_DEAD_BINDER,
        BINDER_WORK_DEAD_BINDER_AND_CLEAR,
        BINDER_WORK_CLEAR_DEATH_NOTIFICATION,
    } type;
};
```

### `binder_txn`
```C++
struct binder_txn
{
    void *target;
    void *cookie;
    uint32_t code;
    uint32_t flags;

    uint32_t sender_pid;
    uint32_t sender_euid;

    uint32_t data_size;
    uint32_t offs_size;
    void *data;
    void *offs;
};
```
binder_transaction_data 的同构体。

### `binder_io`
```C++
struct binder_io
{
    char *data;            /* pointer to read/write from */
    uint32_t *offs;        /* array of offsets */
    uint32_t data_avail;   /* bytes available in data buffer */
    uint32_t offs_avail;   /* entries available in offsets array */

    char *data0;           /* start of data buffer */
    uint32_t *offs0;       /* start of offsets buffer */
    uint32_t flags;
    uint32_t unused;
};
```
这是一个定义很不明确的结构体，功能大体描述如下：可以定义该结构体可以容纳的数据大小、剩余量、起始位置；并可以定义一些特殊数据的偏移量，偏移量数据同样也有起始位置和数据大小。

### `binder_object`
```C++
struct binder_object
{
    uint32_t type;
    uint32_t flags;
    void *pointer;
    void *cookie;
};
```
与 flat_binder_object 结构体对应，是它的反序列化结构体。

### `svcinfo`
```C++
struct svcinfo 
{
    struct svcinfo *next;
    void *ptr;
    struct binder_death death;
    unsigned len;
    uint16_t name[0];
};
```
ptr 记录的是系统service对应的binder句柄值