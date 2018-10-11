---
title: Binder之Service注册
date: 2016-06-21 22:27:28
tags: [源码, Binder]
categories: Android
---

前文[Binder之Service Manager](http://www.muzileecoding.com/framework/binder-servicemanager.html)已经讲述了 SM 如何启动并循环监听消息的，接下去按照逻辑应该是先说如何注册一个服务再说如何获取一个服务，但是从代码层面来说，先说获取服务似乎更好，好纠结。还是按照逻辑来讲吧~

## 研究对象
既然是讲述服务注册，最好的当然是从现有的系统中找出一枚服务，研究它的注册过程。[总纲](http://www.muzileecoding.com/framework/binder.html)中推荐的5篇文章正好有一篇是从这个角度切入分析Binder的：文章⑤[Android深入浅出之Binder机制](http://www.cnblogs.com/innost/archive/2011/01/09/1931456.html)。它是从 MediaServer 来切入分析的。

因为我们主要是看注册的过程，选择什么服务是次要的，因此这里也以该服务为分析对象。__以下 MediaServer 简称 MS__。MS 是为系统提供多媒体功能的服务，比如音频、视频等。
> 和前一篇文章一样，这里仍然沿用 SM 指代 Service Manager。

<!--more-->__【涉及文件】__

| 文件                      | 位置                                       |
| ----------------------- | ---------------------------------------- |
| main_mediaserver.cpp    | /frameworks/base/media/mediaserver/main_mediaserver.cpp |
| IServiceManager.h       | /frameworks/base/include/binder/IServiceManager.h |
| IServiceManager.cpp     | /frameworks/base/libs/binder/IServiceManager.cpp |
| IMediaPlayerService.cpp | /frameworks/base/media/libmedia/IMediaPlayerService.cpp |
| IPCThreadState.cpp      | /frameworks/base/libs/binder/IPCThreadState.cpp |
| ProcessState.cpp        | /frameworks/base/libs/binder/ProcessState.cpp |
| Binder.cpp              | /frameworks/base/libs/binder/Binder.cpp  |
| BpBinder.cpp            | /frameworks/base/libs/binder/BpBinder.cpp |
| Parcel.cpp              | /frameworks/base/libs/binder/Parcel.cpp  |
| IInterface.h            | /frameworks/base/include/binder/IInterface.h |
其余一些零碎文件、头文件，这里不一一列举了。

## 初见 MS
MS 的`main`函数位于 "main_mediaserver.cpp" 中，这个文件的代码内容非常简单:

```C++
using namespace android;

int main(int argc, char** argv)
{
    sp<ProcessState> proc(ProcessState::self());
    sp<IServiceManager> sm = defaultServiceManager();
    LOGI("ServiceManager: %p", sm.get());
    AudioFlinger::instantiate();
    MediaPlayerService::instantiate();
    CameraService::instantiate();
    AudioPolicyService::instantiate();
    ProcessState::self()->startThreadPool();
    IPCThreadState::self()->joinThreadPool();
}
```
就只有一个`main`函数。该方法会初始化很多的多媒体服务，我们选择一个 —— __MediaPlayerService__ 分析就好了。因此以下是我们本次的代码分析路径:
<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/main_mediaserver_main.png" width="320" alt="MS main函数调用"/></div>

## 打开驱动 & 内存映射
`main`函数的第一行代码核心在于`ProcessState::self()`这一句:
>`sp<XXX>` 可以看成 `XXX *`，即指针。

```C++
sp<ProcessState> ProcessState::self()
{
    if (gProcess != NULL) return gProcess;
    
    AutoMutex _l(gProcessMutex);
    if (gProcess == NULL) gProcess = new ProcessState;
    return gProcess;
}
```
这是一个单例方法，确保只创建一个 ProcessState ，我们看看 ProcessState 的构造函数:

```C++
ProcessState::ProcessState()
    : mDriverFD(open_driver())
    , mVMStart(MAP_FAILED)
    , mManagesContexts(false)
    , mBinderContextCheckFunc(NULL)
    , mBinderContextUserData(NULL)
    , mThreadPoolStarted(false)
    , mThreadPoolSeq(1)
{
    if (mDriverFD >= 0) {
        // XXX Ideally, there should be a specific define for whether we
        // have mmap (or whether we could possibly have the kernel module
        // availabla).
#if !defined(HAVE_WIN32_IPC)
        // mmap the binder, providing a chunk of virtual address space to receive transactions.
        mVMStart = mmap(0, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
        if (mVMStart == MAP_FAILED) {
            // *sigh*
            LOGE("Using /dev/binder failed: unable to mmap transaction memory.\n");
            close(mDriverFD);
            mDriverFD = -1;
        }
#else
        mDriverFD = -1;
#endif
    }

    LOG_ALWAYS_FATAL_IF(mDriverFD < 0, "Binder driver could not be opened.  Terminating.");
}
```
构造函数并不复杂，但是关键点很隐蔽： __`mDriverFD(open_driver())`__ ，这里将调用函数`open_driver()`，并将返回值赋值给`mDriverFD`。自然而然我们又要去看看`open_driver()`函数:

```C++
static int open_driver()
{
    int fd = open("/dev/binder", O_RDWR);
    if (fd >= 0) {
        fcntl(fd, F_SETFD, FD_CLOEXEC);
        int vers;
        status_t result = ioctl(fd, BINDER_VERSION, &vers);
        if (result == -1) {
            LOGE("Binder ioctl to obtain version failed: %s", strerror(errno));
            close(fd);
            fd = -1;
        }
        if (result != 0 || vers != BINDER_CURRENT_PROTOCOL_VERSION) {
            LOGE("Binder driver protocol does not match user space protocol!");
            close(fd);
            fd = -1;
        }
        size_t maxThreads = 15;
        result = ioctl(fd, BINDER_SET_MAX_THREADS, &maxThreads);
        if (result == -1) {
            LOGE("Binder ioctl to set max threads failed: %s", strerror(errno));
        }
    } else {
        LOGW("Opening '/dev/binder' failed: %s\n", strerror(errno));
    }
    return fd;
}
```
这里看到了熟悉的代码：它打开了 Binder 驱动。如果打开成功，它会向驱动发送两个命令，一个是`BINDER_VERSION`，这个无关紧要，略过；一个是`BINDER_SET_MAX_THREADS`，这个最终还是会走到`binder_ioctrl`中去，多余代码就不贴了，直接看这条命令的处理case:

```C++
// ubuf 指代传递进入驱动的数据
case BINDER_SET_MAX_THREADS:
		if (copy_from_user(&proc->max_threads, ubuf, sizeof(proc->max_threads))) {
			ret = -EINVAL;
			goto err;
		}
		break;
```

这个处理很简单，proc指向的是`binder_proc`实体，这个结构体中有一个 max_threads 字段，这里就是将传递进来的值赋值到这个字段上，至于字段具体什么作用，后面会讲到。

回到前面，ProcessState 实例在新建的时候就打开了 Binder 驱动，接下去有一句很重要的代码:

```C++
// mmap the binder, providing a chunk of virtual address space to receive transactions.
mVMStart = mmap(0, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
```
`mmap`这个函数不是第一次见，看注释就明白这里是在做什么：这里和 SM 一样，也去 binder 驱动中挖一块内存，便于 Service 和驱动进行通信。按注释的说法，就是 "receive transactions"。

再回到 MS 的`main`函数，执行完第一句之后，我们创建了一个 ProcessState 实例，在这个实例里面打开了 Binder 驱动并做了内存映射，映射起始地址记录在 mVMStart 变量中。

以下是上面分析的函数调用的关系图：
<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/ProcessState实例化.png" alt="ProcessState实例化"/></div>

## 获取 SM 实例
第二行代码是:

```C++
sp<IServiceManager> sm = defaultServiceManager();
```
看函数名字以及返回的变量指针，就知道这和 SM 相关 —— 这里实际上就是在获取 SM 实例。这个方法来自哪里呢？"IServiceManager.cpp":

```C++
sp<IServiceManager> defaultServiceManager()
{
    if (gDefaultServiceManager != NULL) return gDefaultServiceManager;
    
    {
        AutoMutex _l(gDefaultServiceManagerLock);
        if (gDefaultServiceManager == NULL) {
            gDefaultServiceManager = interface_cast<IServiceManager>(
                ProcessState::self()->getContextObject(NULL));
        }
    }
    
    return gDefaultServiceManager;
}
```
又是一个单例方法，调用的是`interface_cast<IServiceManager>(ProcessState::self()->getContextObject(NULL));`来实例化返回对象的，这个方法包含很多内容，我们一步一步看。

### `ProcessState::self()->getContextObject(NULL)`
首先是`ProcessState::self()->getContextObject(NULL)`:

```C++
sp<IBinder> ProcessState::getContextObject(const sp<IBinder>& caller)
{
    return getStrongProxyForHandle(0);
}

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
```
实际上调用的是`getStrongProxyForHandle`获取的返回值，传入参数0，那么这个方法又在干啥呢？我把代码贴在一块了，很明显它又调用了另外一个方法`lookupHandleLocked`，继续传递参数0:

```C++
// 调用代码：lookupHandleLocked(0);
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
`mHandleToObject`是一个 Vector，它的元素是结构体 handle_entry，声明如下:

```C++
struct handle_entry {
	IBinder* binder;
	RefBase::weakref_type* refs;
};

Vector<handle_entry> mHandleToObject;
```
因为进程刚启动，因此`mHandleToObject`是空的，因此`N <= (size_t)handle`判断成立，这样就会新建一个 handle_entry 结构体实例，它的 binder 字段是 NULL 。

新建完成后继续回到`getStrongProxyForHandle`方法，它会取出 binder 字段，并作如下判断:

```C++
if (b == NULL || !e->refs->attemptIncWeak(this))
```
binder 字段为 NULL，因此最终会走到:

```C++
b = new BpBinder(handle); 
e->binder = b;
if (b) e->refs = b->getWeakRefs();
result = b;
```
这里会以 handle 为参数新建一个 BpBinder 对象，并赋值给 e 的 binder 字段，最后这个 b 就被赋值给 result 并作为方法返回值，最终返回到了`defaultServiceManager`方法中。
>BpBinder 后面再做介绍，我们先把 SM 的实例获取讲完。

因此下面这句代码:

```C++
interface_cast<IServiceManager>(ProcessState::self()->getContextObject(NULL))
```
实际上等于:

```C++
interface_cast<IServiceManager>(new BpBinder(0))
```

### `interface_cast`
接下来我们来啃`interface_cast`，它是一个内联的模板函数，定义在 "IInterface.h"中: 

```C++
template<typename INTERFACE>
inline sp<INTERFACE> interface_cast(const sp<IBinder>& obj)
{
    return INTERFACE::asInterface(obj);
}
```
所以前面那行代码又等价于:

```C++
IServiceManager::asInterface(new BpBinder(0))
```
`asInterface`这个方法实际又不是来自`IServiceManager`，而是来自`IInterface`（`IServiceManager`继承于`IInterface`），所以我们又回到了 "IInterface.h" 这个文件，在这个文件里面又发现这个方法实际上是定义在两个宏里面的:

```C++
#define DECLARE_META_INTERFACE(INTERFACE)                               \
    static const android::String16 descriptor;                          \
    static android::sp<I##INTERFACE> asInterface(                       \
            const android::sp<android::IBinder>& obj);                  \
    virtual const android::String16& getInterfaceDescriptor() const;    \
    I##INTERFACE();                                                     \
    virtual ~I##INTERFACE();                                            \


#define IMPLEMENT_META_INTERFACE(INTERFACE, NAME)                       \
    const android::String16 I##INTERFACE::descriptor(NAME);             \
    const android::String16&                                            \
            I##INTERFACE::getInterfaceDescriptor() const {              \
        return I##INTERFACE::descriptor;                                \
    }                                                                   \
    android::sp<I##INTERFACE> I##INTERFACE::asInterface(                \
            const android::sp<android::IBinder>& obj)                   \
    {                                                                   \
        android::sp<I##INTERFACE> intr;                                 \
        if (obj != NULL) {                                              \
            intr = static_cast<I##INTERFACE*>(                          \
                obj->queryLocalInterface(                               \
                        I##INTERFACE::descriptor).get());               \
            if (intr == NULL) {                                         \
                intr = new Bp##INTERFACE(obj);                          \
            }                                                           \
        }                                                               \
        return intr;                                                    \
    }                                                                   \
    I##INTERFACE::I##INTERFACE() { }                                    \
    I##INTERFACE::~I##INTERFACE() { }                                   \
```
恩...不出意外，子类中应该有相关使用声明。我们回到 "IServiceManager.h" 文件中，果然发现有这么一行代码:

```C++
DECLARE_META_INTERFACE(ServiceManager);
```
而在 "IServiceManager.cpp" 中，又有这么一条:

```C++
IMPLEMENT_META_INTERFACE(ServiceManager, "android.os.IServiceManager");
```
看样子搞懂这个宏，就知道`interface_cast`到底在干什么了。先来看看添加这条语句后倒地增加了什么样的方法:

```C++
const android::String16 IServiceManager::descriptor(NAME);
const android::String16& IServiceManager::getInterfaceDescriptor() const {
	return IServiceManager::descriptor;
} 
android::sp<IServiceManager> IServiceManager::asInterface(const android::sp<android::IBinder>& obj) {
	android::sp<IServiceManager> intr;
	//还记得obj是啥么？new BpBinder(0)，所以肯定不为null
	if (obj != NULL) {
		intr = static_cast<IServiceManager*>(obj->queryLocalInterface(IServiceManager::descriptor).get());
		if (intr == NULL) {
			intr = new BpServiceManager(obj);
		}
	}
	return intr;
}

IServiceManager:: IServiceManager() { }
IServiceManager::~ IServiceManager() { }  
```
哈，看到了我们最想看到的函数，也就是`asInterface`，这又是在干啥呢？这里调用了一个 `queryLocalInterface` 方法，我们看看 BpBinder 中的实现，在文件 "Binder.cpp" 中可以找到:

```C++
sp<IInterface>  IBinder::queryLocalInterface(const String16& descriptor)
{
    return NULL;
}
```
因此这里就会执行`new BpServiceManager(obj)`并返回对象，因此下面这句代码:

```C++
interface_cast<IServiceManager>(new BpBinder(0))
```
实际上就等于:

```C++
new BpServiceManager(new BpBinder(0))
```
通过这样一个语句，我们就可以获得一个 SM 实例。实际上到这里我们只知道 `defaultServiceManager()` 方法返回了一个`IServiceManager`的对象，实际指向的是 BpServiceManager 实例，具体这个对象和 SM 什么关系、如何帮助我们完成服务注册我们并不清楚。不要捉急，下面一节讲完就清楚了。

好了，喘口气，下面是调用关系图：
<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/defaultServiceManager.png" alt="defaultServiceManager调用"/></div>

看上去有点复杂，但是更复杂的还在后面。

## 服务注册
这里再贴一下`main`函数的代码:

```C++
using namespace android;

int main(int argc, char** argv)
{
	 // 实例化 Process
    sp<ProcessState> proc(ProcessState::self());
    
    // 获取 SM
    sp<IServiceManager> sm = defaultServiceManager();
    LOGI("ServiceManager: %p", sm.get());
    
    // 初始化服务
    AudioFlinger::instantiate();
    MediaPlayerService::instantiate();
    CameraService::instantiate();
    AudioPolicyService::instantiate();
    
    // 很重要的两个方法，后面分析
    ProcessState::self()->startThreadPool();
    IPCThreadState::self()->joinThreadPool();
}
```
从`main`函数看，到这里主要做了两件事情：

1. 初始化 ProcessState 变量，初始化过程主要是打开 binder 驱动，进行内存映射；
2. 获取 defaultServiceManager，它实际上是一个 BpServiceManager 对象；

如注释所写，接下去就是要初始化服务了。这下面有很多的服务进行初始化，包括音频，视频，照相机等，我们找`MediaPlayerService`为例进行研究。因为这部分分析涉及到的函数很多，调用链较长，因此这里先贴一下主要函数调用关系图（看不清可以右击查看大图）：
<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/服务注册函数调用链.png" alt="服务注册函数调用链"/></div>

首先来看 MediaPlayerService 的`instantiate()`方法了:

```C++
void MediaPlayerService::instantiate() {
    defaultServiceManager()->addService(
            String16("media.player"), new MediaPlayerService());
}
```
因为方法`defaultServiceManager()`是一个单例方法，因此这里获取到的其实还是前面新建的那个实例。接着调用的就是它的`addService`方法，我们终于走到了 __服务注册__ 的地方！

我们来看这个方法的实现（位于 "IServiceManager.cpp" 中）:

```C++
virtual status_t addService(const String16& name, const sp<IBinder>& service)
{
	Parcel data, reply;
	data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());
	data.writeString16(name);
	data.writeStrongBinder(service);
	status_t err = remote()->transact(ADD_SERVICE_TRANSACTION, data, &reply);
	return err == NO_ERROR ? reply.readExceptionCode() : err;
}
```
这里出了一个和 Android Java 层很相似的概念: Parcel，用于序列化对象。从代码看，它先序列化两个字符串，再序列化 service ，最后通过`remote()->transact()`发送到某个地方去了。让我们吸口气，一步步潜下去瞧瞧。

### Parcel 序列化
这里使用 Parcel 对象将数据全部打包，相关方法都集中在`Parcel.cpp`。前面写入了两个字符串，第一个是 RPC 头部，不需要多关注，实际写入值是 "android.os.IServiceManager" （这个方法建议读者自行查看，其实前面还写入了一些信息，会在下面指出），相关代码直接看 `IServiceManager::getInterfaceDescriptor()`即可；接着写入参数 name 的值，也就是传入的 "media.player"；最后写入了 Service，这里有必要好好看看方法`writeStrongBinder`:

```C++
status_t Parcel::writeStrongBinder(const sp<IBinder>& val)
{
    return flatten_binder(ProcessState::self(), val, this);
}

status_t flatten_binder(const sp<ProcessState>& proc,
    const sp<IBinder>& binder, Parcel* out)
{
	// 该结构体在字典中可查
    flat_binder_object obj;
    
    obj.flags = 0x7f | FLAT_BINDER_FLAG_ACCEPTS_FDS;
    
    //传入的是new MediaPlayerService()，因此不为NULL
    if (binder != NULL) {
    	  // 什么鬼？
        IBinder *local = binder->localBinder();
        if (!local) {
            BpBinder *proxy = binder->remoteBinder();
            if (proxy == NULL) {
                LOGE("null proxy");
            }
            const int32_t handle = proxy ? proxy->handle() : 0;
            obj.type = BINDER_TYPE_HANDLE;
            obj.handle = handle;
            obj.cookie = NULL;
        } else {
            obj.type = BINDER_TYPE_BINDER;
            obj.binder = local->getWeakRefs();
            obj.cookie = local;
        }
    } else {
        obj.type = BINDER_TYPE_BINDER;
        obj.binder = NULL;
        obj.cookie = NULL;
    }
    
    return finish_flatten_binder(binder, obj, out);
}
```
它实际调用的是`flatten_binder`方法，看名字意思是把一个 binder 对象扁平化，看上去很有序列化的味道（想象把一个对象转成 JSON），在这个方法里这个 binder 对象就是我们的`new MediaPlayerService()`。

后面还调用了很多 binder 的方法，比如`localBinder()`，`remoteBinder()`，开始大面积出现 IBinder，BpBinder 等概念，这里必须要分析一下一些类之间的关系，这也是为了方便后面分析。

Binder 机制在 Libraries 层面建立了一组对象，它们协同起来与驱动交互，完成 IPC 功能。这里就不一步步看代码了，直接上关系图:
<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/Binder-Lib层架构.png" alt="Binder Lib层架构"/></div>

这里直接画上了`MediaPlayerService`服务并标注了一些重要的属性和方法，好了，看着图我们继续往下走。
>PS: 凡是出现了 MediaPlayerService 的地方，实际都可以由别的 Service 替换。

首先我们要看的是`binder->localBinder();`，这个方法最先声明是在 IBinder 中，我搜索代码只能看到关于该方法如下的实现:

```C++
BBinder* IBinder::localBinder()
{
    return NULL;
}

BpBinder* IBinder::remoteBinder()
{
    return NULL;
}
```
也就是说返回为 NULL ，从 MediaPlayerService 到 IBinder 的继承树中再也没有别的实现，但是这个不符合常理：因为这里研究的是如何注册一个 Service，且 MediaPlayerService 确实又是本地的，按照道理`localBinder()`应该有返回，翻看几篇文章，也没有关于`localBinder()`方法的其余实现代码的展示，这里留作一个__【疑点】__。
>找的眼睛都快瞎了！

那么按照常理实际会执行下面的代码才对:

```C++
obj.type = BINDER_TYPE_BINDER;
obj.binder = local->getWeakRefs();
obj.cookie = local;
```
这里会产生一个type为`BINDER_TYPE_BINDER`的`flat_binder_object`结构体对象，binder 字段记录的是本地 binder 的弱引用对象，__cookie 中记录的则是本地服务实体的地址__。接着就调用`finish_flatten_binder()`方法:

```C++
inline static status_t finish_flatten_binder(
    const sp<IBinder>& binder, const flat_binder_object& flat, Parcel* out)
{
    return out->writeObject(flat, false);
}
```
这里就只是写入对象了：到这里为止，我们把 Service 的名字和 Service 本身写入到一个Parcel对象中去。现在 out 里面的数据应该是这样的：
<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/service注册transact数据.png" width="600" alt="service注册transact数据"/></div>

>data 中还有一些元数据，比如写入字符串之前会写入字符串长度，图中没有标注出来。

这张图非常重要，后面直到服务注册成功，这部分数据格式、内容都不会变化，因此在 SM 中注册时候，最后解析的格式还是按照这个顺序进行。__以下称这幅图为 "服务注册初始数据图"__ 。

### 传输数据
在准备好数据之后，接下来就是调用:

```C++
// data 大致就是上图的样子，reply 不为空
status_t err = remote()->transact(ADD_SERVICE_TRANSACTION, data, &reply);
```
这里顾名思义是在传输数据。`remote()`是什么呢？要知道`addService()`这个方法其实是来自类`BpServiceManager`的，它和上面类图中的`BpMediaPlayerService`位置类似，因此这个方法实际来自于`BpRefBase`。我们看一下这几个类的代码:

```C++
//#############  BpServiceManager  ###############
class BpServiceManager : public BpInterface<IServiceManager>
{
public:
    BpServiceManager(const sp<IBinder>& impl)
        : BpInterface<IServiceManager>(impl)
    {
    }
    ......
}

//#############  BpInterface  ###############
template<typename INTERFACE>
class BpInterface : public INTERFACE, public BpRefBase
{
public:
                                BpInterface(const sp<IBinder>& remote);

protected:
    virtual IBinder*            onAsBinder();
};

//#############  BpRefBase  ###############
class BpRefBase : public virtual RefBase
{
protected:
                            BpRefBase(const sp<IBinder>& o);
    virtual                 ~BpRefBase();
    virtual void            onFirstRef();
    virtual void            onLastStrongRef(const void* id);
    virtual bool            onIncStrongAttempted(uint32_t flags, const void* id);

    inline  IBinder*        remote()                { return mRemote; }
    inline  IBinder*        remote() const          { return mRemote; }

private:
                            BpRefBase(const BpRefBase& o);
    BpRefBase&              operator=(const BpRefBase& o);

    IBinder* const          mRemote;
    RefBase::weakref_type*  mRefs;
    volatile int32_t        mState;
};
```
BpRefBase 的实现如下：

```C++
// get() 方法应该是 sp 持有的，可以类比 Java 的弱引用
BpRefBase::BpRefBase(const sp<IBinder>& o)
    : mRemote(o.get()), mRefs(NULL), mState(0)
{
    extendObjectLifetime(OBJECT_LIFETIME_WEAK);

    if (mRemote) {
        mRemote->incStrong(this);           // Removed on first IncStrong().
        mRefs = mRemote->createWeak(this);  // Held for our entire lifetime.
    }
}
```

同样很隐蔽，mRemote的赋值来自`o.get()`，因此`remote()`返回的实际就是新建 BpServiceManager 时传入的参数，也就是`new BpBinder(0)`，换句话说，我们应该去 BpBinder 中寻找`transact()`函数的实现:

```C++
// 调用代码：status_t err = remote()->transact(ADD_SERVICE_TRANSACTION, data, &reply);
// 参数解释：data 就是服务注册初始数据图所示内容，reply 是一个 Parcel 对象，flags 默认是 0
status_t BpBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    // Once a binder has died, it will never come back to life.
    if (mAlive) {
        status_t status = IPCThreadState::self()->transact(
            mHandle, code, data, reply, flags);
        if (status == DEAD_OBJECT) mAlive = 0;
        return status;
    }

    return DEAD_OBJECT;
}
```
这里又转到`IPCThreadState::self()`里面去了，看上去似乎又是一个获取单例的方法，它的方法如下:

```C++
IPCThreadState* IPCThreadState::self()
{
    if (gHaveTLS) {
restart:
        const pthread_key_t k = gTLS;
        IPCThreadState* st = (IPCThreadState*)pthread_getspecific(k);
        if (st) return st;
        return new IPCThreadState;
    }
    
    if (gShutdown) return NULL;
    
    pthread_mutex_lock(&gTLSMutex);
    if (!gHaveTLS) {
        if (pthread_key_create(&gTLS, threadDestructor) != 0) {
            pthread_mutex_unlock(&gTLSMutex);
            return NULL;
        }
        gHaveTLS = true;
    }
    pthread_mutex_unlock(&gTLSMutex);
    goto restart;
}
```
这个方法的实现涉及到一些 Linux 函数以及线程知识：它并不是前面所说的获取单例，而是获取线程变量副本。相似的机制 Java 中也有，可以查看文章 [ThreadLocal 源码剖析](http://www.muzileecoding.com/java/Java-threadlocal.html)，简单来说，就是为每个线程创建一个变量的副本，保证线程之间不共享原有的变量，保证线程内部只有一个该变量的副本。关于这个知识点，在网上找到一篇文章:[pthread_key_t和pthread_key_create()详解](http://blog.csdn.net/lmh12506/article/details/8452700)，读者读完这两篇文章之后，再看这个函数就会比较清晰，这里就不做详细分析了。总结一下该函数: 为该进程中的每一个线程创建唯一一个 IPCThreadState 对象。
>这个创建并不是线程一启动就进行的，而是在这个变量中获取该变量的时候进行创建的。

好，咱继续走！创建之后调用的是 IPCThreadState 的`transact()`函数: 

```C++
// 调用代码: status_t status = IPCThreadState::self()->transact(mHandle, code, data, reply, flags);
// 参数解释: handle 是 BpBinder 的实例化参数，也就是0，code 是 ADD_SERVICE_TRANSACTION，flags是0
status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags)
{
    status_t err = data.errorCheck();

    flags |= TF_ACCEPT_FDS;

	 // 数据没有问题，判断通过
    if (err == NO_ERROR) {
        err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, NULL);
    }
    
    if (err != NO_ERROR) {
        if (reply) reply->setError(err);
        return (mLastError = err);
    }
    
    if ((flags & TF_ONE_WAY) == 0) {
        if (reply) {
            err = waitForResponse(reply);
        } else {
            Parcel fakeReply;
            err = waitForResponse(&fakeReply);
        }
    } else {
        err = waitForResponse(NULL, NULL);
    }
    
    return err;
}
```
这个方法删除一些Log和无用代码之后，大致就是上面这个样子。首先进来调用的就是`writeTransactionData()`方法:

```C++
// 调用代码: writeTransactionData(BC_TRANSACTION, flags, handle, code, data, NULL);
// 参数解释: handle 为0，code 为 ADD_SERVICE_TRANSACTION， binderFlags 是 TF_ACCEPT_FDS;
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
我们终于遇到一直在等待的重要结构体了: binder_transaction_data。
> 如果读者看过推荐文章①，一定知道 binder_transaction 表示IPC之间的一个请求，而  binder_transaction_data 就是为了这样一个事务包装的结构体对象。

这里主要将之前写入 Parcel 的数据再设置到 binder_transaction_data 结构体中：

```C++
tr.data_size = data.ipcDataSize();  //记录数据大小
tr.data.ptr.buffer = data.ipcData();  // 记录数据内容
tr.offsets_size = data.ipcObjectsCount()*sizeof(size_t);
tr.data.ptr.offsets = data.ipcObjects();
```
这四个数据是有必要说道说道的，前两个很简单，就是我们之前写入的数据尺寸以及内容（注意，内容没有变），内容就是"服务注册初始数据图"所绘制的。后两个要注意了：这两个数据就是为了 flat_binder_object 准备的，在最终传输的数据中，我们是需要记录哪些位置存放着 flat_binder_object 对象的，其中`tr.data.ptr.offsets`中存放的就是每个对象实际的偏移量，`tr.offsets_size`中存放的就是这些偏移数据的大小 —— 记录这些元数据是为后面可以再反序列化出 flat_binder_object 对象。

那么问题来了，mOut 是什么东西呢？它也是个 Parcel 对象，它在`IPCThreadState`初始化的时候被设置相应的初始值：

```C++
IPCThreadState::IPCThreadState()
    : mProcess(ProcessState::self()),
      mMyThreadId(androidGetTid()),
      mStrictModePolicy(0),
      mLastTransactionBinderFlags(0)
{
    pthread_setspecific(gTLS, this);
    clearCaller();
    // 注意，这两个变量的容量都不为0
    mIn.setDataCapacity(256);
    mOut.setDataCapacity(256);
}
```
执行完成`writeTransactionData`之后数据都被写入了 mOut 内，那是在哪里发送数据到驱动层的呢？回到`IPCThreadState::transact()`方法，在调用`writeTransactionData()`方法之后，做了一些错误检测，之后就是调用函数`waitForResponse()`，代码删减一下如下:

```C++
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
    int32_t cmd;
    int32_t err;

    while (1) {
        if ((err = talkWithDriver()) < NO_ERROR) break;
        err = mIn.errorCheck();
        if (err < NO_ERROR) break;
        if (mIn.dataAvail() == 0) continue;
        
        // 注意：下面有一段很长的命令解析，这段数据来自驱动。等分析完数据的发送之后，我们还会回到这里
        cmd = mIn.readInt32();

        switch (cmd) {
        default:
            err = executeCommand(cmd);
            if (err != NO_ERROR) goto finish;
            break;
        }
    }
    return err;
}
```
这里又调用了一个`talkWithDriver()`函数:

```C++
// 方法声明的时候默认 doReceive 是 true
status_t IPCThreadState::talkWithDriver(bool doReceive)
{
    LOG_ASSERT(mProcess->mDriverFD >= 0, "Binder driver is not opened");
    
    binder_write_read bwr;
    
    // Is the read buffer empty?
    // 为 true
    const bool needRead = mIn.dataPosition() >= mIn.dataSize();
    
    // We don't want to write anything if we are still reading
    // from data left in the input buffer and the caller
    // has requested to read the next data.
    const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;
    
    // 将 mOut中 的数据写入到bwr中
    bwr.write_size = outAvail;
    bwr.write_buffer = (long unsigned int)mOut.data();

    // This is what we'll read. 这两个变量名取得很好
    if (doReceive && needRead) {
    	//注意：前面说了，这个 mIn 在 IPCThreadState 中初始化的时候就被设置为 256 的容量
        bwr.read_size = mIn.dataCapacity();
        bwr.read_buffer = (long unsigned int)mIn.data();
    } else {
        bwr.read_size = 0;
        bwr.read_buffer = 0;
    }
    
    // Return immediately if there is nothing to do.
    if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;

    bwr.write_consumed = 0;
    bwr.read_consumed = 0;
    status_t err;
    do {
    #if defined(HAVE_ANDROID_OS)
        if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
            err = NO_ERROR;
        else
            err = -errno;
    #else
        err = INVALID_OPERATION;
    #endif

    } while (err == -EINTR);

    if (err >= NO_ERROR) {
        if (bwr.write_consumed > 0) {
            if (bwr.write_consumed < (ssize_t)mOut.dataSize())
                mOut.remove(0, bwr.write_consumed);
            else
                mOut.setDataSize(0);
        }
        if (bwr.read_consumed > 0) {
            mIn.setDataSize(bwr.read_consumed);
            mIn.setDataPosition(0);
        }
        return NO_ERROR;
    }
    
    return err;
}
```
在和驱动交互之前，还是需要把数据写入 binder_write_read 结构体中，然后使用`ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr)`命令将数据写入驱动。

>这里是第二次遇到这种模式了。这里做了一点小修改：通过两个 Parcel 对象进行数据读写，bwr 是这两个对象和驱动层交互的介质。

回想一下这个时候 `mOut.data()`，也就是bwr.write_buffer中是什么内容？ __是一个命令 `BC_TRANSACTION` + `binder_transaction_data`结构体，`binder_transaction_data`结构体中记录着服务注册初始数据图中所描述的 data 数据结构以及一些元数据(具体转换都在`IPCThreadState::writeTransactionData()`方法中)。__

接着来！看来要知道如何注册的，问题就在于解析`ioctl`期间到底发生了什么事情，我们来看一下驱动层对于`BINDER_WRITE_READ`的命令的解析，之前已经有过类似的例子，执行如下：

```C++
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
    ......
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
        
        if (bwr.write_size > 0) {//写入内容
            //参数：前两个表示发起传输动作的进程和线程
            ret = binder_thread_write(proc, thread, (void __user *)bwr.write_buffer, bwr.write_size, &bwr.write_consumed);
            trace_binder_write_done(ret);
            if (ret < 0) {
                bwr.read_consumed = 0;
                if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
                    ret = -EFAULT;
                goto err;
            }
        }
        
        // write之后紧接着调用read
        if (bwr.read_size > 0) {//读取内容
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
        //再写回去
        if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {
            ret = -EFAULT;
            goto err;
        }
        break;
    }
    ......
    ret = 0;
    ......
    return ret;
}
```
这里`bwr.write_size`和`bwr.read_size`都大于 0，我们先来看`binder_thread_write`。这个函数解析的时候会首先读取一个 cmd 参数，从前面的代码中不难发现，这个 cmd 是`BC_TRANSACTION`，它的处理如下:

```C++
case BC_TRANSACTION:
case BC_REPLY: {
		struct binder_transaction_data tr;
		// 从用户空间拷贝到内核空间
		if (copy_from_user(&tr, ptr, sizeof(tr)))
			return -EFAULT;
		// 已经读取了tr，向后移动指针
		ptr += sizeof(tr);
		
		binder_transaction(proc, thread, &tr, cmd == BC_REPLY);
		break;
}
```
前面几步都已经比较熟悉，数据读取和写入一一对应，binder_transaction_data 对应数据被复制到驱动层。最后调用的是`binder_transaction`函数进行解析，前面两个参数很熟悉，最后一个参数为 false 。
>让我哭一会儿，这个方法在源码里面是一个 400 多行的函数....我要罢工了！

删删删，删完之后还剩下这么多: 

```C++
// tr 的 write 数据中有 binder 节点信息，就是之前写入mOut中的数据
static void binder_transaction(struct binder_proc *proc,
                   struct binder_thread *thread,
                   struct binder_transaction_data *tr, int reply)
{
    struct binder_transaction *t;
    struct binder_work *tcomplete;
    size_t *offp, *off_end;
    struct binder_proc *target_proc;
    struct binder_thread *target_thread = NULL;
    struct binder_node *target_node = NULL;
    struct list_head *target_list;
    wait_queue_head_t *target_wait;
    struct binder_transaction *in_reply_to = NULL;
    struct binder_transaction_log_entry *e;
    uint32_t return_error;
    e = binder_transaction_log_add(&binder_transaction_log);
    e->call_type = reply ? 2 : !!(tr->flags & TF_ONE_WAY);
    e->from_proc = proc->pid;
    e->from_thread = thread->pid;
    e->target_handle = tr->target.handle;
    e->data_size = tr->data_size;
    e->offsets_size = tr->offsets_size;
    // transaction 和 reply 是同一个处理流程，但这里是transaction，因此 reply 为false
    if (reply) {
        //......
    } else {
        // 在 IPCThreadState::writeTransactionData 可以看到 handle 被赋值了，
        // 而且解析中特意指出这个 handle 为 0，因此这个判断不成立
        if (tr->target.handle) {
            struct binder_ref *ref;
            ref = binder_get_ref(proc, tr->target.handle);
            if (ref == NULL) {
                binder_user_error("binder: %d:%d got "
                    "transaction to invalid handle\n",
                    proc->pid, thread->pid);
                return_error = BR_FAILED_REPLY;
                goto err_invalid_target_handle;
            }
            target_node = ref->node;
        } else { 
            // target_node: 目标节点，前面分析说了，binder_node 表示一个服务节点，
            // 这里的目标节点的含义就是发送请求的目标节点，这里是在注册服务，所以目标
            // 节点就是 binder_context_mgr_node，也就是 SM 注册成为守护进程的时
            // 候生成的节点：这也就是说 handle = 0就代表着指向 SM 服务节点。
            target_node = binder_context_mgr_node;
            if (target_node == NULL) {
                return_error = BR_DEAD_REPLY;
                goto err_no_context_mgr_node;
            }
        }
        e->to_node = target_node->debug_id;

        // 获取目标节点所在的进程，也就是 SM 服务进程(在binder_new_node()方法中新建node的时候进行了赋值)
        target_proc = target_node->proc;
        if (target_proc == NULL) {
            return_error = BR_DEAD_REPLY;
            goto err_dead_binder;
        }
        
        // 权限检测
        if (security_binder_transaction(proc->tsk, target_proc->tsk) < 0) {
            return_error = BR_FAILED_REPLY;
            goto err_invalid_target_handle;
        }

        // thread->transaction_stack 为 null，不成立
        if (!(tr->flags & TF_ONE_WAY) && thread->transaction_stack) {
            struct binder_transaction *tmp;
            tmp = thread->transaction_stack;
            if (tmp->to_thread != thread) {
                binder_user_error("binder: %d:%d got new "
                    "transaction with bad transaction stack"
                    ", transaction %d has target %d:%d\n",
                    proc->pid, thread->pid, tmp->debug_id,
                    tmp->to_proc ? tmp->to_proc->pid : 0,
                    tmp->to_thread ?
                    tmp->to_thread->pid : 0);
                return_error = BR_FAILED_REPLY;
                goto err_bad_call_stack;
            }
            while (tmp) {
                if (tmp->from && tmp->from->proc == target_proc)
                    target_thread = tmp->from;
                tmp = tmp->from_parent;
            }
        }
    }

    // 如果找到了合适的线程，获取目标线程的两组List，实际这里 target_thread 为 NULL
    if (target_thread) {
        e->to_thread = target_thread->pid;
        target_list = &target_thread->todo;
        target_wait = &target_thread->wait;
    } else { // 否则获取目标进程的两组List
        target_list = &target_proc->todo;
        target_wait = &target_proc->wait;
    }
    e->to_proc = target_proc->pid;
    /* TODO: reuse incoming transaction for reply */
    // t 是 binder_transaction 结构体，该结构体可以在字典中查询
    t = kzalloc(sizeof(*t), GFP_KERNEL);
    if (t == NULL) {
        return_error = BR_FAILED_REPLY;
        goto err_alloc_t_failed;
    }
    binder_stats_created(BINDER_STAT_TRANSACTION);
    // tcomplete 是 binder_work 结构体，该结构体可以在字典中查询
    tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);
    if (tcomplete == NULL) {
        return_error = BR_FAILED_REPLY;
        goto err_alloc_tcomplete_failed;
    }
    binder_stats_created(BINDER_STAT_TRANSACTION_COMPLETE);
    t->debug_id = ++binder_last_id;
    e->debug_id = t->debug_id;
	
	// reply 为 false， tr-flags 为 TF_ACCEPT_FDS，因此成立
    if (!reply && !(tr->flags & TF_ONE_WAY))
        t->from = thread; // 指向当前线程
    else 
        t->from = NULL;

    // 为 binder_transaction 结构体设置属性，包括把该 transaction 交给哪个 proc，哪个 thread
    t->sender_euid = proc->tsk->cred->euid;
    t->to_proc = target_proc;
    t->to_thread = target_thread;
    // code 为 ADD_SERVICE_TRANSACTION
    t->code = tr->code;
    t->flags = tr->flags;
    t->priority = task_nice(current);
    trace_binder_transaction(reply, t, target_node);
    // 注意，这是在target_proc中进行内存分配，包括数据填入
    t->buffer = binder_alloc_buf(target_proc, tr->data_size,
        tr->offsets_size, !reply && (t->flags & TF_ONE_WAY));
    if (t->buffer == NULL) {
        return_error = BR_FAILED_REPLY;
        goto err_binder_alloc_buf_failed;
    }
    t->buffer->allow_user_free = 0;
    t->buffer->debug_id = t->debug_id;
    t->buffer->transaction = t;
    t->buffer->target_node = target_node;
    trace_binder_transaction_alloc_buf(t->buffer);
    // 增加引用计数
    if (target_node)
        binder_inc_node(target_node, 1, 0, NULL);
    offp = (size_t *)(t->buffer->data + ALIGN(tr->data_size, sizeof(void *)));
    
    // 拷贝 transaction 数据
    if (copy_from_user(t->buffer->data, tr->data.ptr.buffer, tr->data_size)) {
        binder_user_error("binder: %d:%d got transaction with invalid "
            "data ptr\n", proc->pid, thread->pid);
        return_error = BR_FAILED_REPLY;
        goto err_copy_data_failed;
    }
    
    // 拷贝 transaction offset指针
    if (copy_from_user(offp, tr->data.ptr.offsets, tr->offsets_size)) {
        binder_user_error("binder: %d:%d got transaction with invalid "
            "offsets ptr\n", proc->pid, thread->pid);
        return_error = BR_FAILED_REPLY;
        goto err_copy_data_failed;
    }
    
    // offsets 数据记录末尾
    off_end = (void *)offp + tr->offsets_size;

    // 解析 flat_binder_object 
    for (; offp < off_end; offp++) {
        struct flat_binder_object *fp;
        if (*offp > t->buffer->data_size - sizeof(*fp) ||
            t->buffer->data_size < sizeof(*fp) ||
            !IS_ALIGNED(*offp, sizeof(void *))) {
            binder_user_error("binder: %d:%d got transaction with "
                "invalid offset, %zd\n",
                proc->pid, thread->pid, *offp);
            return_error = BR_FAILED_REPLY;
            goto err_bad_offset;
        }
        // 我们之前存进去的扁平化的flat_binder_object对象
        fp = (struct flat_binder_object *)(t->buffer->data + *offp);
        switch (fp->type) {
        case BINDER_TYPE_BINDER:
        case BINDER_TYPE_WEAK_BINDER: {
            struct binder_ref *ref;
            // 获取Binder实体节点，这个proc表示当前进程，也就是发起命令的进程，
            // 初次注册，通过binder_get_node拿不到Binder节点
            struct binder_node *node = binder_get_node(proc, fp->binder);
            if (node == NULL) {
                // 新建binder节点，这个方法之前分析过了：会在proc的nodes红黑树上
                // 创建对应的binder节点，表示该proc提供该服务，这里要注意的是，和
                // SM 不同，这里后面两个参数都实际传递了值进去，因此，binder_node
                // 可以根据cookie来寻找 Libraries 层的服务位置，也就是
                // BBinder —— 我们要注册的服务。
                node = binder_new_node(proc, fp->binder, fp->cookie);
                if (node == NULL) {
                    return_error = BR_FAILED_REPLY;
                    goto err_binder_new_node_failed;
                }
                node->min_priority = fp->flags & FLAT_BINDER_FLAG_PRIORITY_MASK;
                node->accept_fds = !!(fp->flags & FLAT_BINDER_FLAG_ACCEPTS_FDS);
            }
            // 保证指针值一致
            if (fp->cookie != node->cookie) {
                binder_user_error("binder: %d:%d sending u%p "
                    "node %d, cookie mismatch %p != %p\n",
                    proc->pid, thread->pid,
                    fp->binder, node->debug_id,
                    fp->cookie, node->cookie);
                goto err_binder_get_ref_for_node_failed;
            }
            if (security_binder_transfer_binder(proc->tsk, target_proc->tsk)) {
                return_error = BR_FAILED_REPLY;
                goto err_binder_get_ref_for_node_failed;
            }

            // 创建好对应的 binder_node 之后，去对应的 target_proc中寻找是否有该node的引用
            // 没有的话，binder_get_ref_for_node 会新建一个 binder_ref 对象，插入到我们之
            // 前提到过的 binder_proc 的四棵红黑树中的两棵里面去，分别是 refs_by_node 和
            // refs_by_desc。注意，binder_ref 中也包含了该node的引用。
            //
            // binder_get_ref_for_node 寻找 binder_ref 的过程是直接通过比较node来进行的。
            ref = binder_get_ref_for_node(target_proc, node);
            if (ref == NULL) {
                return_error = BR_FAILED_REPLY;
                goto err_binder_get_ref_for_node_failed;
            }
            // 前面提到过，binder 驱动会更改类型
            if (fp->type == BINDER_TYPE_BINDER)
                fp->type = BINDER_TYPE_HANDLE;
            else
                fp->type = BINDER_TYPE_WEAK_HANDLE;

            // 同前面提到过，这里也会更改数据。
            // desc 值，其实就是该 binder_node 节点在 target_proc 中的 handle，通过这个handle
            // 可以直接找到对应的 binder_node 节点，也就是找到对应的服务，这里解释的可能不清楚，等
            // 下一篇文章讲述获取服务的时候，这里就很清楚了。
            fp->handle = ref->desc;
            binder_inc_ref(ref, fp->type == BINDER_TYPE_HANDLE,
                       &thread->todo);
            trace_binder_transaction_node_to_ref(t, node, ref);
        } break;
        ......
    }
    if (reply) {
        ......
    } else if (!(t->flags & TF_ONE_WAY)) { // 成立
        BUG_ON(t->buffer->async_transaction != 0);
        t->need_reply = 1;
        t->from_parent = thread->transaction_stack;
        // 记录这个transaction，等下 SM 回复的时候需要使用这个 transaction
        thread->transaction_stack = t; 
    } else {
        .....
    }
    
    // t->work 是 binder_work 结构体，该结构体在字典中可查询
    t->work.type = BINDER_WORK_TRANSACTION;
    // 添加 t 到目标进程的todo list中去
    list_add_tail(&t->work.entry, target_list);
    tcomplete->type = BINDER_WORK_TRANSACTION_COMPLETE;
    // 添加到当前线程的 todo list中去
    list_add_tail(&tcomplete->entry, &thread->todo);
    // 唤醒队列
    if (target_wait)
        wake_up_interruptible(target_wait);
    return;
......
}
```
还是有200多行...这里我直接在代码里面添加注释来解析代码。到这里我们在目标线程中插入了一个`binder_transaction`，这里的目标进程就是 SM 进程；并在自己线程的 todo 列表中插入了一个`binder_work`结构体实例（这个结构体，等会儿 SM 回复的时候需要使用）。

里面有一个重要函数，这里也使用这种方式进行分析:

```C++
// 参数解释：proc 是目标进程，这里就是指 SM，node 是指服务节点，这里代指 MS 服务 binder 节点
static struct binder_ref *binder_get_ref_for_node(struct binder_proc *proc,
                          struct binder_node *node)
{
    struct rb_node *n;
    // 红黑树
    struct rb_node **p = &proc->refs_by_node.rb_node;
    struct rb_node *parent = NULL;
    // 该结构体字典中可查
    struct binder_ref *ref, *new_ref;

    // 通过比较 node 来寻找对应的 binder_ref 对象
    while (*p) {
        parent = *p;
        ref = rb_entry(parent, struct binder_ref, rb_node_node);
        if (node < ref->node)
            p = &(*p)->rb_left;
        else if (node > ref->node)
            p = &(*p)->rb_right;
        else
            return ref;
    }
    new_ref = kzalloc(sizeof(*ref), GFP_KERNEL);
    if (new_ref == NULL)
        return NULL;
    binder_stats_created(BINDER_STAT_REF);
    new_ref->debug_id = ++binder_last_id;
    new_ref->proc = proc;
    // 注意：binder_ref 中是直接携带 node 引用的，因为在驱动中node以及引用都在
    // 同一个内存空间，因此这样的赋值是可以的。
    new_ref->node = node;
    rb_link_node(&new_ref->rb_node_node, parent, p);
    // 将红黑树节点插入到 refs_by_node 中
    rb_insert_color(&new_ref->rb_node_node, &proc->refs_by_node);
    new_ref->desc = (node == binder_context_mgr_node) ? 0 : 1;

    // 生成desc值，实际就是一个 binder_node 节点在 target_proc 中的 handle 值
    for (n = rb_first(&proc->refs_by_desc); n != NULL; n = rb_next(n)) {
        ref = rb_entry(n, struct binder_ref, rb_node_desc);
        if (ref->desc > new_ref->desc)
            break;
        new_ref->desc = ref->desc + 1;
    }

    //红黑树操作
    ......
    return new_ref;
}
```
总结一下这一段代码中有关 binder_node 和 binder_ref 的操作：

__当一个`BC_TRANSACTION`命令被处理的时候，binder 驱动会根据携带的元数据解析数据内容，查看是否有 binder 节点信息，如果存在 binder 节点，那么会检测当前进程的`binder_proc`结构体的红黑树中是否存在代表该节点的`binder_node`结构体，如果没有则需要新建并放到红黑树中去，其次会检测目标进程是否有对该节点的引用，如果没有，则需要新建`binder_ref`结构体并挂载到目标进程的红黑树中__。

### 与 SM 接头
列出的`binder_transaction`方法代码的最后几句如下:

```C++
//唤醒队列
if (target_wait)
	wake_up_interruptible(target_wait);
```
还记得我们分析 SM 的时候，在循环监听部分，它在最后做的是什么吗？

```C++
ret = wait_event_freezable_exclusive(proc->wait, binder_has_proc_work(proc, thread));
```
一看就是对应的好么！碰巧上面的 target_wait 就是 SM 的 wait 队列，因此`wake_up_interruptible`就唤醒了正在等待的 SM 服务。所以我们又转回 SM。但在转到 SM 之前，我们得知道 MS 后面会做什么？

### 等待 SM 的答复
`binder_transaction`方法执行完毕之后，`binder_thread_write`函数也执行完毕，返回到`binder_ioctl`函数。接下去要执行如下代码: 

```C++

// write之后紧接着调用read
if (bwr.read_size > 0) {//读取内容
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
//再写回去
if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {
	ret = -EFAULT;
	goto err;
}
```
`bwr` 变量在传入之前赋值如下：

```C++
bwr.read_size = mIn.dataCapacity(); // 256
bwr.read_buffer = (long unsigned int)mIn.data();
bwr.read_consumed = 0;
```
`bwr.read_size > 0`这个判断是成立的，因此我们进入`binder_thread_read`函数：

```C++
static int binder_thread_read(struct binder_proc *proc,
                  struct binder_thread *thread,
                  void  __user *buffer, int size,
                  signed long *consumed, int non_block)
{
	// ptr 指向 read_buffer的起始位置
    void __user *ptr = buffer + *consumed;
    void __user *end = buffer + size;
    int ret = 0;
    int wait_for_proc_work;
    // 入参为0，写入命令 BR_NOOP
    if (*consumed == 0) {
        if (put_user(BR_NOOP, (uint32_t __user *)ptr))
            return -EFAULT;
        ptr += sizeof(uint32_t);
    }
retry:
    // 根据前面 binder_transaction 的代码，两者都不为空
    wait_for_proc_work = thread->transaction_stack == NULL &&
                list_empty(&thread->todo);

    //设置状态为等待
    thread->looper |= BINDER_LOOPER_STATE_WAITING;
    if (wait_for_proc_work)
        proc->ready_threads++;
    binder_unlock(__func__);
    if (wait_for_proc_work) {
        ......
    } else {
        if (non_block) {
            if (!binder_has_thread_work(thread))
                ret = -EAGAIN;
        } else // 执行这里，但是 binder_has_thread_work 返回 true，可以见下面函数的解析
            ret = wait_event_freezable(thread->wait, binder_has_thread_work(thread));
    }
    binder_lock(__func__);
    if (wait_for_proc_work)
        proc->ready_threads--;
    thread->looper &= ~BINDER_LOOPER_STATE_WAITING;
    if (ret)
        return ret;
    while (1) {
        uint32_t cmd;
        struct binder_transaction_data tr;
        struct binder_work *w;
        struct binder_transaction *t = NULL;
        if (!list_empty(&thread->todo)) // 此时不为空，这里会取一个 binder_work 出来，之后变为空
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
        ......
        case BINDER_WORK_TRANSACTION_COMPLETE: { // 可以看 binder_transaction 函数
            cmd = BR_TRANSACTION_COMPLETE;
            // 又写入一个新的命令：BR_TRANSACTION_COMPLETE
            if (put_user(cmd, (uint32_t __user *)ptr))
                return -EFAULT;
            ptr += sizeof(uint32_t);
            binder_stat_br(proc, thread, cmd);
            // 资源释放
            list_del(&w->entry);
            kfree(w);
            binder_stats_deleted(BINDER_STAT_TRANSACTION_COMPLETE);
        } break;
        ......
        }
        if (!t)
            continue;
        ......
        break;
    }
    ......
    return 0;
}
```
相关代码解析已经写作代码注释，可以看到它往`bwr`中参数写入了两个命令：`BR_NOOP`和`BR_TRANSACTION_COMPLETE`，接着返回到`binder_ioctl`函数，执行:

```C++
if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {
	ret = -EFAULT;
	goto err;
}
```
将数据又写回到用户空间，在用户空间等待的，自然是`IPCThreadState::talkWithDriver()`方法中的如下代码：

```C++
if (err >= NO_ERROR) {
	if (bwr.write_consumed > 0) {
		if (bwr.write_consumed < (ssize_t)mOut.dataSize())
			mOut.remove(0, bwr.write_consumed);
		else // 写入数据全部读取完毕，清零
			mOut.setDataSize(0);
	}
	if (bwr.read_consumed > 0) {
		// 设置数据所在范围
		mIn.setDataSize(bwr.read_consumed);
		mIn.setDataPosition(0);
	}
	return NO_ERROR;
}
```
这里主要是数据清理以及标记有效数据存储在 mIn 中的什么位置，这里的有效数据就是两个命令，我们继续往下看，就又回到了`IPCThreadState::waitForResponse`方法：

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
        case BR_TRANSACTION_COMPLETE:
            if (!reply && !acquireResult) goto finish;
            break;
            ......
        default:
            err = executeCommand(cmd); // 执行 executeCommand 方法
            if (err != NO_ERROR) goto finish;
            break;
        }
    }

finish:
    if (err != NO_ERROR) {
        if (acquireResult) *acquireResult = err;
        if (reply) reply->setError(err);
        mLastError = err;
    }
    
    return err;
}

status_t IPCThreadState::executeCommand(int32_t cmd)
{
    BBinder* obj;
    RefBase::weakref_type* refs;
    status_t result = NO_ERROR;
    
    switch (cmd) {
    ......
    case BR_NOOP: // 不做任何处理
        break;
    }

    ......
    return result;
}
```
这时候 mIn 第一个读出的命令是 `BR_NOOP`，什么也不做。`while`循环继续执行，执行的时候出现了什么情况呢？又进入了`IPCThreadState::talkWithDriver()`：（PS：又得走一遍？）

```C++
// doReceive 默认为 true
status_t IPCThreadState::talkWithDriver(bool doReceive)
{
	binder_write_read bwr;
    
    // Is the read buffer empty?
    const bool needRead = mIn.dataPosition() >= mIn.dataSize();
    
    // We don't want to write anything if we are still reading
    // from data left in the input buffer and the caller
    // has requested to read the next data.
    const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;
    
    bwr.write_size = outAvail;
    bwr.write_buffer = (long unsigned int)mOut.data();

    // This is what we'll read.
    if (doReceive && needRead) {
        bwr.read_size = mIn.dataCapacity();
        bwr.read_buffer = (long unsigned int)mIn.data();
    } else { // 走这里
        bwr.read_size = 0;
        bwr.read_buffer = 0;
    }
    
    // Return immediately if there is nothing to do.
    if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;

    ......
}
```
mIn 的数据在驱动层返回的时候设置过一次：

```C++
mIn.setDataSize(bwr.read_consumed);
mIn.setDataPosition(0);
```
dataPosition 代表有效数据的起始点，dataSize 代表数据的大小，前面说过这个时候数据包含两个命令，现在只读取了一个，还剩一个，因此`mIn.dataPosition() < mIn.dataSize()`，因此`needRead`为false，`doReceive`为false，因此 outAvail 是 0，那么 `(bwr.write_size == 0) && (bwr.read_size == 0)`成立，直接返回 `NO_ERROR`。

好了，这里并没有走的很深入，我们又回到`IPCThreadState::waitForResponse`方法：

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
        case BR_TRANSACTION_COMPLETE:
        	// reply 是传进来的，这里只是break，循环继续
            if (!reply && !acquireResult) goto finish;
            break;
            ......
        }
    }

finish:
    ......
    
    return err;
}
```
命令读完啦，第三次进入`talkWithDriver()`：

```C++
status_t IPCThreadState::talkWithDriver(bool doReceive)
{
    LOG_ASSERT(mProcess->mDriverFD >= 0, "Binder driver is not opened");
    
    binder_write_read bwr;
    
    // Is the read buffer empty?
    // 为 true，经过两轮循环之后，mIn 中的数据已经读取完毕
    const bool needRead = mIn.dataPosition() >= mIn.dataSize();
    
    // We don't want to write anything if we are still reading
    // from data left in the input buffer and the caller
    // has requested to read the next data.
    // mOut 前面已经被清零，因此 outAvail 为 0
    const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;
    
    bwr.write_size = outAvail;
    bwr.write_buffer = (long unsigned int)mOut.data();

    // This is what we'll read.
    if (doReceive && needRead) {
        bwr.read_size = mIn.dataCapacity(); // 不为 0
        bwr.read_buffer = (long unsigned int)mIn.data();
    } else {
        bwr.read_size = 0;
        bwr.read_buffer = 0;
    }

    // Return immediately if there is nothing to do.
    if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;

    bwr.write_consumed = 0;
    bwr.read_consumed = 0;
    status_t err;
    do {
        // 进入binder驱动
        if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
            err = NO_ERROR;
        else
            err = -errno;
    } while (err == -EINTR);

    .....
    
    return err;
}
```
代码注释如上，不多解释，咱又进入驱动层了，因为只有`bwr.read_size`大于0，因此直接进入函数`binder_thread_read`：

```C++
static int binder_thread_read(struct binder_proc *proc,
                  struct binder_thread *thread,
                  void  __user *buffer, int size,
                  signed long *consumed, int non_block)
{
    void __user *ptr = buffer + *consumed;
    void __user *end = buffer + size;
    int ret = 0;
    int wait_for_proc_work;
    if (*consumed == 0) { // 同样滴，这里还是 0
        if (put_user(BR_NOOP, (uint32_t __user *)ptr))
            return -EFAULT;
        ptr += sizeof(uint32_t);
    }
retry:
    // thread->transaction_stack 不为空，&thread->todo 为空，因此 wait_for_proc_work 为 false
    wait_for_proc_work = thread->transaction_stack == NULL &&
                list_empty(&thread->todo);
    //设置状态为等待
    thread->looper |= BINDER_LOOPER_STATE_WAITING;
    if (wait_for_proc_work)
        proc->ready_threads++;
    binder_unlock(__func__);

    if (wait_for_proc_work) {
        ......
    } else {
        if (non_block) {
            if (!binder_has_thread_work(thread))
                ret = -EAGAIN;
        } else // 走到这里，这时候 thread 的 todo 列表为空，阻塞
            ret = wait_event_freezable(thread->wait, binder_has_thread_work(thread));
    }
    ....
    return 0;
}
```
到这里，最终阻塞在了`thread`的`todo`队列上。

也就是说，MediaPlayerService 在发起一个注册请求的同时，通过向自己当前线程的`todo`队列上挂载 binder_work 对象，可以很顺利的向 Libraries 层发送消息。而 Libraries 此时处于一个循环读取数据的状态，在没有数据可以读取的情况下，就会被阻塞住，直到`todo`再次不为空。

## SM 的处理
开始这一节之前，读者可以回顾一下文章 [Binder之Service Manager](http://www.muzileecoding.com/framework/binder-servicemanager.html) 的 5.3 和 5.4 小节。

__【注意】__ 下面的函数和前面的分析很类似，但是都是运行在 SM 进程中的，千万不要混淆了！

首先这里是因为条件符合才返回的，因此会走到`binder_thread_read`函数的`while`循环中去:

```C++
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
            
        // 根据不同的 work 类型进行处理
        switch (w->type) {
	        case BINDER_WORK_TRANSACTION: {
	        t = container_of(w, struct binder_transaction, work);
	        }break;
        }
        ......
}
```
这里从 thread/proc 的`todo`队列中获取 binder_work 来进行处理，以下是根据不同的 work 类型进行处理，前面插入 binder_work 的代码是这样的：

```C++
// t->work 是 binder_work 结构体，该结构体在字典中可查询
t->work.type = BINDER_WORK_TRANSACTION;
// 添加 t 到目标进程的todo list中去
list_add_tail(&t->work.entry, target_list);
```
因此类型是"BINDER_WORK_TRANSACTION"，所以接着调用`container_of`方法获取 t，即 binder_transaction 对象。
>`container_of`的作用：根据结构体的成员变量获取所在结构体的首地址。
>
>代码里面藏着很多这样的函数/宏 —— 强大但是很不好阅读。

获取到 binder_transaction 对象之后，剩下的就是解析了:

```C++
// tr 是 binder_transaction_data 结构体
struct binder_transaction_data tr;
struct binder_work *w;
struct binder_transaction *t = NULL;

if (!t)// 通过
	continue;

 // 赋值：t->buffer->target_node = target_node; 
 // 因此条件成立，且target_node指向的是 binder_context_mgr_node
if (t->buffer->target_node) {
	struct binder_node *target_node = t->buffer->target_node;
	// tr 是 binder_transaction_data 类型，还记得target这两个变量的值么？
	tr.target.ptr = target_node->ptr;
	tr.cookie =  target_node->cookie;
	t->saved_priority = task_nice(current);
	if (t->priority < target_node->min_priority && !(t->flags & TF_ONE_WAY))
		binder_set_nice(t->priority);
	else if (!(t->flags & TF_ONE_WAY) || t->saved_priority > target_node->min_priority)
		binder_set_nice(target_node->min_priority);
	
	//注意这个命令的赋值
	cmd = BR_TRANSACTION;
} else {
	tr.target.ptr = NULL;
	tr.cookie = NULL;
	cmd = BR_REPLY;
}

tr.code = t->code;
tr.flags = t->flags;
tr.sender_euid = t->sender_euid;
if (t->from) {
	struct task_struct *sender = t->from->proc->tsk;
	tr.sender_pid = task_tgid_nr_ns(sender, current->nsproxy->pid_ns);
} else {
	tr.sender_pid = 0;
}
tr.data_size = t->buffer->data_size;
tr.offsets_size = t->buffer->offsets_size;
// 根据老罗的文章：这里正是 binder 机制的精髓，即如何做到复制一次的。
tr.data.ptr.buffer = (void *)t->buffer->data +  proc->user_buffer_offset;
tr.data.ptr.offsets = tr.data.ptr.buffer + ALIGN(t->buffer->data_size, sizeof(void *));

// @read数据@ 下面就是把cmd 和 tr 结构体 copy 到用户空间，cmd = BR_TRANSACTION;
if (put_user(cmd, (uint32_t __user *)ptr))
	return -EFAULT;
ptr += sizeof(uint32_t);

if (copy_to_user(ptr, &tr, sizeof(tr)))
	return -EFAULT;
ptr += sizeof(tr);

trace_binder_transaction_received(t);
binder_stat_br(proc, thread, cmd);
// 处理完毕，删除节点
list_del(&t->work.entry);
t->buffer->allow_user_free = 1;
// 添加服务的时候是需要回复的，这里判断成立
if (cmd == BR_TRANSACTION && !(t->flags & TF_ONE_WAY)) {
	t->to_parent = thread->transaction_stack;
	t->to_thread = thread;
	thread->transaction_stack = t; // 放到栈顶 
} else {
	t->buffer->transaction = NULL;
	kfree(t);
	binder_stats_deleted(BINDER_STAT_TRANSACTION);
}
// 跳出循环
break;
```
相关解析都在注释中，之前传过来的数据都被拷贝到用户空间中去了，之后就跳出循环，回到`binder_ioctl()`函数：

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
        	// 数据都被拷贝到用户空间了
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
        
        //再次拷贝数据到用户空间
        if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {
            ret = -EFAULT;
            goto err;
        }
        break;
    }
    ret = 0;
err:
	......
}
```
好了，所有的数据顺利传输到了用户空间，在用户空间等待它的是`binder_loop()`函数：

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
在`ioctl()`返回之后，调用的就是`binder_parse()`函数，这个函数可以处理很多的命令，我们就看相关部分:

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
这里读出来的命令就是`BR_TRANSACTION`，然后将数据部分（一个 binder_transaction_data 结构体，读者可以搜"@read数据@"定位）解析成`binder_txn`结构体(对比结构体字段就可以知道转换确实可行)。接着是声明了两个 binder_io 结构体，在字典中可查。在这里还有两个方法，也一并列举一下：

```C++
void bio_init_from_txn(struct binder_io *bio, struct binder_txn *txn)
{
    bio->data = bio->data0 = txn->data;
    bio->offs = bio->offs0 = txn->offs;
    bio->data_avail = txn->data_size;
    bio->offs_avail = txn->offs_size / 4;
    bio->flags = BIO_F_SHARED;
}

void bio_init(struct binder_io *bio, void *data,
              uint32_t maxdata, uint32_t maxoffs)
{
    uint32_t n = maxoffs * sizeof(uint32_t);

    if (n > maxdata) {
        bio->flags = BIO_F_OVERFLOW;
        bio->data_avail = 0;
        bio->offs_avail = 0;
        return;
    }

    bio->data = bio->data0 = data + n;
    bio->offs = bio->offs0 = data;
    bio->data_avail = maxdata - n;
    bio->offs_avail = maxoffs;
    bio->flags = 0;
}
```
这个结构体在这里解释的意义不大，下一篇文章[Binder之Service查询](http://www.muzileecoding.com/framework/binder-service-lookup.html)中有较详细的解释。

最后调用func来处理命令，func是从 SM 的`main`函数中传递过来的，它真实的指向是`svcmgr_handler`函数：

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
		......
    case SVC_MGR_ADD_SERVICE:
        s = bio_get_string16(msg, &len); // "media.player"
        ptr = bio_get_ref(msg);
        if (do_add_service(bs, s, len, ptr, txn->sender_euid))
            return -1;
        break;
	
		......
    }

    bio_put_uint32(reply, 0);
    return 0;
}
```
这里的 code 最初是什么呢？读者往上翻翻就知道是`ADD_SERVICE_TRANSACTION`，它是定义在 "IServiceManager.h" 中的一个枚举值：

```C++
enum {
	GET_SERVICE_TRANSACTION = IBinder::FIRST_CALL_TRANSACTION,
	CHECK_SERVICE_TRANSACTION,
	ADD_SERVICE_TRANSACTION,
	LIST_SERVICES_TRANSACTION,
};
```
`IBinder::FIRST_CALL_TRANSACTION`的定义如下：`FIRST_CALL_TRANSACTION  = 0x00000001`，即1，那么ADD_SERVICE_TRANSACTION就是 3。而这里 code 的几个可能值类型也是枚举值：

```C++
enum {
    SVC_MGR_GET_SERVICE = 1,
    SVC_MGR_CHECK_SERVICE,
    SVC_MGR_ADD_SERVICE,
    SVC_MGR_LIST_SERVICES,
};
```
所以，`SVC_MGR_ADD_SERVICE`的值也是 3，即执行该 case。从代码看应该正好有四句数据读取代码：

```C++
strict_policy = bio_get_uint32(msg); // 读取 StrictModePolicy，一个int值
s = bio_get_string16(msg, &len); // “android.os.IServiceManager”
s = bio_get_string16(msg, &len);
ptr = bio_get_ref(msg);
```
还记得服务注册初始数据图么？它正好也是四个部分，与这里是一一对应，这部分数据从一开始的分析就出现了，一直到现在才开始解析，调用链极长。第四个变量还需要看一下方法`bio_get_ref`:

```C++
void *bio_get_ref(struct binder_io *bio)
{
    struct binder_object *obj;

    obj = _bio_get_obj(bio);
    if (!obj)
        return 0;

    if (obj->type == BINDER_TYPE_HANDLE)
        return obj->pointer;

    return 0;
}

static struct binder_object *_bio_get_obj(struct binder_io *bio)
{
    unsigned n;
    unsigned off = bio->data - bio->data0;

    for (n = 0; n < bio->offs_avail; n++) {
        if (bio->offs[n] == off)
        	//获取 binder_object 对象
            return bio_get(bio, sizeof(struct binder_object));
    }

    bio->data_avail = 0;
    bio->flags |= BIO_F_OVERFLOW;
    return 0;
}
```
这里还是得看服务注册初始数据图：
<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/service注册transact数据.png" width="600" alt="service注册transact数据"/></div>

实际上我们在这个位置存储的对象并不是 binder_object 结构体，而是 flat_binder_object 结构体，但是字段上看，的确是可以强转的。另外不知道读者还是否记得，binder 驱动在传递 binder 节点的时候，已经把 Type 改成了 BINDER_TYPE_HANDLE 了，因此这里的 ptr 实际就是 SM 为 MediaPlayerService 分配的 handle 值。

接下去就到了正主出现的时刻：`do_add_service`：

```C++
//参数解释：s 就是 "media.player"
int do_add_service(struct binder_state *bs,
                   uint16_t *s, unsigned len,
                   void *ptr, unsigned uid)
{
	// 该结构体在字典中可查
    struct svcinfo *si;
	
    if (!ptr || (len == 0) || (len > 127))
        return -1;
        
    if (!svc_can_register(uid, s)) {
        LOGE("add_service('%s',%p) uid=%d - PERMISSION DENIED\n",
             str8(s), ptr, uid);
        return -1;
    }

    si = find_svc(s, len);
    if (si) {
        if (si->ptr) {
            LOGE("add_service('%s',%p) uid=%d - ALREADY REGISTERED, OVERRIDE\n",
                 str8(s), ptr, uid);
            svcinfo_death(bs, si);
        }
        si->ptr = ptr;
    } else {
        si = malloc(sizeof(*si) + (len + 1) * sizeof(uint16_t));
        if (!si) {
            LOGE("add_service('%s',%p) uid=%d - OUT OF MEMORY\n",
                 str8(s), ptr, uid);
            return -1;
        }
        si->ptr = ptr;
        si->len = len;
        memcpy(si->name, s, (len + 1) * sizeof(uint16_t));
        si->name[len] = '\0';
        si->death.func = svcinfo_death;
        si->death.ptr = si;
        si->next = svclist;
        svclist = si;
    }

    binder_acquire(bs, ptr);
    binder_link_to_death(bs, ptr, &si->death);
    return 0;
}
```
这个函数一进来首先是参数判断，从这里可以看出，实际上 Service 名字的长度是有限制的，转成字符数组必须在长度 127 以内；第二步是权限检测：

```C++
static struct {
    unsigned uid;
    const char *name;
} allowed[] = {
#ifdef LVMX
    { AID_MEDIA, "com.lifevibes.mx.ipc" },
#endif
    { AID_MEDIA, "media.audio_flinger" },
    { AID_MEDIA, "media.player" },
    { AID_MEDIA, "media.camera" },
    { AID_MEDIA, "media.audio_policy" },
    { AID_DRM,   "drm.drmManager" },
    { AID_NFC,   "nfc" },
    { AID_RADIO, "radio.phone" },
    { AID_RADIO, "radio.sms" },
    { AID_RADIO, "radio.phonesubinfo" },
    { AID_RADIO, "radio.simphonebook" },
/* TODO: remove after phone services are updated: */
    { AID_RADIO, "phone" },
    { AID_RADIO, "sip" },
    { AID_RADIO, "isms" },
    { AID_RADIO, "iphonesubinfo" },
    { AID_RADIO, "simphonebook" },
};

int svc_can_register(unsigned uid, uint16_t *name)
{
    unsigned n;
    
    if ((uid == 0) || (uid == AID_SYSTEM))
        return 1;

    for (n = 0; n < sizeof(allowed) / sizeof(allowed[0]); n++)
        if ((uid == allowed[n].uid) && str16eq(name, allowed[n].name))
            return 1;

    return 0;
}
```
只有 root 进程或者 system server 进程或者具有特殊名字的服务才能注册的。再接着就是查看是否有同名服务注册：

```C++
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
这里有必要解释一下 svclist 这个变量了，它是一个链表，元素就是 svcinfo，所有注册过来的服务都以 svcinfo 记录下来添加在 svclist 中，因此函数`find_svc()`实际在做事情是遍历该链表，找出是否有同名的服务已经注册过了。如果有，则更新服务，用新的 ptr 替换旧有的 ptr，否则分配新的 svcinfo 结构体，记录服务信息，并挂载到 svclist 链表上，执行完毕后返回 0。

好了，到这里，__服务终于注册到 SM 了__。实际上还没有完结，因为 MediaPlayerService 还在等待回复啊！

## 注册答复
注册答复的起始点我们还得回到`binder_parse`方法：
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
在调用`svcmgr_handler`函数处理请求之后，会接着调用`binder_send_reply`发送答复：

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
    } else { // 走这里
        data.txn.flags = 0;
        data.txn.data_size = reply->data - reply->data0;
        data.txn.offs_size = ((char*) reply->offs) - ((char*) reply->offs0);
        data.txn.data = reply->data0;
        data.txn.offs = reply->offs0;
    }
    binder_write(bs, &data, sizeof(data));
}
```
这里包装了很多数据，通过`binder_write`又写回到 驱动中：

```C++
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
注意，只有write部分是有数据的，接着又回到了`binder_thread_write`方法中，其中有两个命令，一个是`BC_FREE_BUFFER`，另外一个是`BC_REPLY`，前者顾名思义就是释放缓冲区，我们就不去看了，后者比较重要：

```C++
case BC_TRANSACTION:
case BC_REPLY: {
	struct binder_transaction_data tr;
	if (copy_from_user(&tr, ptr, sizeof(tr)))
		return -EFAULT;
		ptr += sizeof(tr);
		binder_transaction(proc, thread, &tr, cmd == BC_REPLY);
		break;
}
```
这个方法已经来过一次了，只不过上次是`BC_TRANSACTION`命令。自然而然又要进入`binder_transaction`了：

```C++
// 参数解释：reply 是 true
static void binder_transaction(struct binder_proc *proc,
                   struct binder_thread *thread,
                   struct binder_transaction_data *tr, int reply)
{
    struct binder_transaction *t;
    struct binder_work *tcomplete;
    size_t *offp, *off_end;
    struct binder_proc *target_proc;
    struct binder_thread *target_thread = NULL;
    struct binder_node *target_node = NULL;
    struct list_head *target_list;
    wait_queue_head_t *target_wait;
    struct binder_transaction *in_reply_to = NULL;
    struct binder_transaction_log_entry *e;
    uint32_t return_error;
    e = binder_transaction_log_add(&binder_transaction_log);
    e->call_type = reply ? 2 : !!(tr->flags & TF_ONE_WAY);
    e->from_proc = proc->pid;
    e->from_thread = thread->pid;
    e->target_handle = tr->target.handle;
    e->data_size = tr->data_size;
    e->offsets_size = tr->offsets_size;
    if (reply) { // true
        in_reply_to = thread->transaction_stack; // 之前放了一个进去，不为null
        binder_set_nice(in_reply_to->saved_priority);
        if (in_reply_to->to_thread != thread) {
            .....
            goto err_bad_call_stack;
        }
        thread->transaction_stack = in_reply_to->to_parent;
        target_thread = in_reply_to->from; // 找到发送请求的线程
        if (target_thread == NULL) { 
            return_error = BR_DEAD_REPLY;
            goto err_dead_binder;
        }
        // 校验，保证是同一个 transaction
        if (target_thread->transaction_stack != in_reply_to) {
            goto err_dead_binder;
        }
        // 本次回复的目标进程
        target_proc = target_thread->proc;
    } else {
        .....
    }
    if (target_thread) {
        e->to_thread = target_thread->pid;
        target_list = &target_thread->todo;
        target_wait = &target_thread->wait;
    } else {
        target_list = &target_proc->todo;
        target_wait = &target_proc->wait;
    }
    e->to_proc = target_proc->pid;
    t = kzalloc(sizeof(*t), GFP_KERNEL);
    binder_stats_created(BINDER_STAT_TRANSACTION);
    tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);
    binder_stats_created(BINDER_STAT_TRANSACTION_COMPLETE);
    t->debug_id = ++binder_last_id;
    e->debug_id = t->debug_id;
    if (!reply && !(tr->flags & TF_ONE_WAY))
        t->from = thread;
    else // 走这里
        t->from = NULL; 
    t->sender_euid = proc->tsk->cred->euid;
    t->to_proc = target_proc;
    t->to_thread = target_thread;
    t->code = tr->code;
    t->flags = tr->flags;
    t->priority = task_nice(current);
    trace_binder_transaction(reply, t, target_node);
    t->buffer = binder_alloc_buf(target_proc, tr->data_size,
        tr->offsets_size, !reply && (t->flags & TF_ONE_WAY));
    t->buffer->allow_user_free = 0;
    t->buffer->debug_id = t->debug_id;
    t->buffer->transaction = t;
    t->buffer->target_node = target_node;
    trace_binder_transaction_alloc_buf(t->buffer);
    if (target_node)
        binder_inc_node(target_node, 1, 0, NULL);
    offp = (size_t *)(t->buffer->data + ALIGN(tr->data_size, sizeof(void *)));
    if (copy_from_user(t->buffer->data, tr->data.ptr.buffer, tr->data_size)) {
        return_error = BR_FAILED_REPLY;
        goto err_copy_data_failed;
    }
    if (copy_from_user(offp, tr->data.ptr.offsets, tr->offsets_size)) {
        return_error = BR_FAILED_REPLY;
        goto err_copy_data_failed;
    }
    if (!IS_ALIGNED(tr->offsets_size, sizeof(size_t))) {
        return_error = BR_FAILED_REPLY;
        goto err_bad_offset;
    }

    ......
    
    if (reply) {
        // 执行pop，删除该 binder_transaction
        binder_pop_transaction(target_thread, in_reply_to);
    } else if (!(t->flags & TF_ONE_WAY)) {
        ......
    } else {
        ......
    }
    t->work.type = BINDER_WORK_TRANSACTION;
    // 又是挂载任务
    list_add_tail(&t->work.entry, target_list);
    tcomplete->type = BINDER_WORK_TRANSACTION_COMPLETE;
    list_add_tail(&tcomplete->entry, &thread->todo);
    // 唤醒线程
    if (target_wait)
        wake_up_interruptible(target_wait);
    return;
    ......
}
```
注释已经添加，和之前进来差不多，只不过角色互换了：SM 往 MediaPlayerService 的 申请服务注册线程上面挂载了一个任务。最终 SM 的线程唤醒了在`target_wait`上等待的线程，即我们在 __2.3.4 节__ 描述的 MediaPlayerService 的申请服务注册线程。然后 SM 欢快的回去进行收尾工作。

MediaPlayerService 的申请服务注册线程被唤醒后会做什么，这里不做分析了，读者可以自行分析，下篇文章会涉及。

## ThreadPool
在服务初始化完成之后，"main_mediaserver" 会执行下面两行代码:

```C++
ProcessState::self()->startThreadPool();
IPCThreadState::self()->joinThreadPool();
```
下面就来解析一下这两行代码：

### `ProcessState::self()->startThreadPool()`
该方法的实现如下：

```C++
void ProcessState::startThreadPool()
{
    AutoMutex _l(mLock);
    if (!mThreadPoolStarted) {
        mThreadPoolStarted = true;
        spawnPooledThread(true);
    }
}
``
设置`mThreadPoolStarted`属性之后，调用方法`spawnPooledThread`：

​```C++
// 参数解释：isMain为true
void ProcessState::spawnPooledThread(bool isMain)
{
    if (mThreadPoolStarted) {
        int32_t s = android_atomic_add(1, &mThreadPoolSeq);
        char buf[32];
        sprintf(buf, "Binder Thread #%d", s);
        LOGV("Spawning new pooled thread, name=%s\n", buf);
        sp<Thread> t = new PoolThread(isMain);
        t->run(buf);
    }
}
```
主要工作是新建了 PoolThread 实例，并调用`run()`方法，该类定义如下：

```C++
class PoolThread : public Thread
{
public:
    PoolThread(bool isMain)
        : mIsMain(isMain)
    {
    }
    
protected:
    virtual bool threadLoop()
    {
        IPCThreadState::self()->joinThreadPool(mIsMain);
        return false;
    }
    
    const bool mIsMain;
};
```
`run()`来源于父类，会调用虚方法`threadLoop()`，而在`threadLoop()`中调用的却是`IPCThreadState::self()->joinThreadPool(mIsMain)`，好了，万剑归宗，直接看这个方法吧。

### 7.2 `IPCThreadState::self()->joinThreadPool()`
该方法的在 header 文件中的定义如下:

```C++
void joinThreadPool(bool isMain = true);
```
默认参数为 true。
>老罗的文章在这里有个小错误，默认值确实是 true，我看了1.6，2.2.3，4.0.4几个版本都是true。

这个方法的实现如下：

```C++
void IPCThreadState::joinThreadPool(bool isMain)
{
    
    // 进入 Looper 状态
    mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);
    
    // This thread may have been spawned by a thread that was in the background
    // scheduling group, so first we will make sure it is in the default/foreground
    // one to avoid performing an initial transaction in the background.
    // 调度线程的优先级
    androidSetThreadSchedulingGroup(mMyThreadId, ANDROID_TGROUP_DEFAULT);
        
    status_t result;
    do {
        int32_t cmd;
        
        ......

        // 读取命令
        result = talkWithDriver();
        if (result >= NO_ERROR) {
            size_t IN = mIn.dataAvail();
            if (IN < sizeof(int32_t)) continue;
            cmd = mIn.readInt32();

            // 调用该函数执行命令
            result = executeCommand(cmd);
        }
        
        // 调度线程的优先级
        androidSetThreadSchedulingGroup(mMyThreadId, ANDROID_TGROUP_DEFAULT);

        // Let this thread exit the thread pool if it is no longer
        // needed and it is not the main process thread.
        if(result == TIMED_OUT && !isMain) {
            break;
        }
    } while (result != -ECONNREFUSED && result != -EBADF);

    // 退出 Looper 状态
    mOut.writeInt32(BC_EXIT_LOOPER);
    // 不再接受数据
    talkWithDriver(false);
}
```
所以这里是起了两个线程，不停的从驱动中读取数据，并交给`executeCommand()`执行命令。在处理各种命令的时候，有一段如下的代码：

```C++
if (tr.target.ptr) {
	sp<BBinder> b((BBinder*)tr.cookie);
	const status_t error = b->transact(tr.code, buffer, &reply, tr.flags);
	if (error < NO_ERROR) reply.setError(error);
}
```
这段代码可谓精华了！还记得`cookie`记录的是什么么？服务实例在用户空间的引用地址！所以这里是根据这个地址又拿到了该服务的实例，然后还记得这张图么？
<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/Binder-Lib层架构.png" alt="Binder Lib层架构"/></div>

BBinder的`transact()`方法实现如下：

```C++
status_t BBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    data.setDataPosition(0);

    status_t err = NO_ERROR;
    switch (code) {
        case PING_TRANSACTION:
            reply->writeInt32(pingBinder());
            break;
        default:
            err = onTransact(code, data, reply, flags);
            break;
    }

    if (reply != NULL) {
        reply->setDataPosition(0);
    }

    return err;
}
```
这里调用的又是`onTransact()`，而这个方法又是在BBinder的子类，比如`BnMediaPlayerService`中实现的 —— 即可以根据具体的服务 API 进行不同的解析，并完成相关功能。

## 总结
代码分析完毕，读者可能觉得整个过程无比繁杂，有些方法又臭又长，还不止调用一次。但实际上这块机制并不复杂，理解起来困难可能是因为：

1. C++ 语言编写，对于 Java 开发者来说语言不熟悉，C++ 本身的语言特性也导致阅读起来比较困难；
2. 涉及驱动层，驱动知识造成障碍；
3. 不能调试，如果是一个开源框架，我们是可以写一个 Demo 去调试某个点的，这样就不用费这么多精力把脑子当做计算机使用了；
4. 对 Libraries 和 Binder 驱动层的交互协议不清楚，这样在数据上就比较难搞清楚结构，方法调用链又长，调用几层之后对于参数的内容就模糊了；
5. 代码问题，这两层的代码写的并不好，注释也少，理解起来非常费力；

>和 framework 层的代码比起来，这里的方法调用链其实并不算长。

这里类比网络请求做一些文字描述，希望有助于读者搞清楚 Binder 的机理。大体过程上有以下几个关键部分：

1. 使用`binder_write_read`结构体在 Libraries 层和驱动层之间做数据传输介质，类比网络请求，该结构体就相当于 Request 和 Response 的合体；
2. 有传输介质之后，传输使用的手段是`ioctl`、`open`等函数，类比网络请求，就像`openConnection()`等函数；
3. 如果我们要发送数据，则在`binder_write_read`结构体的 write 部分写入数据，驱动层会 copy 数据到内核空间，并读取数据进行处理，当然，这个时候也会携带一些命令，比如`BINDER_WRITE_READ`，类比网络请求，这些命令类似于`GET`，`POST`等；
4. 如果`binder_write_read`的 read 部分要求读取数据，那么驱动层会查看有没有事务要处理，有没有数据要传输给上层（具体传输什么数据，由场景确定），如果有就写入到 read 部分并 copy 到 Libraries 层，函数返回，Libraries 层读取数据进行处理，同样的这个时候数据中也会有命令，类比网络请求，这些命令就像"200"，"404"，"500"等 Http Status Code；如果没有数据，则会阻塞在自己的事务处理队列上，
   5. 驱动层拿到数据之后，会做一些处理，比如生成`binder_node`节点，在目标进程中生成`binder_ref`节点，并操作红黑树、插入`binder_transaction`等等，这些过程看上去极为复杂，原因是我们一开始并不了解 binder 具体要做什么。实际上 binder 记录这些数据只为两件事：binder_transaction 能够正确发送到目标进程和目标进程处理完事务之后能够正确找到回复进程。这是 IPC 的主要任务，其余的数据都是为了这两个目标准备的。举个例子，比如在注册的时候在目标进程的 transaction_stack 中插入 binder_transaction 节点，就是为了处理完事务之后可以找到回复进程；

以上前四点和一个网络请求并无二致，只是数据规则变了一下，最后一点则是机制本身导致的特性。另外这里有一个核心点一定要整理一下：

__每一个服务都会对应在驱动层生成一个`binder_node`节点，这个节点是记录在本服务进程在驱动层的映射`binder_proc`结构体中的，binder驱动会维护很多的`binder_proc`结构体，在除本进程外的`binder_proc`结构体中，引用本进程的服务，一律使用的是`binder_ref`结构体！至于如何通过`binder_ref`结构体获取到对应的服务，正是下一篇文章所要探索的内容。__

以上，如果读者想要了解原理的话，直接读这部分即可。如果读者想了解一些较细致的原理的话，墙裂建议阅读推荐文章①和文章③两系列博客，还是那个评价：整理有序，图文并茂。实乃对付此等巨兽的神兵之作。

至此，服务注册分析完成。