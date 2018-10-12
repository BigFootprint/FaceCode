---
title: Binder之Service查询
date: 2016-06-21 22:27:28
tags: [源码, Binder]
categories: Android
---

前一篇文章[Binder之Service注册](http://www.muzileecoding.com/framework/binder-service-register.html)描述了一个 Service 如何在 SM 中完成服务注册，接下来就是要探索如何向 SM 查询这个服务。有前一篇的基础，希望这一篇可以快一些~
>【注意】服务查询和服务注册过程其实非常接近，因此本文会大量引用前一篇文章的分析过程，读者可以仔细阅读上一篇文章之后再来阅读这篇文章。

本文将以 __前文__ 指代文章[Binder之Service注册](http://www.muzileecoding.com/framework/binder-service-register.html)。<!--more-->

## 研究入口
上一篇研究的对象是 MediaServer 和 MediaPlayerService，可以猜测一下在整个系统中必然有`getMediaPlayerService()`方法的调用，在 AndroidXRef 中搜索 "getMediaPlayerService" 字符串，很快就可以发现有很多类中调用了这个方法，我们选择 "IMediaDeathNotifier.cpp" 中的一段代码作为入口进行研究：

```C++
// establish binder interface to MediaPlayerService
/*static*/const sp<IMediaPlayerService>&
IMediaDeathNotifier::getMediaPlayerService()
{
    LOGV("getMediaPlayerService");
    Mutex::Autolock _l(sServiceLock);
    if (sMediaPlayerService.get() == 0) {
        sp<IServiceManager> sm = defaultServiceManager();
        sp<IBinder> binder;
        do {
            binder = sm->getService(String16("media.player"));
            if (binder != 0) {
                break;
             }
             usleep(500000); // 0.5 s
        } while(true);

        if (sDeathNotifier == NULL) {
        sDeathNotifier = new DeathNotifier();
    }
    binder->linkToDeath(sDeathNotifier);
    sMediaPlayerService = interface_cast<IMediaPlayerService>(binder);
    }
    LOGE_IF(sMediaPlayerService == 0, "no media player service!?");
    return sMediaPlayerService;
}
```
这个代码其实就是`getMediaPlayerService`方法的实现代码。可以看到核心代码其实只有三行：

```C++
sp<IServiceManager> sm = defaultServiceManager();
binder = sm->getService(String16("media.player"));
sMediaPlayerService = interface_cast<IMediaPlayerService>(binder);
```
第一行代码是获取 SM 实例的，这一点在前文中已经有非常详细的解析过程，这里就不再重复解析了。重点是后面两句。

__【涉及文件】__

| 文件                      | 位置                                       |
| ----------------------- | ---------------------------------------- |
| IMediaDeathNotifier.cpp | /frameworks/base/media/libmedia/IMediaDeathNotifier.cpp |
包括前文所提到的文件。

## 发送服务获取命令
服务获取调用的代码是:

```C++
binder = sm->getService(String16("media.player"));
```

`getService()`方法的实现如下:

```C++
virtual sp<IBinder> getService(const String16& name) const {
	unsigned n;
	for (n = 0; n < 5; n++){
		// 调用checkService方法获取
		sp<IBinder> svc = checkService(name);
		if (svc != NULL) return svc;
		LOGI("Waiting for service %s...\n", String8(name).string());
		sleep(1);
	}
	return NULL;
}

virtual sp<IBinder> checkService( const String16& name) const {
	Parcel data, reply;
	data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());
	data.writeString16(name);
	remote()->transact(CHECK_SERVICE_TRANSACTION, data, &reply);
	return reply.readStrongBinder();
}
```
这些方法名字虽然第一次见，可是内容应该非常熟悉了 —— 和注册服务非常非常接近，只不过这一次命令有所变化。关于`transact()`方法的追踪，上一篇文章中也有非常详细的分析，详见前文 5.2 节。根据前文，我们最终会来到：

```C++
// 调用代码: writeTransactionData(BC_TRANSACTION, flags, handle, code, data, NULL);
// 参数解释: handle 为 0，code 为 CHECK_SERVICE_TRANSACTION， binderFlags 是 TF_ACCEPT_FDS;
status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,
    int32_t handle, uint32_t code, const Parcel& data, status_t* statusBuffer)
{
	// 该结构体在字典中可查
    binder_transaction_data tr;

    tr.target.handle = handle;
    tr.code = code;
    tr.flags = binderFlags;
    tr.cookie = 0;
    tr.sender_pid = 0;
    tr.sender_euid = 0;
    
    const status_t err = data.errorCheck();
    if (err == NO_ERROR) {
    	// 将之前写在 data 中的数据再换一种形式写到 binder_transaction_data 中
        tr.data_size = data.ipcDataSize(); 
        tr.data.ptr.buffer = data.ipcData(); 
        tr.offsets_size = data.ipcObjectsCount()*sizeof(size_t);
        tr.data.ptr.offsets = data.ipcObjects(); 
    } else if (statusBuffer) {
        tr.flags |= TF_STATUS_CODE;
        *statusBuffer = err;
        tr.data_size = sizeof(status_t);
        tr.data.ptr.buffer = statusBuffer;
        tr.offsets_size = 0;
        tr.data.ptr.offsets = NULL;
    } else {
        return (mLastError = err);
    }
    
    mOut.writeInt32(cmd);//BC_TRANSACTION
    mOut.write(&tr, sizeof(tr));
    
    return NO_ERROR;
}
```
这里要注意的是，除了命令不一样，数据上也有差别了。前文中是进行服务注册，所以数据中是携带了一个序列化的服务节点的，但这里很简单，只有想要获取的服务名字而已。

## SM 处理命令
中间的驱动处理和前文基本一致，这里略去分析，主要是看 SM 如何处理这个命令，处理函数是`svcmgr_handler`: 

```C++
uint16_t svcmgr_id[] = { 
    'a','n','d','r','o','i','d','.','o','s','.',
    'I','S','e','r','v','i','c','e','M','a','n','a','g','e','r' 
};

int svcmgr_handler(struct binder_state *bs,
                   struct binder_txn *txn,
                   struct binder_io *msg,
                   struct binder_io *reply)
{
    struct svcinfo *si;
    uint16_t *s;
    unsigned len;
    void *ptr;
    uint32_t strict_policy;

	// svcmgr_handle 被赋值为 svcmgr，即BINDER_SERVICE_MANAGER，即0
    if (txn->target != svcmgr_handle)
        return -1;

    // Equivalent to Parcel::enforceInterface(), reading the RPC
    // header with the strict mode policy mask and the interface name.
    // Note that we ignore the strict_policy and don't propagate it
    // further (since we do no outbound RPCs anyway).
    strict_policy = bio_get_uint32(msg);
    s = bio_get_string16(msg, &len);
    // svcmgr_id是一个字符数组，见最上面，这里判断是通过的，因为s读取出来就是"android.os.IServiceManager"
    if ((len != (sizeof(svcmgr_id) / 2)) ||
        memcmp(svcmgr_id, s, sizeof(svcmgr_id))) {
        fprintf(stderr,"invalid id %s\n", str8(s));
        return -1;
    }

    switch(txn->code) {
    case SVC_MGR_GET_SERVICE:
    case SVC_MGR_CHECK_SERVICE:
        s = bio_get_string16(msg, &len); // "media.player"
        ptr = do_find_service(bs, s, len);
        if (!ptr)
            break;
        bio_put_ref(reply, ptr);
        return 0;
		......
    }

    bio_put_uint32(reply, 0);
    return 0;
}
```
在这里其实可以看到，获取服务和查询服务的处理流程是一样的。
>至于为什么会落到`SVC_MGR_CHECK_SERVICE`这个命令，前文也有解释。

在获取服务名称之后，调用`do_find_service`方法来查询服务: 

```C++
void *do_find_service(struct binder_state *bs, uint16_t *s, unsigned len)
{
    struct svcinfo *si;
    si = find_svc(s, len);

//    LOGI("check_service('%s') ptr = %p\n", str8(s), si ? si->ptr : 0);
    if (si && si->ptr) {
        return si->ptr;
    } else {
        return 0;
    }
}

struct svcinfo *find_svc(uint16_t *s16, unsigned len)
{
    struct svcinfo *si;

    for (si = svclist; si; si = si->next) {
        if ((len == si->len) &&
            !memcmp(s16, si->name, len * sizeof(uint16_t))) {
            return si;
        }
    }
    return 0;
}
```
函数`find_svc`前文已经解析过了，这里其实就是找到在 svclist 中是否有对应的 svcinfo 节点，名称与需要查找的服务名称一致，如果有，则返回 svcinfo 的 ptr 字段，ptr 字段的来源前文也有解释: 服务在 SM 中分配得到的 handle 值。也就是说，返回这个 handle 值就好了。

找到 ptr 之后，我们回到`do_find_service`，在这里会调用`bio_put_ref`方法:

```C++
void bio_put_ref(struct binder_io *bio, void *ptr)
{
	// 该结构体在字典中可查
    struct binder_object *obj;

    if (ptr)
        obj = bio_alloc_obj(bio);
    else
        obj = bio_alloc(bio, sizeof(*obj));

    if (!obj)
        return;

    obj->flags = 0x7f | FLAT_BINDER_FLAG_ACCEPTS_FDS;
    obj->type = BINDER_TYPE_HANDLE;
    obj->pointer = ptr;
    obj->cookie = 0;
}
```
这里有必要好好分析一下 binder_io 这个结构体相关的方法了，它们集中在 "binder.c" 文件中。首先来看`bio_alloc_obj`方法: 

```C++
static struct binder_object *bio_alloc_obj(struct binder_io *bio)
{
    struct binder_object *obj;

	// 下面解释 bio_alloc 方法
    obj = bio_alloc(bio, sizeof(*obj));
    
    if (obj && bio->offs_avail) {
    	// 表示偏移数据可记录数目又少了1，因为下面即将记录新的偏移信息
        bio->offs_avail--;
        // 记录新的 binder_object 在结构体中的偏移量
        *bio->offs++ = ((char*) obj) - ((char*) bio->data0);
        // 返回 binder_object 结构体的起始位置
        return obj;
    }

    bio->flags |= BIO_F_OVERFLOW;
    return 0;
}
```
这个方法声明了一个 `binder_object` 的结构体，实例化方法调用的是`bio_alloc`: 

```C++
static void *bio_alloc(struct binder_io *bio, uint32_t size)
{
    size = (size + 3) & (~3);
    if (size > bio->data_avail) {
        bio->flags |= BIO_F_OVERFLOW;
        return 0;
    } else {
        void *ptr = bio->data;
        bio->data += size;
        bio->data_avail -= size;
        return ptr;
    }
}
```
该方法非常简单，首先检测传入的参数 binder_io 是否还有足够的容量分配 size 的空间，如果足够，就操作 data 和 data_avail 变量，仅这样来表示空间分配。方法会返回分配空间的起始地址。综上，这组方法以及 binder_io 结构体其实是对内存分配的一个封装。

回到`bio_alloc_obj`方法后的代码都有注释，不难理解。再回到`bio_put_ref`方法后，会在该结构体中填充以下数据: 

```C++
obj->flags = 0x7f | FLAT_BINDER_FLAG_ACCEPTS_FDS;
obj->type = BINDER_TYPE_HANDLE;
obj->pointer = ptr;
obj->cookie = 0;
```
这部分数据很重要，后面还会用到。回到`svcmgr_handler`函数，还会执行下面语句才会返回:

```C++
bio_put_uint32(reply, 0);

void bio_put_uint32(struct binder_io *bio, uint32_t n)
{
    uint32_t *ptr = bio_alloc(bio, sizeof(n));
    if (ptr)
        *ptr = n;
}
```
还是在 binder_io 结构体中分配空间写入数据。执行完毕之后，回到函数`binder_parse`中:

```C++
int binder_parse(struct binder_state *bs, struct binder_io *bio,
                 uint32_t *ptr, uint32_t size, binder_handler func)
{
    int r = 1;
    uint32_t *end = ptr + (size / 4);

    while (ptr < end) {
        uint32_t cmd = *ptr++;
        switch(cmd) {
            ......
        case BR_TRANSACTION: {
        	//该结构体字典中可查
            struct binder_txn *txn = (void *) ptr;
            if ((end - ptr) * sizeof(uint32_t) < sizeof(struct binder_txn)) {
                LOGE("parse: txn too small!\n");
                return -1;
            }
            binder_dump_txn(txn);
            if (func) {
                unsigned rdata[256/4];
                // 该结构体字典中可查
                struct binder_io msg;
                struct binder_io reply;
                int res;

                bio_init(&reply, rdata, sizeof(rdata), 4);
                bio_init_from_txn(&msg, txn);
                // 刚刚在这里，不清楚的可以看前文
                res = func(bs, txn, &msg, &reply);
                binder_send_reply(bs, &reply, txn->data, res);
            }
            ptr += sizeof(*txn) / sizeof(uint32_t);
            break;
        }
        .....
    }

    return r;
}
```

接下去就是要执行`binder_send_reply`方法发送 reply 数据:

```C++
// 参数解释：status 为0
void binder_send_reply(struct binder_state *bs,
                       struct binder_io *reply,
                       void *buffer_to_free,
                       int status)
{
    struct {
        uint32_t cmd_free;
        void *buffer;
        uint32_t cmd_reply;
        struct binder_txn txn;
    } __attribute__((packed)) data;

    data.cmd_free = BC_FREE_BUFFER; // 注意命令
    data.buffer = buffer_to_free;
    data.cmd_reply = BC_REPLY; // 注意命令
    data.txn.target = 0;
    data.txn.cookie = 0;
    data.txn.code = 0;
    if (status) {
        data.txn.flags = TF_STATUS_CODE;
        data.txn.data_size = sizeof(int);
        data.txn.offs_size = 0;
        data.txn.data = &status;
        data.txn.offs = 0;
    } else { // 走这里，和前面的分析一对比就非常清晰
        data.txn.flags = 0;
        data.txn.data_size = reply->data - reply->data0;
        data.txn.offs_size = ((char*) reply->offs) - ((char*) reply->offs0);
        data.txn.data = reply->data0;
        data.txn.offs = reply->offs0;
    }
    binder_write(bs, &data, sizeof(data));
}
```
reply 中的数据部分包含两组数据，一个是一个 binder_io 结构体，一个则是整数 0。接下去就不分析了，和前文一样，我们直接进入驱动。

## 驱动节点生成
根据前文的分析，答复的处理肯定经过方法`binder_transaction`，这也是本文分析的重点，我们看看和服务注册具体有什么区别。

```C++
static void binder_transaction(struct binder_proc *proc,
                   struct binder_thread *thread,
                   struct binder_transaction_data *tr, int reply)
{
	......
    // 核心不同！确实有数据，是通过 binder_object 结构体写入的
    off_end = (void *)offp + tr->offsets_size;
    for (; offp < off_end; offp++) {
        struct flat_binder_object *fp;
        fp = (struct flat_binder_object *)(t->buffer->data + *offp);
        switch (fp->type) {
        ......
        case BINDER_TYPE_HANDLE:
        case BINDER_TYPE_WEAK_HANDLE: {
            // 获取服务注册的时候在 SM 中生成的 binder_ref
            struct binder_ref *ref = binder_get_ref(proc, fp->handle);
            // 权限检测
            if (security_binder_transfer_binder(proc->tsk, target_proc->tsk)) {
                return_error = BR_FAILED_REPLY;
                goto err_binder_get_ref_failed;
            }
            if (ref->node->proc == target_proc) {// 如果请求的服务节点的进程就是请求发起的进程
                if (fp->type == BINDER_TYPE_HANDLE)
                    fp->type = BINDER_TYPE_BINDER; // 直接更改为 binder 实体节点，不需要引用
                else
                    fp->type = BINDER_TYPE_WEAK_BINDER;
                fp->binder = ref->node->ptr;
                fp->cookie = ref->node->cookie;
                binder_inc_node(ref->node, fp->type == BINDER_TYPE_BINDER, 0, NULL);
                trace_binder_transaction_ref_to_node(t, ref);
            } else {
                struct binder_ref *new_ref;
                // ⭐️在目标进程，也就是请求进程中查看该节点的引用，没有则新建
                new_ref = binder_get_ref_for_node(target_proc, ref->node);
                // 拿到该服务节点在本进程中的 handle 值
                fp->handle = new_ref->desc;
                binder_inc_ref(new_ref, fp->type == BINDER_TYPE_HANDLE, NULL);
                trace_binder_transaction_ref_to_ref(t, ref,
                                    new_ref);
            }
        } break;
        ......
        }
    }
    if (reply) {
        BUG_ON(t->buffer->async_transaction != 0);
        binder_pop_transaction(target_thread, in_reply_to);
    } else if (!(t->flags & TF_ONE_WAY)) {
        ......
    } else {
        ......
    }
    // 挂载节点
    t->work.type = BINDER_WORK_TRANSACTION;
    list_add_tail(&t->work.entry, target_list);
    tcomplete->type = BINDER_WORK_TRANSACTION_COMPLETE;
    list_add_tail(&tcomplete->entry, &thread->todo);
    if (target_wait)
        wake_up_interruptible(target_wait);
    return;
    ......
}
```
这个方法前半部分的处理和服务注册都类似，核心不同在于驱动对数据的处理上：

__服务注册，binder_transaction中的 binder 节点是一个 binder_node 实体节点，在驱动中它会将它换成 BINDER_TYPE_HANDLE 类型，并在请求注册的进程上生成新的实体节点；但是服务查询中，却只会判断该服务是否属于当前进程，如果服务属于当前进程，则会换成 BINDER_TYPE_BINDER 类型；否则会以 BINDER_TYPE_HANDLE 类型继续传递处理。尤其要注意⭐️的地方！__

经过数据处理，binder_transaction 的数据内容已经变化。到这里，SM 的处理全部完成。

## 查询方的等待
查询方此时必然在`binder_thread_read`方法中等待: 

```C++
static int binder_thread_read(struct binder_proc *proc,
                  struct binder_thread *thread,
                  void  __user *buffer, int size,
                  signed long *consumed, int non_block)
{
    ......
    while (1) {
        uint32_t cmd;
        struct binder_transaction_data tr;
        struct binder_work *w;
        struct binder_transaction *t = NULL;
        if (!list_empty(&thread->todo))
            w = list_first_entry(&thread->todo, struct binder_work, entry);
        else if (!list_empty(&proc->todo) && wait_for_proc_work)
            w = list_first_entry(&proc->todo, struct binder_work, entry);
        else {
            if (ptr - buffer == 4 && !(thread->looper & BINDER_LOOPER_STATE_NEED_RETURN)) /* no data added */
                goto retry;
            break;
        }
        if (end - ptr < sizeof(tr) + 4)
            break;
        switch (w->type) {
        case BINDER_WORK_TRANSACTION: {
            t = container_of(w, struct binder_transaction, work);
        } break;
        ......
        }
        if (!t)
            continue;
        
        // reply 的时候，该字段赋值为 NULL
        if (t->buffer->target_node) {
            ......
        } else { // 走这里
            tr.target.ptr = NULL;
            tr.cookie = NULL;
            cmd = BR_REPLY;
        }
        tr.code = t->code;
        tr.flags = t->flags;
        tr.sender_euid = t->sender_euid;
        if (t->from) {
            struct task_struct *sender = t->from->proc->tsk;
            tr.sender_pid = task_tgid_nr_ns(sender,
                            current->nsproxy->pid_ns);
        } else {
            tr.sender_pid = 0;
        }
        tr.data_size = t->buffer->data_size;
        tr.offsets_size = t->buffer->offsets_size;
        tr.data.ptr.buffer = (void *)t->buffer->data +
                    proc->user_buffer_offset;
        tr.data.ptr.offsets = tr.data.ptr.buffer +
                    ALIGN(t->buffer->data_size,
                        sizeof(void *));
        if (put_user(cmd, (uint32_t __user *)ptr))
            return -EFAULT;
        ptr += sizeof(uint32_t);
        if (copy_to_user(ptr, &tr, sizeof(tr)))
            return -EFAULT;
        ptr += sizeof(tr);
        trace_binder_transaction_received(t);
        binder_stat_br(proc, thread, cmd);
        list_del(&t->work.entry);
        t->buffer->allow_user_free = 1;
        if (cmd == BR_TRANSACTION && !(t->flags & TF_ONE_WAY)) {
            t->to_parent = thread->transaction_stack;
            t->to_thread = thread;
            thread->transaction_stack = t;
        } else {
            t->buffer->transaction = NULL;
            kfree(t);
            binder_stats_deleted(BINDER_STAT_TRANSACTION);
        }
        break;
    }
    return 0;
}
```
这里有必要详细分析一下数据内容了。这里面有一段很重要的数据拷贝: 

```C++
tr.data_size = t->buffer->data_size;
tr.offsets_size = t->buffer->offsets_size;
tr.data.ptr.buffer = (void *)t->buffer->data + proc->user_buffer_offset;
tr.data.ptr.offsets = tr.data.ptr.buffer + ALIGN(t->buffer->data_size, sizeof(void *));
if (put_user(cmd, (uint32_t __user *)ptr))
	return -EFAULT;
ptr += sizeof(uint32_t);
if (copy_to_user(ptr, &tr, sizeof(tr)))
	return -EFAULT;
```
首先将数据内容和偏移数据大小记录下来，接着将数据地址映射到用户空间，以便于用户空间可以直接访问这块内存。之后拷贝`BR_TRANSACTION`命令，拷贝 binder_transaction_data 结构体后返回。中间过程比较冗长，在前文中也做了分析，我们直接到处理返回结果的地方。

```C++
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
    int32_t cmd;
    int32_t err;

    while (1) {
        if ((err=talkWithDriver()) < NO_ERROR) break;
        err = mIn.errorCheck();
        if (err < NO_ERROR) break;
        if (mIn.dataAvail() == 0) continue;
        
        cmd = mIn.readInt32();
        switch (cmd) {
            ......
        case BR_REPLY:
            {
                binder_transaction_data tr;
                err = mIn.read(&tr, sizeof(tr));
                LOG_ASSERT(err == NO_ERROR, "Not enough command data for brREPLY");
                if (err != NO_ERROR) goto finish;

                if (reply) {
                    if ((tr.flags & TF_STATUS_CODE) == 0) {
                        reply->ipcSetDataReference(
                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                            tr.data_size,
                            reinterpret_cast<const size_t*>(tr.data.ptr.offsets),
                            tr.offsets_size/sizeof(size_t),
                            freeBuffer, this);
                    } else {
                        err = *static_cast<const status_t*>(tr.data.ptr.buffer);
                        freeBuffer(NULL,
                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                            tr.data_size,
                            reinterpret_cast<const size_t*>(tr.data.ptr.offsets),
                            tr.offsets_size/sizeof(size_t), this);
                    }
                } else {
                    ......
                    continue;
                }
            }
            goto finish;
            ......
        }
    }
    ......
    return err;
}
```
这里首先从 mIn 中把 binder_transaction_data 结构体读出来，然后调用`Parcel.ipcSetDataReference()`方法把数据写入到 reply 中去：

```C++
/* 调用代码:reply->ipcSetDataReference(
 *                          reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
 *                          tr.data_size,
 *                          reinterpret_cast<const size_t*>(tr.data.ptr.offsets),
 *                          tr.offsets_size/sizeof(size_t),
 *                          freeBuffer, this);
 */ 
void Parcel::ipcSetDataReference(const uint8_t* data, size_t dataSize,
    const size_t* objects, size_t objectsCount, release_func relFunc, void* relCookie)
{
    freeDataNoInit();
    mError = NO_ERROR;
    mData = const_cast<uint8_t*>(data);
    mDataSize = mDataCapacity = dataSize;
    mDataPos = 0;
    LOGV("setDataReference Setting data pos of %p to %d\n", this, mDataPos);
    mObjects = const_cast<size_t*>(objects);
    mObjectsSize = mObjectsCapacity = objectsCount;
    mNextObjectHint = 0;
    mOwner = relFunc;
    mOwnerCookie = relCookie;
    scanForFds();
}
```
这几个参数前面基本都见过，不多做解释，但是 objectsCount 的计算式是`tr.offsets_size/sizeof(size_t)`，即偏移数据量尺寸除以偏移记录大小，因此就是偏移记录的条数，也就是记录的对象数，因为是查询服务，从前面的分析来看，这里只返回了一个对象。然后根据 objects 参数，就可以从 data 中获取到所有的 binder 节点数据。

而 reply 参数是怎么来的呢？来自这里:

```C++
remote()->transact(CHECK_SERVICE_TRANSACTION, data, &reply);
```

接下去一句便是: 

```C++
return reply.readStrongBinder();
```
如上分析，此时 reply 中已经有有关 binder 的数据了，那么是怎么读取的呢？

```C++
sp<IBinder> Parcel::readStrongBinder() const
{
    sp<IBinder> val;
    unflatten_binder(ProcessState::self(), *this, &val);
    return val;
}

status_t unflatten_binder(const sp<ProcessState>& proc,
    const Parcel& in, sp<IBinder>* out)
{
    const flat_binder_object* flat = in.readObject(false);
    
    if (flat) {
        switch (flat->type) {
            case BINDER_TYPE_BINDER:
                *out = static_cast<IBinder*>(flat->cookie);
                return finish_unflatten_binder(NULL, *flat, in);
            case BINDER_TYPE_HANDLE:
                *out = proc->getStrongProxyForHandle(flat->handle);
                return finish_unflatten_binder(
                    static_cast<BpBinder*>(out->get()), *flat, in);
        }        
    }
    return BAD_TYPE;
}
```

实际的读取是在`readObject`中进行的:

```C++
const flat_binder_object* Parcel::readObject(bool nullMetaData) const
{
    const size_t DPOS = mDataPos;
    // 首先判断剩余数据量是否足以读取出一个 flat_binder_object
    if ((DPOS+sizeof(flat_binder_object)) <= mDataSize) {
        // 强转读取
        const flat_binder_object* obj
                = reinterpret_cast<const flat_binder_object*>(mData+DPOS);
        mDataPos = DPOS + sizeof(flat_binder_object);
        ......
        // Ensure that this object is valid...
        size_t* const OBJS = mObjects;
        const size_t N = mObjectsSize;
        size_t opos = mNextObjectHint;
        
        if (N > 0) {
            LOGV("Parcel %p looking for obj at %d, hint=%d\n",
                 this, DPOS, opos);
            
            // Start at the current hint position, looking for an object at
            // the current data position.
            if (opos < N) {
                while (opos < (N-1) && OBJS[opos] < DPOS) {
                    opos++;
                }
            } else {
                opos = N-1;
            }
            if (OBJS[opos] == DPOS) {
                mNextObjectHint = opos+1;
                return obj;
            }
        
            // Look backwards for it...
            while (opos > 0 && OBJS[opos] > DPOS) {
                opos--;
            }
            if (OBJS[opos] == DPOS) {
                // Found it!
                LOGV("Parcel found obj %d at index %d with backward search",
                     this, DPOS, opos);
                mNextObjectHint = opos+1;
                LOGV("readObject Setting data pos of %p to %d\n", this, mDataPos);
                return obj;
            }
        }
    }
    return NULL;
}
```
这里是一个简单的读取，但是会做一些校验。在当前情况下，当然可以找到我们想要的 flat_binder_object 结构体对象，接着回到`unflatten_binder`，执行如下代码: 

```C++
if (flat) {
	switch (flat->type) {
		case BINDER_TYPE_BINDER:
				*out = static_cast<IBinder*>(flat->cookie);
 				return finish_unflatten_binder(NULL, *flat, in);
		case BINDER_TYPE_HANDLE:
				*out = proc->getStrongProxyForHandle(flat->handle);
				return finish_unflatten_binder(
                    	static_cast<BpBinder*>(out->get()), *flat, in);
        }        
    }
```
很明显，我们返回的 Binder 类型是 BINDER_TYPE_HANDLE 的。out 最后被赋值为 `proc->getStrongProxyForHandle(flat->handle);`，`getStrongProxyForHandle`这个函数我们也是见过的，这里再分析一次，因为情景有些不同。

```C++
sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
{
    sp<IBinder> result;

    AutoMutex _l(mLock);

	// handle 为0
    handle_entry* e = lookupHandleLocked(handle);

    if (e != NULL) {
        // We need to create a new BpBinder if there isn't currently one, OR we
        // are unable to acquire a weak reference on this current one.  See comment
        // in getWeakProxyForHandle() for more info about this.
        IBinder* b = e->binder;
        if (b == NULL || !e->refs->attemptIncWeak(this)) {
            b = new BpBinder(handle); 
            e->binder = b;
            if (b) e->refs = b->getWeakRefs();
            result = b;
        } else {
            // This little bit of nastyness is to allow us to add a primary
            // reference to the remote proxy when this team doesn't have one
            // but another team is sending the handle to us.
            result.force_set(b);
            e->refs->decWeak(this);
        }
    }

    return result;
}

ProcessState::handle_entry* ProcessState::lookupHandleLocked(int32_t handle)
{
    const size_t N = mHandleToObject.size();
    if (N <= (size_t)handle) {
        handle_entry e;
        e.binder = NULL;
        e.refs = NULL;
        status_t err = mHandleToObject.insertAt(e, N, handle+1-N);
        if (err < NO_ERROR) return NULL;
    }
    return &mHandleToObject.editItemAt(handle);
}
```
因为是初次获取这个服务，因此实际上`lookupHandleLocked`返回的还是一个新建的 handle_entry 结构体，因此实际上`getStrongProxyForHandle`返回的对象就是`new BpBinder(handle)`，这整个过程返回的则是`new BpBinder(flat->handle)`对象。

## 最后的交互
现在Client已经拿到这个`new BpBinder(flat->handle)`了，那么怎么和远程服务通信呢？这个其实在注册服务中也分析过了，只不过那时候`flat->handle`为 0 而已，而在`binder_transaction`方法中，是否为 0 是有区别的:

```C++
if (tr->target.handle) {
	struct binder_ref *ref;
	ref = binder_get_ref(proc, tr->target.handle);
	target_node = ref->node;
} else {
	target_node = binder_context_mgr_node;
	if (target_node == NULL) {
		return_error = BR_DEAD_REPLY;
		goto err_no_context_mgr_node;
	}
}
```
如果 handle 为 0，默认目标服务节点就是`binder_context_mgr_node`，但如果不为 0，则会在进程(`binder_proc`对象)的红黑树中寻找，这个寻找一定是OK的，因为在查询服务的时候就把`binder_ref`节点插入到红黑树中去了。自然而然也就能定位到目标进程，从而发送请求。

## interface_cast
在发送服务请求之前，还需要做一次转换，因为我们还不知道远程服务的API是什么样子的，这就是`IMediaDeathNotifier::getMediaPlayerService()`方法中最后一句的作用:

```C++
sMediaPlayerService = interface_cast<IMediaPlayerService>(binder);
```
`interface_cast`在前文已经分析过一次了，很容易推断出最终`interface_cast<IMediaPlayerService>(binder);`返回的值是：__`new BpMediaPlayerService(new BpBinder(flat->handle))`__。

这在下图中也能看到该对象:
<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/Binder-Lib层架构.png" alt="Binder Lib层架构"/></div>

这里就不多做分析了。

## 总结
至此，搜寻服务也分析完毕。如果读者对这几篇文章都已经比较熟悉，墙裂建议阅读推荐文章①和文章③两系列博客！整个分析下来，Binder 机制所做的重要事情如下：

1. 启动 SM，并且固定 SM 的 handle 值为0，这样任何其余的服务都可以随时获取到 SM 服务；
2. 服务启动后都会去 SM 注册，这个注册的过程是：在驱动层本进程的映射数据结构`binder_proc`中生成对应的`binder_node`节点，这个节点在穿越驱动边界的时候，会在目标进程中生成`binder_ref`结构体，注册的时候，这个目标进程就是 SM，也就是说 SM 会持有所有服务的引用；
3. 服务查询的时候，也是问询 SM ，SM 就通过名字查询对应的服务引用，在返回的时候同样需要穿越驱动边界，驱动又会查询目标进程是否有该服务节点的引用，没有的话又会去目标进程生成一个，并且生成对应的 handle 值，这个值会最总返回到 Libraries 层；
4. 在 Librries 层会通过 `new BpXXXService(new BpBinder(handle))`的形式生成服务对象这其实是一个代理，最外层的`new BpXXXService()`最主要的目的是找到`XXXService`的接口，而内部`new BpBinder(handle)`则是为了将请求转发给真正的服务进程，因为这个时候持有 handle，就可以在驱动层查询到对应的`binder_node`节点，从而可以向对方 proc/thread 挂载任务并唤醒对方；

以上。