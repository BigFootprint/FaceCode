---
title: OC Runtime（二）
date: 2016-12-10 20:44:27
tags: [Runtime]
categories: iOS
---

在前一篇文章[OC Runtime（一）](http://timebridge.space/2016/12/10/OC-Runtime/)（以下简称"前文"）里面，通过对[《ObjC Runtime Guide》](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008048-CH1-SW1)文档的阅读，基本了解了 Runtime 的概念和能力。本文在此基础上进一步深入探索，为什么 Runtime 具备这样的能力，OC 又是基于怎样的结构实现当前的特性的。

> 在 Apple 的[Source Browser](https://opensource.apple.com/tarballs/objc4/)网站上可以下载到 Runtime 的源代码，代码下载下来解压后可以导入到 XCode 中查看。

## 入口

前文提到，开发者在 OC 中的消息发送最终会被转化为 `objc_msgSend(receiver, selector, arg1, arg2, ...)`这样的调用格式，`objc_msgSend`方法定义在 message.h 中：

```objective-c
id objc_msgSend(id self, SEL op, ...)
```

第一个和第二个参数在前文中有相应解释，这里它们的类型分别是 id 和 SEL，并且函数的返回值类型也是 id。那么这两个类型代表什么呢？我们就从这里入手。<!-- More -->

## SEL 和 id

在项目中简单搜索，很快就能发现两者的定义：

```objective-c
typedef struct objc_selector *SEL;
typedef struct objc_object *id;
```

SEL 是一个指向 objc_selector 的指针，id 是一个指向 objc_object 的指针。

### objc_selector

关于 objc_selector 这个结构体的定义，runtime 库中并没有给出，但是可以参考以下两份资料：

* [objc.h](https://sourceware.org/svn/gcc/tags/stack-last-merge/libobjc/objc/objc.h)
* [Professional Swift](https://books.google.com.hk/books?id=onyzCAAAQBAJ&pg=PA237&lpg=PA237&dq=the+definition+of+objc_selector&source=bl&ots=p2rIQX9Wjb&sig=MxfZ9CPPzWSlAKqyz1xMZJt_10k&hl=zh-CN&sa=X&ved=0ahUKEwj7vbD-4enQAhXFLhoKHa__Cz0Q6AEIVDAI#v=onepage&q=the%20definition%20of%20objc_selector&f=false)

其中第二份资料提到：

> However, on OS X and iOS, a struct objc_selector is simply a C string internally.

结合我们对 SEL 的使用看，可以认为 SEL 等于一个字符串。比如下面的例子：

```objective-c
Method method = class_getInstanceMethod([MsgTest class], @selector(realMethod));
NSLog(@"%@", NSStringFromSelector(method_getName(method)));
```

就会输出：realMethod。

读者可以去看看`method_getName`这个方法的返回值类型，就是 SEL，也就是说 SEL 是用来表示方法名的。

### objc_object

这个结构体定义在 object-private.h 中，只有一个属性：

```c++
struct objc_object {
private:
    isa_t isa;

public:
    //各种方法
}
```

这个属性是 isa_t 类型，它是一个联合体：

```c++
union isa_t 
{
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

    Class cls;
    uintptr_t bits;
  	//下面是三个结构体
}
```

其中核心部分是一个 Class 属性，Class 是什么呢？定义如下：

```objective-c
typedef struct objc_class *Class;
```

是一个指向 objc_class 的指针，也就是说，__每一个对象内部都存储着指向自己所属类型的指针__。

objc_class 的定义则如下：

```c++
struct objc_class : objc_object {
    Class superclass;
    const char *name;
    uint32_t version;
    uint32_t info;
    uint32_t instance_size;
    struct old_ivar_list *ivars;
    struct old_method_list **methodLists;
    Cache cache;
    struct old_protocol_list *protocols;
  	// CLS_EXT only
    const uint8_t *ivar_layout;
    struct old_class_ext *ext;
    //各种方法
}
```

到这里就很有意思了，虽然这些代码注释不多，但根据名字，依然能够看出一些端倪。

* 首先这个结构体是继承 objc_object 的，这表明类也是一个对象，同时也会有一个 isa 属性；
* 其次这个结构体包含一个 superclass 的 Class 属性，这应该是指向它的父类；
* 接着，这个结构体包含三个结构体，从名字看，应该是依次包含所有变量、方法和实现的协议；
* 还有一个 Cache 变量，这个应该是前文提到的用于缓存被调用方法的；
* 最后，有一个 old_class_ext 扩展结构体属性；

#### 变量、方法、协议和属性

变量列表的结构定义如下：

```c++
struct old_ivar {
    char *ivar_name;
    char *ivar_type;
    int ivar_offset;
#ifdef __LP64__
    int space;
#endif
};

struct old_ivar_list {
    int ivar_count;
#ifdef __LP64__
    int space;
#endif
    /* variable length structure */
    struct old_ivar ivar_list[1];
};
```

看上去很简单，这个列表里面存储着的元素是 old_var，而这个结构体中有两个很重要的成员：变量名和变量类型。方法列表大致相同：

```c++
struct old_method {
    SEL method_name;
    char *method_types;
    IMP method_imp;
};

struct old_method_list {
    void *obsolete;

    int method_count;
#ifdef __LP64__
    int space;
#endif
    /* variable length structure */
    struct old_method method_list[1];
};
```

但是方法列表的元素结构体有些特殊，首先方法名是 SEL 类型的，这个前面已经有分析。method_types 是什么？IMP 是又什么东西呢？

如果读者阅读了前文提到的[Type Encodings](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100-SW1)小节，其实不是很难理解 method_types，我们可以使用如下的代码打印出它的案例，还是以前文的 MsgTest 类为例：

```objective-c
Method method = class_getInstanceMethod([MsgTest class], @selector(realMethod));
NSLog(@"%s", method_getTypeEncoding(method));
```

这段代码会输出：v16@0:8。

> v 表示 void，即返回值为 void，16 表示 整个方法参数占位的总长度，@0 表示在参数区域偏移为 0 的位置有一个 OC 对象，还记得前面说到的方法在被编译之后，第一个参数是 id 类型，即 self 么？:8 表示在参数列表偏移为 8 字节的地方有一个 SEL 对象，也就是前面分析的第二个参数 _cmd。
>
> 这里由于内存对齐的原因，会导致很多类型都站 4 个字节或者 8 个字节，读者可以自己实验。

IMP 的定义如下：

```c++
/// A pointer to the function of a method implementation. 
#if !OBJC_OLD_DISPATCH_PROTOTYPES
typedef void (*IMP)(void /* id, SEL, ... */ ); 
#else
typedef id (*IMP)(id, SEL, ...); 
#endif
```

注释说的很明白：指向方法实现的函数指针。其实在前文中提到 class_addMethod 的时候，已经遇到并使用过它了。

协议部分的定义如下：

```c++
struct old_protocol {
    Class isa;
    const char *protocol_name;
    struct old_protocol_list *protocol_list;
    struct objc_method_description_list *instance_methods;
    struct objc_method_description_list *class_methods;
};

struct old_protocol_list {
    struct old_protocol_list *next;
    long count;
    struct old_protocol *list[1];
};
```

协议元素 old_protocol 包含两个结构体，分别描述实例方法和类方法。最终都指向以下元素的一个集合：

```c++
/// Defines a method
struct objc_method_description {
	SEL name;               /**< The name of the method */
	char *types;            /**< The types of the method arguments */
};
```

这里倒是很清楚的指出了__SEL 代表的就是方法名__，而 types 指向的是参数类型。只不过协议是没有方法实现的，因此这里没有 IMP 指针。其余的部分就不过多探究了。

#### 属性

乍一看，objc_class 结构体中并没有属性相关的内容，那是因为属性藏到了 old_class_ext 结构体中：

```c++
struct old_class_ext {
    uint32_t size;
    const uint8_t *weak_ivar_layout;
    struct old_property_list **propertyLists;
};
```

old_property_list 结构体的内容如下：

```c++
struct old_property {
    const char *name;
    const char *attributes;
};

struct old_property_list {
    uint32_t entsize;
    uint32_t count;
    struct old_property first;
};
```

到这边会看到一些熟悉的东西，这个列表里的元素是 old_property 结构体，还记得前文中打印属性的 name 和attributes 的例子么？这就是那两个值存储的地方。

#### 缓存

Cache 的定义如下：

```c++
typedef struct objc_cache *Cache 
```

它是一个指向 objc_cache 的指针：

```c++
struct objc_cache {
    unsigned int mask /* total = mask + 1 */                 OBJC2_UNAVAILABLE;
    unsigned int occupied                                    OBJC2_UNAVAILABLE;
    Method buckets[1]                                        OBJC2_UNAVAILABLE;
};
```

Method 又是啥？其实前面的例子中已经用到，它的定义如下：

```C++
typedef struct old_method *Method;
```

就是一个指向 old_method 结构体的指针，这个结构体前面已经展示过了，因此 Cache 可以简单的认为就是一个存储着方法列表的结构体。这和我们之前了解到的和猜测的也相符合。

#### 类和元类

虽然前面对 objc_class 结构体的展示省略了所有的方法，但是其中有一个方法还是比较重要的：

```c++
// NOT identical to this->ISA() when this is a metaclass
Class getMeta() {
	if (isMetaClass()) return (Class)this;
	else return this->ISA();
}
```

这里涉及到一个元类的概念，当调用`getMeta()`方法的时候，如果当前类就是一个元类，返回自身就好，否则返回的是：

```c++
inline Class objc_object::ISA() 
{
    assert(!isTaggedPointer()); 
    return isa.cls;
}
```

这是个内联函数，最终值为 isa.cls —— 指向的是一个 Class 对象。那么究竟什么是 Meta Class 呢？读者可以参考这篇文章：[What is a meta-class in Objective-C?](https://www.cocoawithlove.com/2010/01/what-is-meta-class-in-objective-c.html) 它的主要结论如下：

1. 任何包含指向 Class 结构体指针的结构体都可以被认为是 objc_object，这也是为什么 OC 中可以向对象发送消息的原因：因为 Class 中包含了方法列表，这个在前文以及前面的分析中都已经提到；
2. Class 也是一个对象，根据前面的分析，objc_class 是继承 objc_object 的，也就是说，Class 也可以接收消息，比如`[NSObject alloc]`；
3. 既然 Class 也是一个对象，那就必然有一个指针指针一个类型，以符合结论 1 中定义的 objc_object 的结构体特性，Class 指向的这个类型也必须包含一个 Method 列表以表明我们可以在 Class 上执行哪些方法；
4. Meta-Class，即元类，就是为这个需求存在的：当向一个 Class 对象发送消息的时候，就会在元类中寻找消息的处理者，也就是说，元类存储着类方法，每一个类都必须有自己独特的元类，因为每个类的类方法都可能不一样；
5. 既然元类也有消息处理能力，即表明元类也和 Class 一样，也是一个对象，那它也必须有一个 Class。所有的元类都以继承树中的最基类的元类作为它们的 Class，也就是 NSObject 的所有子类的元类都以 NSObject 的元类作为自己的 Class。
6. Class 通过 superclass 指向指向自己的父类，meta-class 则通过自己的 superclass 指向父类的元类。

总结下来就是下面这幅图（来源不明）：

<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/类与元类.jpg" height="360" alt="Android调优工具汇总"/></div>

__最基类（NSObject）是所有子类的基类，最基类的元类是最基类所有子类的元类的最基类，最基类的元类是所有子类以及自身元类的元类。__

## 总结

至此，Runtime 中比较重要的数据结构分析完毕，结合前文，我们也基本知道了 OC 下面是怎样的一套数据结构在支撑它的特性。

Runtime 库中有部分实现代码在其中，在了解了基本的数据结构之后，接着就是探索 OC 特性与这些数据结构之间的关系，比如 Category 的实现原理、Method Swizzling等。