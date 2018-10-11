---
title: Binder之Service Manager
date: 2016-06-21 22:27:28
tags: [源码, Binder]
categories: Android
---

## 简介
__Service Manager (以下简称 SM )在 Binder 框架中负责的是维护 Service 的查询工作：给我一个名字，给你想要的服务。__要注意的是：这里所说的 SM 并不是我们日常开发中用到的 SM，那个是 Java 层的封装，这里所讲的 SM 位于 Libraries 层。

__【涉及文件】__

| 文件                   | 位置                                       |
| -------------------- | ---------------------------------------- |
| service_manager.c    | /frameworks/base/cmds/servicemanager/service_manager.c |
| Libraries 层 binder.c | /frameworks/base/cmds/servicemanager/binder.c |
| 驱动层 binder.c         | kernel/common/drivers/staging/android/binder.c |

<!--more-->
##主要函数调用
SM 的启动 main 函数位于"service_manager.c"文件中:

```C++
int main(int argc, char **argv)
{
    struct binder_state *bs;
    // #define BINDER_SERVICE_MANAGER ((void*) 0) 
    void *svcmgr = BINDER_SERVICE_MANAGER;
    // 1. 打开 Binder 驱动
    bs = binder_open(128*1024);
    
    // 2. 注册成为 SM
    if (binder_become_context_manager(bs)) {
        LOGE("cannot become context manager (%s)\n", strerror(errno));
        return -1;
    }

    svcmgr_handle = svcmgr;
    // 3. 启动循环监听
    binder_loop(bs, svcmgr_handler);
    return 0;
}
```
主要步骤就如代码中注释的那样，分为三步:
<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/service_manager_main.png" width="320" alt="SM main函数调用"/></div>

我们就从这三步入手，逐步解析 SM 的启动过程。

## 打开 Binder 驱动
`binder_open`，顾名思义就是打开 binder 驱动，来看看它在打开时做了什么:

```C++
//调用代码: bs = binder_open(128*1024);
struct binder_state *binder_open(unsigned mapsize)
{
	 // 该结构可在字典中查询
    struct binder_state *bs;

    bs = malloc(sizeof(*bs));
    if (!bs) {
        errno = ENOMEM;
        return 0;
    }

	 // 1. 打开驱动
    bs->fd = open("/dev/binder", O_RDWR);
    if (bs->fd < 0) {
        fprintf(stderr,"binder: cannot open device (%s)\n",
                strerror(errno));
        goto fail_open;
    }

    bs->mapsize = mapsize;
    // 2. 内存映射
    bs->mapped = mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0);
    if (bs->mapped == MAP_FAILED) {
        fprintf(stderr,"binder: cannot map device (%s)\n",
                strerror(errno));
        goto fail_map;
    }

        /* TODO: check version */

    return bs;

fail_map:
    close(bs->fd);
fail_open:
    free(bs);
    return 0;
}
```
如注释所示，主要步骤分为两步，我们一步一步来看。

### 驱动层`binder_open`
程序 binder 驱动设备调用的是`open`方法，这个方法实际会调用到驱动层的`binder_open`方法，将打开的文件描述符记录到 binder_state 的 fd 字段: 

>至于为什么会有 Libraries 的`open`方法映射到驱动层的`binder_open`方法，这属于驱动知识，不详细解释，后面也会有很多这样的映射例子，一般是前面加上`binder_`做映射，这个关系是 binder 驱动自己定义的。

```
static int binder_open(struct inode *nodp, struct file *filp)
{
    //声明proc的数据结构，可在字典中查询
    struct binder_proc *proc;
    proc = kzalloc(sizeof(*proc), GFP_KERNEL);
    if (proc == NULL)
        return -ENOMEM;

    //增加引用计数
    get_task_struct(current);
    
    //记录当前进程到proc中
    proc->tsk = current;

    //初始化链表
    INIT_LIST_HEAD(&proc->todo);
    //初始化等待队列
    init_waitqueue_head(&proc->wait);
    //设置优先级
    proc->default_priority = task_nice(current);
    binder_lock(__func__);
    binder_stats_created(BINDER_STAT_PROC);
    //把binder_proc添加到binder_procs链表中去
    hlist_add_head(&proc->proc_node, &binder_procs);
    //记录进程ID
    proc->pid = current->group_leader->pid;
    INIT_LIST_HEAD(&proc->delivered_death);

    //保存进程信息
    filp->private_data = proc;
    binder_unlock(__func__);
    return 0;
}
```
分配`binder_proc`对象之后，后面的代码都有注释，这里 current 其实是一个宏，指向的是一个内联函数，返回的结果指向当前的进程，具体在 SOF 的问题 [what is the “current” in linux kernel source](http://stackoverflow.com/questions/12434651/what-is-the-current-in-linux-kernel-source) 下面有详细的解释。接着就是初始化一些队列，记录环境值。最后会将 proc 保存到`filp->private_data`中去，后面就可以从这个字段中再把 proc 读取出来。
>这里涉及到很多的C++、Linux、驱动知识，比如将 proc 保存到 `filp->private_data` 中去。

这个就是打开 binder 驱动做的主要工作，以下是调用路径图:
<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/binder_open.png" alt="打开binder驱动"/></div>

### 驱动层`binder_mmap`
接着程序将传入的 mapsize 记录到 binder_state 的 mapsize 字段( main 函数传入的值是 128*1024)；最后调用`mmap`函数进行内存映射。`mmap`是系统调用，可以理解为从驱动中划分一块内存出来映射到本进程中，从而 binder 驱动和本进程都可以操作这块内存。

这块较复杂，功力不足暂时分析不了，如果读者有兴趣可以去看老罗的文章[浅谈Service Manager成为Android进程间通信（IPC）机制Binder守护进程之路](http://blog.csdn.net/luoshengyang/article/details/6621566)，直接搜 "binder_mmap" 就好。

至此，binder 驱动打开，并且做好了映射，终于可以直接和驱动通信了。
>这个过程类似于打开了一个文件，定位好了写入位置和可以写入的数据量，接下去就可以写入数据了。

## 成为服务管理者
现在我们已经打开了 binder 驱动，并且记录下它的文件描述符，接着 service_manager 开始申请成为服务管理者了，它调用的方法是`binder_become_context_manager`。这个方法同样存在于 Libraries 层 "binder.c" 文件中:

```C++
int binder_become_context_manager(struct binder_state *bs)
{
    //#define BINDER_SET_CONTEXT_MGR  _IOW('b', 7, int) 
    return ioctl(bs->fd, BINDER_SET_CONTEXT_MGR, 0);
}
```
这里只有一行很简单的调用。`ioctl`也是一个系统调用，最终这个调用会映射到 binder 驱动的 `binder_ioctl`。

这么一来就进入了驱动层，`binder_ioctl`是一个较大的方法，它可以处理很多的命令。调用这个方法的第二个参数就代表命令，而上面的调用传进来的是`BINDER_SET_CONTEXT_MGR `，我们就先看这个命令的处理过程:

```C++
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
	 // 通过 filp->private_data 获取进程
    struct binder_proc *proc = filp->private_data;
    // 该结构体在字典中可查
    struct binder_thread *thread;
    
    //获取到线程
    thread = binder_get_thread(proc);
    if (thread == NULL) {
        ret = -ENOMEM;
        goto err;
    }
    
    switch (cmd) {
    case BINDER_SET_CONTEXT_MGR:
        // binder_context_mgr_node是一个全局静态变量
        if (binder_context_mgr_node != NULL) {
            printk(KERN_ERR "binder: BINDER_SET_CONTEXT_MGR already set\n");
            ret = -EBUSY;
            goto err;
        }
        ret = security_binder_set_context_mgr(proc->tsk);
        if (ret < 0)
            goto err;
        if (binder_context_mgr_uid != -1) {
            if (binder_context_mgr_uid != current->cred->euid) {
                printk(KERN_ERR "binder: BINDER_SET_"
                       "CONTEXT_MGR bad uid %d != %d\n",
                       current->cred->euid,
                       binder_context_mgr_uid);
                ret = -EPERM;
                goto err;
            }
        } else
            binder_context_mgr_uid = current->cred->euid;
        // 新建binder_node
        binder_context_mgr_node = binder_new_node(proc, NULL, NULL);
        if (binder_context_mgr_node == NULL) {
            ret = -ENOMEM;
            goto err;
        }
        binder_context_mgr_node->local_weak_refs++;
        binder_context_mgr_node->local_strong_refs++;
        binder_context_mgr_node->has_strong_ref = 1;
        binder_context_mgr_node->has_weak_ref = 1;
        break;
    default:
        ret = -EINVAL;
        goto err;
    }
    ret = 0;
err:
    if (thread)
        thread->looper &= ~BINDER_LOOPER_STATE_NEED_RETURN;
    return ret;
}
```
__注意:__ 这里是删除了很多代码的，只把核心相关的部分展示出来了。

我们来分析一下代码，首先是调用`binder_get_thread`方法查找 binder_thread:

```C++
static struct binder_thread *binder_get_thread(struct binder_proc *proc)
{
    struct binder_thread *thread = NULL;
    struct rb_node *parent = NULL;

    // threads: binder_proc进程内用于处理用户请求的线程组成的红黑树(关联binder_thread->rb_node)
    struct rb_node **p = &proc->threads.rb_node;
    // 遍历红黑树寻找线程节点
    while (*p) {
        parent = *p;
        thread = rb_entry(parent, struct binder_thread, rb_node);
        if (current->pid < thread->pid)
            p = &(*p)->rb_left;
        else if (current->pid > thread->pid)
            p = &(*p)->rb_right;
        else
            break;
    }

    // 没找到就新建，初始化线程
    if (*p == NULL) {
        thread = kzalloc(sizeof(*thread), GFP_KERNEL);
        if (thread == NULL)
            return NULL;
        binder_stats_created(BINDER_STAT_THREAD);
        thread->proc = proc;

        // 记录进程的pid
        thread->pid = current->pid;
        // 初始化thread的两个队列
        init_waitqueue_head(&thread->wait);
        INIT_LIST_HEAD(&thread->todo);
        // 挂载节点到 &proc->threads 红黑树上
        rb_link_node(&thread->rb_node, parent, p);
        rb_insert_color(&thread->rb_node, &proc->threads);
        // 设置状态
        thread->looper |= BINDER_LOOPER_STATE_NEED_RETURN;
        thread->return_error = BR_OK;
        thread->return_error2 = BR_OK;
    }
    return thread;
}
```
传递进来的是当前进程的记录对象`binder_proc`，具体的解释已经标注在代码中了。这里标注一个__【疑点】__：按照这里的算法，这棵红黑树是否只可能有一个thread？

>这里给出挂载红黑树的两个操作的解释:
>
>```C++
>// 它把 parent 设为 node 的父结点，并且让 rb_link 指向 node。
>static inline void rb_link_node(struct rb_node * node, struct rb_node * parent, struct rb_node ** rb_link);
>
>// 它把已确定父结点的 node 结点融入到以 root 为根的红黑树中
>void rb_insert_color(struct rb_node *node, struct rb_root *root);
>```

OK，回到前面，获取到 binder_thread 之后，就进入命令`BINDER_SET_CONTEXT_MGR`的处理阶段，首先是如下一个判断:

```C++
if (binder_context_mgr_node != NULL) {
	printk(KERN_ERR "binder: BINDER_SET_CONTEXT_MGR already set\n");
	ret = -EBUSY;
	goto err;
}
```
`binder_context_mgr_node`是一个全局变量，它的声明如下:

```C++
// 该结构体在字典中可查
static struct binder_node *binder_context_mgr_node;
```
它是一个全局静态的 binder_node 结构体，这里的判断是保证全局只有一个 binder_context_mgr_node，也就是说只有一个 SM。第一次进来该变量没有新建过，即该变量为 NULL，所以会去调用方法`binder_new_node`新建：

```C++
// 调用代码：binder_context_mgr_node = binder_new_node(proc, NULL, NULL);
static struct binder_node *binder_new_node(struct binder_proc *proc,
                       void __user *ptr,
                       void __user *cookie)
{
    struct rb_node **p = &proc->nodes.rb_node;
    struct rb_node *parent = NULL;
    struct binder_node *node;
    while (*p) {
        parent = *p;
        node = rb_entry(parent, struct binder_node, rb_node)
        if (ptr < node->ptr)
            p = &(*p)->rb_left;
        else if (ptr > node->ptr)
            p = &(*p)->rb_right;
        else
            return NULL;
    }
    node = kzalloc(sizeof(*node), GFP_KERNEL);
    if (node == NULL)
        return NULL;
    binder_stats_created(BINDER_STAT_NODE);
    rb_link_node(&node->rb_node, parent, p);
    rb_insert_color(&node->rb_node, &proc->nodes);
    node->debug_id = ++binder_last_id;
    node->proc = proc;
    node->ptr = ptr;
    node->cookie = cookie;
    node->work.type = BINDER_WORK_NODE;
    INIT_LIST_HEAD(&node->work.entry);
    INIT_LIST_HEAD(&node->async_todo);
    binder_debug(BINDER_DEBUG_INTERNAL_REFS,
             "binder: %d:%d node %d u%p c%p created\n",
             proc->pid, current->pid, node->debug_id,
             node->ptr, node->cookie);
    return node;
}
```
这里第一行代码就遇到了 binder_proc 中的第二棵红黑树 nodes。nodes 节点记录的是属于该进程的所有 binder_node 节点，前面说了，binder_node 节点可以看做是一个服务实体，因此这棵树上记录的就是该进程提供的所有服务。新建完成之后就是一些赋值操作以及挂载 binder_node 的动作，注意，这里传进来的 ptr 和 cookie 都是 NULL。
>后面就会知道 cookie 为 NULL 其实表明 Libraries 层没有对应的服务。这是 SM 这个服务的特殊性。

扯到哪了？哦，对了，我们已经在驱动层为 SM 创建了一个全局的服务实体，以下是调用关系图：
<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/binder_become_context_manager2.png" alt="打开binder驱动"/></div>

接下去是对强引用、弱引用的维护，这个涉及到服务实体的释放，我们不需要太关心。上面两个步骤是差不多的路子，都是从 Libraries 层走到驱动层，接下去还有一个步骤，还得把这个过程再走一遍。

## 启动循环监听
再回到 service_manager 的`main`函数，SM 程序调用`binder_loop`进入循环监听:

```C++
void binder_loop(struct binder_state *bs, binder_handler func)
{
    int res;
    // 该结构体在字典中可查
    struct binder_write_read bwr;
    unsigned readbuf[32];

    bwr.write_size = 0;
    bwr.write_consumed = 0;
    bwr.write_buffer = 0;
    
    readbuf[0] = BC_ENTER_LOOPER;
    //告诉Binder驱动程序， Service Manager要进入循环了
    binder_write(bs, readbuf, sizeof(unsigned));

    for (;;) {
        bwr.read_size = sizeof(readbuf);
        bwr.read_consumed = 0;
        bwr.read_buffer = (unsigned) readbuf;

        res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);

        if (res < 0) {
            LOGE("binder_loop: ioctl failed (%s)\n", strerror(errno));
            break;
        }

        res = binder_parse(bs, 0, readbuf, bwr.read_consumed, func);
        if (res == 0) {
            LOGE("binder_loop: unexpected reply?!\n");
            break;
        }
        if (res < 0) {
            LOGE("binder_loop: io error %d %s\n", res, strerror(errno));
            break;
        }
    }
}
```
这个函数虽然看上去比较简单，但是要分析的东西非常多。

### 线程状态更改
`for`循环前一段代码稍微有点凌乱，整理之后可以看出首先执行的是:

```C++
unsigned readbuf[32];

readbuf[0] = BC_ENTER_LOOPER;
binder_write(bs, readbuf, sizeof(unsigned));
```
bs 是之前声明的`binder_state`结构体，这里把`BC_ENTER_LOOPER`传递进去了，我们直接去看`binder_write`函数是怎么使用这些参数的:

```C++
// 调用代码：binder_write(bs, readbuf, sizeof(unsigned))
int binder_write(struct binder_state *bs, void *data, unsigned len)
{
    struct binder_write_read bwr;
    int res;
    bwr.write_size = len;
    bwr.write_consumed = 0;
    bwr.write_buffer = (unsigned) data;
    bwr.read_size = 0;
    bwr.read_consumed = 0;
    bwr.read_buffer = 0;
    res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);
    if (res < 0) {
        fprintf(stderr,"binder_write: ioctl failed (%s)\n",
                strerror(errno));
    }
    return res;
}
```
这里用到了 binder_write_read 结构体，在字典中有提到：这个结构体只是负责传输数据，具体语义由上下文环境自行定义。这里就可以很明显的看出来这种玩法：刚刚传进来的 data 字段被赋值到了 bwr 的 write 部分了，即 write_buffer 字段被赋予了`BC_ENTER_LOOPER`命令，并且 write_consumed 也被赋予了0，表示 binder 驱动没有读取任何数据，通过 wite_consumed 和 write_size 两个字段，就可以指定 write_buffer 中的数据哪一段是有效的，binder 驱动会根据这两个值读取有效数据。数据以及位置设定好之后，再调用`ioctl`方法，这个方法我们前面有过一次分析，它也会携带一个命令，这次携带的是`BINDER_WRITE_READ`，到对应驱动函数中看一下这个命令的处理方式:

```C++
// 调用代码：res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
    struct binder_proc *proc = filp->private_data;

    unsigned int size = _IOC_SIZE(cmd);
    void __user *ubuf = (void __user *)arg;

    switch (cmd) {
    case BINDER_WRITE_READ: {
        struct binder_write_read bwr;
        // 验证数据格式是否正确
        if (size != sizeof(struct binder_write_read)) {
            ret = -EINVAL;
            goto err;
        }
        if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {
            ret = -EFAULT;
            goto err;
        }

        if (bwr.write_size > 0) {
            ret = binder_thread_write(proc, thread, (void __user *)bwr.write_buffer, bwr.write_size, &bwr.write_consumed);
            trace_binder_write_done(ret);
            if (ret < 0) {
                bwr.read_consumed = 0;
                if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
                    ret = -EFAULT;
                goto err;
            }
        }
        
        if (bwr.read_size > 0) {
            ret = binder_thread_read(proc, thread, (void __user *)bwr.read_buffer, bwr.read_size, &bwr.read_consumed, filp->f_flags & O_NONBLOCK);
            trace_binder_read_done(ret);
            if (!list_empty(&proc->todo))
                wake_up_interruptible(&proc->wait);
            if (ret < 0) {
                if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
                    ret = -EFAULT;
                goto err;
            }
        }
        binder_debug(BINDER_DEBUG_READ_WRITE,
                 "binder: %d:%d wrote %ld of %ld, read return %ld of %ld\n",
                 proc->pid, thread->pid, bwr.write_consumed, bwr.write_size,
                 bwr.read_consumed, bwr.read_size);
        
        if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {
            ret = -EFAULT;
            goto err;
        }
        break;
    }
    ret = 0;
err:
    if (thread)
        thread->looper &= ~BINDER_LOOPER_STATE_NEED_RETURN;
    binder_unlock(__func__);
    wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);
    if (ret && ret != -ERESTARTSYS)
        printk(KERN_INFO "binder: %d:%d ioctl %x %lx returned %d\n", proc->pid, current->pid, cmd, arg, ret);
err_unlocked:
    trace_binder_ioctl_done(ret);
    return ret;
}
```
我们来开动小火车分析 case 的代码。首先是一个判断，这是判断数据格式是否正确。接着调用函数`copy_from_user`，这个函数的 user 是用户空间的意思，它的目的是从用户空间拷贝数据到内核空间。

我们调用`binder_ioctl`的时候传入的第三个参数是`&bwr`，也就是一个 binder_write_read 结构体，这里就是将这个结构体拷贝到驱动层的 bwr 结构体中。
>读者应当熟悉这种模式，这很类似于网络请求。Http请求定义了八种方法，最常见的是Post，Get两种，这就类似于该函数的 cmd 参数，而具体的语义其实是由请求内容确定的，这包括API结构，比如我的接口叫做`getorder.api`，伴随着这个接口发出去的还有一些数据，这些就类似于 binder_write_read 结构体了。处理完成之后，也在该请求上返回数据。
>
>后面还会见到这种模式的反复使用，它是 binder 机制 Libraries 层和驱动层通信的一种典型模式。 

理所应当的，接下来就是解析数据的过程。因为传递参数中只有 write 部分有数据，那么接下去的两个判断只有下面这个会执行:

```C++
if (bwr.write_size > 0) {
	ret = binder_thread_write(proc, thread, (void __user *)bwr.write_buffer, bwr.write_size, &bwr.write_consumed);
	trace_binder_write_done(ret);
	if (ret < 0) {
		bwr.read_consumed = 0;
		if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
			ret = -EFAULT;
		goto err;
	}
}
```
这里调用了`binder_thread_write`来处理 binder_write_read 结构体的内容，这个函数也是一个非常复杂的函数，因为它需要解析该结构体中可能传输过来的所有的命令，这里我们只看其中一部分，也就是处理命令`BC_ENTER_LOOPER`的地方:

```C++
int binder_thread_write(struct binder_proc *proc, struct binder_thread *thread,
            void __user *buffer, int size, signed long *consumed)
{
    uint32_t cmd;
    void __user *ptr = buffer + *consumed;
    void __user *end = buffer + size;
    while (ptr < end && thread->return_error == BR_OK) {
        if (get_user(cmd, (uint32_t __user *)ptr))//读取cmd
            return -EFAULT;

        ptr += sizeof(uint32_t);
        trace_binder_command(cmd);
        if (_IOC_NR(cmd) < ARRAY_SIZE(binder_stats.bc)) {
            binder_stats.bc[_IOC_NR(cmd)]++;
            proc->stats.bc[_IOC_NR(cmd)]++;
            thread->stats.bc[_IOC_NR(cmd)]++;
        }
        switch (cmd) {
        case BC_ENTER_LOOPER://Service将要进入Loop状态时，发送此消息
            if (thread->looper & BINDER_LOOPER_STATE_REGISTERED) {
                thread->looper |= BINDER_LOOPER_STATE_INVALID;
                binder_user_error("binder: %d:%d ERROR:"
                    " BC_ENTER_LOOPER called after "
                    "BC_REGISTER_LOOPER\n",
                    proc->pid, thread->pid);
            }
            //表示进入enter状态，looper用于记录状态
            thread->looper |= BINDER_LOOPER_STATE_ENTERED;
            break;
        }
        // 注意!!!
        *consumed = ptr - buffer;
    }
    return 0;
}
```
这里的入参 buffer 指向的是我们在结构体 binder_write_read 中写入的数据，也就是:

```C++
bwr.write_buffer = (unsigned) data;
```
而 data 就是`BC_ENTER_LOOPER`，所以 ptr 实际是指向这段数据的开始，end 指向的是这段数据的末尾。`get_user`也是一个内核函数，这里可以认为就是从 ptr 开始读取一个 uint32_t 值存储到 cmd 中，那么这个 cmd 就是`BC_ENTER_LOOPER`，接着指针后移。
>从这里的读取方式来看，可以猜到这是一种非常固定的"命令+数据"的模式，是两层之间约定俗称的协议。

之后就进入处理阶段，字典里面说到 binder_thread 的 looper 字段记录是线程当前的状态，这里主要进行的就是状态设置，在检测完状态是否正确之后，调用以下语句:

```C++
thread->looper |= BINDER_LOOPER_STATE_ENTERED;
```
设置状态`ENTERED`，就表示要进入循环状态了。讲完这个再回到`binder_ioctl`方法去，还没完呢，正常执行完上面的方法之后，会走到这段代码中去:

```
if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {
	ret = -EFAULT;
	goto err;
}
```
`copy_to_user`看名字就知道和`copy_from_user`函数什么关系了：这个函数是从内核空间拷贝数据到用户空间。这里拷贝的还是 binder_write_read 结构体，要把内核空间的修改反映到 Libraries 层：这里唯一的改动就是 write_consumed 的指针的移动，这个指针在读完数据之后，被执行了如下操作:

```C++
ptr += sizeof(uint32_t);
*consumed = ptr - buffer;
```
ptr 原本指向 buffer 数据的开始，根据上面的写法，实际上`*consumed`就等于`sizeof(uint32_t)`，也就是`BC_ENTER_LOOPER`的长度，这就表示驱动已经读取了 write 部分的数据。

好了到这里总算分析完成`binder_write(bs, readbuf, sizeof(unsigned));`这一句了。它只做一件事情：告诉驱动层我要进入循环监听状态了。读者务必应熟悉这种交互方式，这里是最简单清楚的一个例子。这里是调用关系图：
<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/binder_write.png" alt="告诉binder驱动即将进入循环监听"/></div>

### 循环 & 监听
终于进入正题了，前面都是状态准备而已。在分析代码之前，可以先从猜测一下它会怎么做？到目前为止，状态都已经设置完毕，驱动层和当前进程也有共享内存来传输数据，那比较合理的做法就是去读取这块共享内存，看是否有数据可以处理，如果有则读出来进行处理，如果没有，则阻塞，直到有数据写入，被驱动层唤醒再返回。那么实际上是不是这样的呢？Read the fucking code !

`binder_loop`函数接下去的是一个无限循环，再贴一下代码:

```C++
// 调用代码：readbuf[0] = BC_ENTER_LOOPER;
for (;;) {
	bwr.read_size = sizeof(readbuf);
	bwr.read_consumed = 0;
	bwr.read_buffer = (unsigned) readbuf;

	res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);

	if (res < 0) {
		LOGE("binder_loop: ioctl failed (%s)\n", strerror(errno));
		break;
	}

	res = binder_parse(bs, 0, readbuf, bwr.read_consumed, func);
	if (res == 0) {
		LOGE("binder_loop: unexpected reply?!\n");
		break;
	}
	if (res < 0) {
		LOGE("binder_loop: io error %d %s\n", res, strerror(errno));
		break;
	}
}
```
我们看循环的第一次执行过程。首先还是在操作 binder_write_read 结构体，只不过这里操作的是 read 部分：它告诉驱动，我要读取数据，读取的数据大小是`sizeof(readbuf)`。执行的命令也是同样的`BINDER_WRITE_READ`，因此实际上还是走的这段代码:

```C++
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
    struct binder_proc *proc = filp->private_data;

    unsigned int size = _IOC_SIZE(cmd);
    void __user *ubuf = (void __user *)arg;

    switch (cmd) {
    case BINDER_WRITE_READ: {
        struct binder_write_read bwr;
        if (size != sizeof(struct binder_write_read)) {
            ret = -EINVAL;
            goto err;
        }
        if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {
            ret = -EFAULT;
            goto err;
        }

        if (bwr.write_size > 0) {
            ret = binder_thread_write(proc, thread, (void __user *)bwr.write_buffer, bwr.write_size, &bwr.write_consumed);
            trace_binder_write_done(ret);
            if (ret < 0) {
                bwr.read_consumed = 0;
                if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
                    ret = -EFAULT;
                goto err;
            }
        }
        
        if (bwr.read_size > 0) {
            ret = binder_thread_read(proc, thread, (void __user *)bwr.read_buffer, bwr.read_size, &bwr.read_consumed, filp->f_flags & O_NONBLOCK);
            trace_binder_read_done(ret);
            if (!list_empty(&proc->todo))
                wake_up_interruptible(&proc->wait);
            if (ret < 0) {
                if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
                    ret = -EFAULT;
                goto err;
            }
        }
        
        if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {
            ret = -EFAULT;
            goto err;
        }
        break;
    }
    ret = 0;
err:
    if (thread)
        thread->looper &= ~BINDER_LOOPER_STATE_NEED_RETURN;
    binder_unlock(__func__);
    wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);
    if (ret && ret != -ERESTARTSYS)
        printk(KERN_INFO "binder: %d:%d ioctl %x %lx returned %d\n", proc->pid, current->pid, cmd, arg, ret);
err_unlocked:
    trace_binder_ioctl_done(ret);
    return ret;
}
```
注意，在这里分析的时候不要被之前`BC_ENTER_LOOPER`的 bwr 结构体影响，那个写入的 bwr 是在`binder_write`方法中新创建的，这里的 bwr 是另外一个变量，就是在`binder_loop`中创建的。因此在`for`循环执行前，bwr 的状态是这样的：

```C++
struct binder_write_read bwr;
bwr.write_size = 0;
bwr.write_consumed = 0;
bwr.write_buffer = 0;
```

根据前面的代码，此时`bwr.read_size`的值是`sizeof(readbuf)`，因此实际执行的是:

```C++
// 调用代码：bwr.read_size = sizeof(readbuf);
if (bwr.read_size > 0) {
	ret = binder_thread_read(proc, thread, (void __user *)bwr.read_buffer, bwr.read_size, &bwr.read_consumed, filp->f_flags & O_NONBLOCK);
	trace_binder_read_done(ret);
	if (!list_empty(&proc->todo))
		wake_up_interruptible(&proc->wait);
	if (ret < 0) {
		if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
			ret = -EFAULT;
		goto err;
	}
}
```
刚进来就直接调用`binder_thread_read`函数:

```C++
/* 
 * 调用代码：ret = binder_thread_read(proc, thread, (void __user *)bwr.read_buffer, bwr.read_size, &bwr.read_consumed, filp->f_flags & O_NONBLOCK);
 * 参数解释：
 * bwr.read_size = sizeof(readbuf);
 * bwr.read_consumed = 0;
 * bwr.read_buffer = (unsigned) readbuf;
 * 
 * non_block 是 false 的，因为打开驱动的时候使用的代码是 open("/dev/binder", O_RDWR);
 * 并没有设置 O_NONBLOCK 标识，因此这个操作就是阻塞的
 */
static int binder_thread_read(struct binder_proc *proc,
                  struct binder_thread *thread,
                  void  __user *buffer, int size,
                  signed long *consumed, int non_block)
{
    void __user *ptr = buffer + *consumed;
    void __user *end = buffer + size;
    int ret = 0;
    int wait_for_proc_work;
    if (*consumed == 0) {
    	// 拷贝"BR_NOOP"命令到prt地址，也就是read部分的数据区
    	if (put_user(BR_NOOP, (uint32_t __user *)ptr))
            return -EFAULT;
        // 移动指针
        ptr += sizeof(uint32_t);
    }
retry:
	 // 成立，因此 wait_for_proc_work 为 true
    wait_for_proc_work = thread->transaction_stack == NULL && list_empty(&thread->todo);
    // 不成立，thread->return_error 在 binder_thread 的 return_error 字段在创建的时候就被初始化为 BR_OK，
    // 具体可见方法`binder_get_thread`方法中新建 binder_thread 结构体的部分
    // 另外 ptr = end
    if (thread->return_error != BR_OK && ptr < end) {
        if (thread->return_error2 != BR_OK) {
            if (put_user(thread->return_error2, (uint32_t __user *)ptr))
                return -EFAULT;
            ptr += sizeof(uint32_t);
            binder_stat_br(proc, thread, thread->return_error2);
            if (ptr == end)
                goto done;
            thread->return_error2 = BR_OK;
        }
        if (put_user(thread->return_error, (uint32_t __user *)ptr))
            return -EFAULT;
        ptr += sizeof(uint32_t);
        binder_stat_br(proc, thread, thread->return_error);
        thread->return_error = BR_OK;
        goto done;
    }
    // 设置状态为等待
    thread->looper |= BINDER_LOOPER_STATE_WAITING;
    // 表示有一个线程进入就绪状态
    if (wait_for_proc_work)
        proc->ready_threads++;
    binder_unlock(__func__);
    if (wait_for_proc_work) {
    	// 之前已经设置 BINDER_LOOPER_STATE_ENTERED 状态，因此判断不成立
        if (!(thread->looper & (BINDER_LOOPER_STATE_REGISTERED |
                    BINDER_LOOPER_STATE_ENTERED))) {
            binder_user_error("binder: %d:%d ERROR: Thread waiting "
                "for process work before calling BC_REGISTER_"
                "LOOPER or BC_ENTER_LOOPER (state %x)\n",
                proc->pid, thread->pid, thread->looper);
            wait_event_interruptible(binder_user_error_wait,
                         binder_stop_on_user_error < 2);
        }
        binder_set_nice(proc->default_priority);
        //打开binder设备的时候使用的是open("/dev/binder", O_RDWR)，而不是open("/dev/ttys", O_RDWR|O_NONBLOCK)，因此是阻塞的，所以non_block为false
        if (non_block) {
            if (!binder_has_proc_work(proc, thread))
                ret = -EAGAIN;
        } else
        	// 最终会执行到这里
            ret = wait_event_freezable_exclusive(proc->wait, binder_has_proc_work(proc, thread));
    } else {
        if (non_block) {
            if (!binder_has_thread_work(thread))
                ret = -EAGAIN;
        } else
            ret = wait_event_freezable(thread->wait, binder_has_thread_work(thread));
    }
    binder_lock(__func__);
    // 阻塞释放之后，减少一个就绪的线程
    if (wait_for_proc_work)
        proc->ready_threads--;
    // 表示线程不在等待状态
    thread->looper &= ~BINDER_LOOPER_STATE_WAITING;
    if (ret)
        return ret;
    while (1) {
        // ....
    }
    return 0;
}
```
代码中相关注释都已经标明具体在做什么，最终会执行到`wait_event_freezable_exclusive(proc->wait, binder_has_proc_work(proc, thread));`这一句。从网上找的资料看`wait_event_freezable_exclusive`基本等同于`wait_event_interruptible_exclusive`，老罗的分析中代码就是写的这个函数（它的源码版本比较老），调用这个方法之后进程就被阻塞，直到`binder_has_thread_work(thread)`为 true:

```C++
static int binder_has_thread_work(struct binder_thread *thread)
{
    return !list_empty(&thread->todo) || thread->return_error != BR_OK ||
        (thread->looper & BINDER_LOOPER_STATE_NEED_RETURN);
}
```
`thread->todo`列表不为空的时候，该函数返回true，至于这个列表中存放的什么，下两篇文章中会解释。

这里详细解释一下`wait_event_interruptible_exclusive`这个函数，Linux中还有一个类似的函数叫做`wait_event_interruptible`，exclusive表示排他进程，这个我们不去考虑，官方文档上对于这个方法的解释是:

```
wait_event_interruptible — sleep until a condition gets true

【Description】
The process is put to sleep (TASK_INTERRUPTIBLE) until the condition evaluates to true or a signal is received. The condition is checked each time the waitqueue wq is woken up.

wake_up has to be called after changing any variable that could change the result of the wait condition.

The function will return -ERESTARTSYS if it was interrupted by a signal and 0 if condition evaluated to true.
```
我们要看的其实是这句：__The function will return -ERESTARTSYS if it was interrupted by a signal and 0 if condition evaluated to true.__ 换句话说，如果是被中断的，返回的是`-ERESTARTSYS`，如果是条件满足后返回的，则返回0。

为什么要解释这个呢？因为在阻塞恢复以后，有这样一段代码:

```C++
if (ret)
	return ret;
```
ret 正是该函数的返回值，因此如果是条件满足后阻塞恢复，会继续往下执行。

OK，到这里，我们也把循环监听分析完了，以下是调用关系图:
<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/binder_read.png" alt="告诉binder驱动进入循环监听"/></div>

### 处理命令
当循环监听到消息，即有消息返回的时候，首先会通过一个while循环初步处理数据，之后返回无限循环中，调用方法`binder_parse`解析请求:

```C++
res = binder_parse(bs, 0, readbuf, bwr.read_consumed, func);
```
因为这里没有实际的场景，就暂不分析请求处理这块的代码，下面两篇文章中都会涉及到该函数的分析。

## 总结
综上，SM 启动的过程分析完毕。它现在在等待着`binder_has_thread_work`为真。