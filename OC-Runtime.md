---
title: OC Runtime（一）
date: 2016-12-10 10:59:43
tags: [Runtime]
categories: iOS
---

> 参考资料：[《ObjC Runtime Guide》](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008048-CH1-SW1)。

## 简介

### 背景

简单来说，__Runtime 是 OC 语言的运行环境__。

OC 会尽量将决议延迟到运行时做出而非在编译和连接期间，比如方法调用具体是执行哪段代码。这意味着 OC 不仅需要一个特定的编译器，而且需要一个运行系统来执行编译后的代码从而实现决议延迟，这有点像 JVM 和 Java的关系。因此研究 Runtime 有助于了解 OC 的运行，帮助认识 OC 的本质。

Runtime 分为两个版本，“modern”和“legacy”。modern 版本从 OC2.0 开始引入，在[Objective-C Runtime Reference](https://developer.apple.com/reference/objectivec/1657527-objective_c_runtime)中有详细的开发接口描述。modern 版本相比于 legacy 版本的最大亮点在于：在 legacy 版本的 Runtime 中，如果你改变了实例变量的布局，就必须重新编译所有的子类，在 modern 版本的 Runtime 中则不需要。另外， modern 版本 Runtime 支持从声明的 property 综合推理（synthesis）出实例变量。

iPhone 应用和 OS X v10.5以及之后版本的 64-bit 程序使用的是 modern 版本，其余的应用使用的是 legacy 版本。<!-- More -->

> legacy 版本开发接口在 Objective-C 1 Runtime Reference 中有描述，但基本不需要去关心了。

### 调用 Runtime 的方式

OC 语言和 Runtime 有三种层次的交互：

* __通过 OC 源码交互。__ 这种交互并不是说通过 OC 方法去调用 runtime ，而是指编译器会通过创建合适的数据结构和方法调用来实现语言的动态特征，比如用于描述类、Category、协议以及 selector。最主要的一个 runtime 函数就是用于发送消息的，后面会有描述，它就是通过 OC 层面的消息发送表达式来调用的。
* __使用 Foundation 中定义的 NSObject 类交互。 __ Cocoa 中大部分的对象都是 NSObject 的子类，所以它们也继承了 NSobject 的方法，其中有部分方法是用于内省的，比如`class`方法——用于询问一个对象它的类是什么；用于判断继承关系的方法`isKindOfClass:`和`isMemberOfClass:`；判断一个对象是否能接受某个消息的方法`respondsToSelector:`；判断一个对象是否实现了某个协议的方法`conformsToProtocol:`；查询方法实现地址的方法`methodForSelector:`。这些方法赋予了对象内省的能力。
* __直接调用 runtime 函数。__ Runtime 系统是一个动态链接库，由一组函数和数据结构组成的，它位于系统的 /usr/include/objc 目录下面。这里的很多函数允许开发者在开发应用的时候使用纯 C 语言来复制编译器的行为。其中一些就是前面提到的 NSObject 内省函数的基础。所有的这些函数都可以在文档 [Objective-C Runtime Reference](https://developer.apple.com/reference/objectivec/1657527-objective_c_runtime) 中查阅。

### Runtime的内容 

Runtime总共包含三个方面的内容：

* 类的动态加载
* 消息发送
* 属性内省

下面一一阐述。

## 类的动态加载

OC 可以在运行时加载和链接新的类和 Category，加载完成后和程序中原始的类以及 Category 并无二致。我们有两种方式可以进行类动态加载：

1. objc/objc-load.h 文件中定义的 runtime 函数；
2. Cocoa 的 NSBundle 类；

具体操作因为文档没有描述，本文也不打算拓展，计划后面补充。

## 消息发送

在 OC 代码中我们通过`[]`来表达一个消息的发送，比如`[self class]`，这个调用最终会被转化为`objc_msgSend`函数调用。

### `objc_msgSend`函数

`objc_msgSend`函数的转换如下：如果 OC 中的调用是`[receiver message]`，那么会被转化为`objc_msgSend(receiver, selector)`，其中 selector 代表的就是 message，如果有参数，就会转化为`objc_msgSend(receiver, selector, arg1, arg2, ...)`调用样式，通过这个函数就可以实现动态绑定：

1. 它首先会找到 selector 指向的方法实现。由于不同的类可能会实现同样的方法（指的是方法名字一样），因此这个寻找过程是依赖于 receiver 的，简单来说，就是找到 receiver 上的这个方法的实现；
2. 之后调用这个方法实现，将对象实例以及参数传递给方法（传递对象实例是为了引用对象的实例变量）；
3. 最终它会返回方法返回的值；

> 苹果建议永远不要自己去调用这个函数。

发送消息的关键在于编译器为每一个类和对象创建的数据结构。每一个类都包含两个关键的元素：

* 指向父类的指针；
* 类分发表。这张表定义了 selector 到类定义方法的映射。

每一个对象实例创建之后，都会携带有一个指向类结构的指针变量——`isa`。这个指针可以让对象知道自己的类型以及自己的所有父类。

<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/消息发送框架.png" width="280" alt="消息发送框架"/></div>

如图，当一个对象接收到一个消息时，`objc_msgSend`函数会根据对象的`isa`指针找到类结构体，从而找到类的分发表，查询是否有 selector 对应的方法体，如果没有，则根据 superclass 找到父类的分发表，周而复始直到找到对应的方法体。__这就是运行时动态绑定方法的流程__。

> 找不到的处理后面会提到。

这种实现的缺点很明显：如果每次都去一层层找，肯定比直接进行方法调用要慢很多，这种差别就像在链表和数组中定位一个元素一样，一个通过下标定位，一个通过遍历定位，差距不言而喻。因此在每个类中会有额外的缓存结构来存储这个类经常使用到的方法——包括自身和父类的方法。对象每次收到一个消息的时候都会首先搜寻缓存结构，搜寻不到才会去搜寻分发表。通过这种方式，可以缩小消息发送和方法调用之间的速度差距。后面还会提到别的手段用于缩小这种差距。

> 缓存基于局部性原理。

### 隐藏的参数

当使用`objc_msgSend`函数传递消息的时候，它不仅会传递原来消息的所有参数，还会默认传递两个额外的参数：

1. 接受对象 receiver；
2. 表示方法的 selector；

这两个参数在编译器编译时被“偷偷”插入，从而使得`objc_msgSend`可以还原出完整的消息发送信息：

1. 消息是什么（selector）；
2. 消息是发送给谁的（receiver）；
3. 参数是什么；

虽然参数是隐形的，但是在代码里面却可以使用它们，它们分别是`self`和`_cmd`。比如下面的例子：

```objective-c
- (void)strange {
    id  target = getTheReceiver();
    SEL method = getTheMethod();
    if ( target == self || method == _cmd )
        return nil;
    return [target performSelector:method];
}
```

其中 self 就代表消息 strange 所在的对象，_cmd 就表示代表 strange 的 selector。__self 就是方法为什么可以操作对象实例变量的关键所在：通过self，可以引用到对象本身，从而可以引用对象的变量__。

### 获取方法地址

动态绑定处理消息方法的过程比较耗时，除了可以使用缓存加快查询速度，还可以通过获取方法地址直接调用方法的方式来避免动态绑定。这种方法并不常见，除非你在短时间内需要高频次的调用某个方法，并且直接调用已经造成性能问题。

NSObject 中有一个方法是用于获取方法地址的：`methodForSelector`，通过这个方法可以获得一个函数指针，用法如下：

```objective-c
void (*setter)(id, SEL, BOOL);
int i;
setter = (void (*)(id, SEL, BOOL))[target
    methodForSelector:@selector(setFilled:)];
for ( i = 0 ; i < 1000 ; i++ )
    setter(targetList[i], @selector(setFilled:), YES);
```

声明的函数指针中也标记除了隐形参数，在调用的时候需要把对应的值设置进去。

### 方法动态解析

OC 除了可以通过 `objc_msgSend`函数动态绑定已有的方法，还可以通过实现方法`resolveInstanceMethod:`和`resolveInstanceMethod:`动态提供类方法和实例方法实现。__OC 方法本质上来说就是至少接受 self 和 _cmd 两个参数的 C 函数__。通过调用方法`class_addMethod`可以为类添加一个函数，举例如下：

```objective-c
void dynamicMethodIMP(id self, SEL _cmd) {
    // implementation ....
}

@implementation MyClass
+ (BOOL)resolveInstanceMethod:(SEL)aSEL {
	if (aSEL == @selector(resolveThisMethodDynamically)) {
		class_addMethod([self class], aSEL, (IMP) dynamicMethodIMP, "v@:");
		return YES;
	}
	return [super resolveInstanceMethod:aSEL];
}
@end
```

消息转发（后面会提）和方法动态解析绝大部分情况下是正交的，方法动态解析会在消息转发之前进行。如果`respondsToSelector:`或者`instancesRespondToSelector:`被调用，方法动态解析是有机会为 selector 提供一个 IMP（可以认为是函数指针）的。但如果想在动态解析之后进行消息转发，这里可以返回 NO。

### 消息转发

前面提到，`objc_msgSend`函数会在分发表中寻找 selector 对应的方法体，如果找到 NSObject 还是找不到怎么办呢？一种情况就是动态方法解析，如果返回 YES，就表示对于的方法体已经添加到了类中，NO 则相反。那么如果是返回是 NO，那是不是程序就出错，直接 crash 了呢？OC 在这个时候还提供了最后一道关卡：在报错之前，runtime 会向消息的原本发送对象发送一个`forwardInvocation:`消息，并携带一个 NSInvocation 参数，这个参数包装了原始的消息和参数。我们可以在这个地方对一个消息进行最后的处理。

首先顾名思义，这个方法一般用于将消息转发给别的对象。通过实现这个方法，我们可以给消息一个默认的响应，或者通过别的手段来防止出错。如果我们不去重载这个方法，那么就会执行 NSObject 的默认实现，即调用`doesNotRecognizeSelector:`方法，从而报错。举例如下：

```objective-c
- (void)forwardInvocation:(NSInvocation *)anInvocation
{
    if ([someOtherObject respondsToSelector:[anInvocation selector]])
        [anInvocation invokeWithTarget:someOtherObject];
    else
    	[super forwardInvocation:anInvocation];
}
```

这方法可以扮演不能识别的消息的转发中心，或者作为一个中转站，将所有的消息转发到一个特定的对象；也可以将一个消息转换为别的消息，甚至直接吞没一些消息，以避免错误发生。简而言之，未被识别的消息以及未能动态解析的消息都会被转到这里进行处理，因此这里可以有很多玩法，比如多继承模拟，比如实现代理。

> 注意：想要这个方法被调用，还需要重载另外一个方法`methodSignatureForSelector:`，详细见下面例子。

除了`forwardInvocation`这个方法之外，还有一个方法也可以扮演这个角色——`forwardingTargetForSelector:`。这个方法通过返回一个消息的接收者来完成对消息的转发，后面示例会给出用法。

#### 消息转发模拟多继承

消息转发是可以模拟出多继承的。如下图：

<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/消息转发.png" width="320" alt="消息转发"/></div>

Warrior 通过把 negotiate 方法转发给 Diplomat，看上去就像是继承了 Diplomat，另外一方面 Warrior 本身就继承了 NSObject，从而就具备了两条继承线。

> 这种多继承的“模拟”其实就是组合的一个变种，只不过实现起来更加方便。

实际上要完全模拟多继承并没有这么简单，我们需要修改很多默认的内省函数来进一步完善这种模拟，比如:

* `respondsToSelector:`
* `isKindOfClass: `
* `nstancesRespondToSelector: `
* `methodSignatureForSelector:`
* `methodSignatureForSelector:`

举个例子：

```objective-c
- (BOOL)respondsToSelector:(SEL)aSelector
{
    if ( [super respondsToSelector:aSelector] )
        return YES;
    else {
        /* Here, test whether the aSelector message can     *
         * be forwarded to another object and whether that  *
         * object can respond to it. Return YES if it can.  */
    }
	return NO; 
}
```

### 小结

前面描述了消息处理的过程，总共包括三种方式：

1. 分发表；
2. 动态解析；
3. 消息转发；

下面给一个例子实践一下，为了节省篇幅和方便阅读，h 和 m 就写在一个代码块里面了。首先声明 MsgTest 类：

```objective-c
@interface MsgTest : NSObject
-(void)realMethod;
@end
  
void dynamicMethodIMP(id self, SEL _cmd) {
    NSLog(@"%@", @"Dynamic Resolving.");
}

@interface MsgTest ()
@property(nonatomic, strong) MsgResovler *msgResolver;
@end

@implementation MsgTest
@synthesize msgResolver;

-(instancetype)init{
    if(self = [super init]){
        msgResolver = [[MsgResovler alloc] init];
    }
    return self;
}

-(void)realMethod{
    NSLog(@"%@", @"Real Method");
}

+ (BOOL)resolveInstanceMethod:(SEL)aSEL {
    if (aSEL == @selector(resolveThisMethodDynamically)) {
        class_addMethod([self class], aSEL, (IMP) dynamicMethodIMP, "v@:");
        return YES;
    }
    return [super resolveInstanceMethod:aSEL];
}

- (id)forwardingTargetForSelector:(SEL)aSelector {
    if (aSelector == @selector(forwardTarget)) {
        return msgResolver; // 返回实际处理对象
    }
    return nil;
}

- (void)forwardInvocation:(NSInvocation *)anInvocation
{
    if ([msgResolver respondsToSelector:[anInvocation selector]])
        [anInvocation invokeWithTarget:msgResolver];
    else
        [super forwardInvocation:anInvocation];
}

// 消息转发的时候，不要忘了实现这个方法
- (NSMethodSignature*)methodSignatureForSelector:(SEL)selector
{
    NSMethodSignature* signature = [super methodSignatureForSelector:selector];
    if (!signature) {
        signature = [msgResolver methodSignatureForSelector:selector];
    }
    return signature;
}
@end
```

这是核心类，会根据前面提到的消息处理方式分别接受四种消息，接着为消息转发声明消息接受类 MsgResovler：

```objective-c
@interface MsgResovler : NSObject
-(void)forwardMsg;
-(void)forwardTarget;
@end
  
@implementation MsgResovler
-(void)forwardMsg {
    NSLog(@"%@", @"Forwarded Msg");
}

-(void)forwardTarget{
    NSLog(@"%@", @"Forwarding Target");
}
@end
```

最后尝试调用：

```objective-c
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        MsgTest *msgTest = [[MsgTest alloc] init];
        NSLog(@"%@", @"============= 真实方法 =============");
        [msgTest realMethod];
        NSLog(@"%@", @"============= 动态解析 =============");
        [msgTest performSelector:@selector(resolveThisMethodDynamically)];
        NSLog(@"%@", @"============= 消息转发 =============");
        [msgTest performSelector:@selector(forwardMsg)];
      	NSLog(@"%@", @"============= 消息转发 =============");
        [msgTest performSelector:@selector(forwardTarget)];
    }
    return 0;
}
```

就可以看到控制台输出：

```xml
============= 真实方法 =============
Real Method
============= 动态解析 =============
Dynamic Resolving.
============= 消息转发 =============
Forwarded Msg
============= 消息转发 =============
Forwarding Target
Program ended with exit code: 0
```

四个方法都顺利完成调用，没有报错。那么这几种方法的调用顺序是什么样子的呢？借用网上一幅图：

<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/消息发送路线.png" width="360" alt="消息转发路线"/></div>

一目了然。

## 属性内省

### Type Encoding

在说属性内省之前，需要先说一个概念——Type Encoding。Type Encoding 实际是用于表达方法返回值和参数的简洁表达式，开发者可以通过`@encode()`编译命令对某一个特定类型进行转化，比如：

```objective-c
char *buf1 = @encode(int **);//^^i
char *buf2 = @encode(struct key);//要看key内容
char *buf3 = @encode(Rectangle);//要看Rectangle类
```

具体对应列表如下：

|      Code      |                 Meaning                  |
| :------------: | :--------------------------------------: |
|       c        |                  A char                  |
|       i        |                  An int                  |
|       s        |                 A short                  |
|       l        |                  A long                  |
|       q        |               A long long                |
|       C        |             An unsigned char             |
|       I        |             An unsigned int              |
|       S        |            An unsigned short             |
|       L        |             An unsigned long             |
|       Q        |          An unsigned long long           |
|       f        |                 A float                  |
|       d        |                 A double                 |
|       B        |          A C++ bool or C99_Bool          |
|       v        |                  A void                  |
|       *        |        A character string(char *)        |
|       @        | An object(whether statically typed or typed id) |
|       #        |          A class object(Class)           |
|       :        |          A method selector(SEL)          |
|  [array type]  |                 An array                 |
| {name=type...} |               A structure                |
|  (name=type…)  |                 A union                  |
|      bnum      |         A bit bield of num bits          |
|     ^type      |                A pointer                 |
|       ?        | An unknown type(among other things, this code is used for function pointers) |

> 注意：OC 不支持 long double 类型，@encode(long double) 会返回 d，也就是会被当做 double 执行。

下面是一些例子：

|       例子       |                    解释                    |
| :------------: | :--------------------------------------: |
|     [12^f]     |         一个包含12个指向 float 元素的指针的数组         |
| {example=@*i}  |     一个包含对象、char指针和int值的结构体，名为example     |
| ^{example=@*i} |                 上述结构体的指针                 |
|  ^^{example}   |           上述指针的指针，这时候可以忽略结构体内容           |
| {NSObject=# }  | 对象会被当做结构体对待，@encode(NSObject)就会产生左边的表达式，NSObject 的对象实例只有一个 Class 类型的 isa 指针。 |

>关于@encode{NSObject}，输出的其实是NSObject这个类它的实例的表达式：包含一个 isa 指针以及属性。

### Property 类型和函数

当编译器在编译 property 的时候，它会产生一些与类、Category以及协议相关的描述元数据。OC 提供了一些函数可以通过名字搜索类或者协议上的 property，从而获取这些元数据。

Property 实际是一个 objc_property 指针：

```objective-c
typedef struct objc_property *Property;
```

开发者可以通过`class_copyPropertyList `和` protocol_copyPropertyList `两个函数来获取类/协议的属性数组：

```objective-c
objc_property_t *class_copyPropertyList(Class cls, unsigned int *outCount)
objc_property_t *protocol_copyPropertyList(Protocol *proto, unsigned int *outCount)
```

举个例子，声明如下一个类：

```objective-c
@interface Lender : NSObject {
    float ins;
}
@property float alone;
@end
```

通过如下方法可以获取属性列表：

```objective-c
id LenderClass = objc_getClass("Lender");
unsigned int outCount;
objc_property_t *properties = class_copyPropertyList(LenderClass, &outCount);
```

获取属性之后，可以根据`property_getName`来获取属性名：

```objective-c
const char *property_getName(objc_property_t property)
```

通过指定某个类的某个属性名字，也可以获取到一个属性：

```objective-c
objc_property_t class_getProperty(Class cls, const char *name)
objc_property_t protocol_getProperty(Protocol *proto, const char *name, BOOL
isRequiredProperty, BOOL isInstanceProperty)
```

接着调用`property_getAttributes`函数可以获取属性的名字以及 @encode 类型字符串。

```objective-c
const char *property_getAttributes(objc_property_t property)
```

>@encode 类型字符串 可以参见[Property Type String](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtPropertyIntrospection.html#//apple_ref/doc/uid/TP40008048-CH101-SW1)和[Property Attribute Description Examples](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtPropertyIntrospection.html#//apple_ref/doc/uid/TP40008048-CH101-SW1)。

把这些代码融合在一块儿，就可以获得一个类/协议的所有属性：

```objective-c
id LenderClass = objc_getClass("Lender");
unsigned int outCount, i;
objc_property_t *properties = class_copyPropertyList(LenderClass, &outCount);
for (i = 0; i < outCount; i++) {
	objc_property_t property = properties[i];
	fprintf(stdout, "%s %s\n", property_getName(property), property_getAttributes(property));
}
```

输出结果如下：

```O
alone Tf,V_alone
```

> 实践结果表明：Category 中的属性会被一并打印，而实例变量不会被打印。

## 总结

[《ObjC Runtime Guide》](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008048-CH1-SW1)是学习 Runtime 的入门读物，本文是该文档的阅读笔记，主要目的是搞清楚两件事情：

1. 什么是 Runtime；
2. Runtime 能做什么；

除此之外没有任何的探究拓展。在此基础上，后续会探索 Runtime 是如何实现的，也就是本文提到的函数和数据结构，进而发现 Runtime 更为本质的内在和更为强大的用法。