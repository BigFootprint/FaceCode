---
title: Java动态代理实现
date: 2015-12-13 15:10:05
tags: [源码]
categories: Java
---

本文整理自《Thinking In Java》和 IBM 技术博客[Java 动态代理机制分析及扩展](https://www.ibm.com/developerworks/cn/java/j-lo-proxy1/)，对动态类的生成部分做了补充。<!--more-->

## 使用
例子来自《Thinking In Java》。

```java
class MethodSelector implements InvocationHandler {
     private Object proxied;
     public MethodSelector(Object proxied) {
          this.proxied = proxied;
     }
     public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
          if(method.getName().equals("interesting"))
               print("Proxy detected the interesting method");
          return method.invoke(proxied, args);
     }
}

interface SomeMethods {
     void boring1();
     void boring2();
     void interesting(String arg);
     void boring3();
}

class Implementation implements SomeMethods {
     public void boring1() { print("boring1"); }
     public void boring2() { print("boring2"); }
     public void interesting(String arg) {
          print("interesting " + arg);
     }
     public void boring3() { print("boring3"); }
}

class SelectingMethods {
     public static void main(String[] args) {
          SomeMethods proxy= (SomeMethods)Proxy.newProxyInstance(
               SomeMethods.class.getClassLoader(),
               new Class[]{ SomeMethods.class },
               new MethodSelector(new Implementation()));
          proxy.boring1();
          proxy.boring2();
          proxy.interesting("bonobo");
          proxy.boring3();
     }
}

//输出
boring1
boring2
Proxy detected the interesting method
interesting bonobo
```
它的使用非常简单，重点就在于InvocationHandler的实现。这里可以对方法的调用做出转发，很像APO？

## 实现
### 分析前提
【环境】JDK 1.6  
【代码】[Proxy.java](http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/6-b27/java/lang/reflect/Proxy.java#Proxy.newProxyInstance%28java.lang.ClassLoader%2Cjava.lang.Class%5B%5D%2Cjava.lang.reflect.InvocationHandler%29)

### 代理机制及其特点
首先让我们来了解一下如何使用 Java 动态代理。具体有如下四步骤：

1. 通过实现 InvocationHandler 接口创建自己的调用处理器；
2. 通过为 Proxy 类指定 ClassLoader 对象和一组 interface 来创建动态代理类；
3. 通过反射机制获得动态代理类的构造函数，其唯一参数类型是调用处理器接口类型；
4. 通过构造函数创建动态代理类实例，构造时调用处理器对象作为参数被传入；

而实际使用的时候，我们只需要做如下两步:

```java
// InvocationHandlerImpl 实现了 InvocationHandler 接口，并能实现方法调用从代理类到委托类的分派转发
InvocationHandler handler = new InvocationHandlerImpl(..); 

// 通过 Proxy 直接创建动态代理类实例
Interface proxy = (Interface)Proxy.newProxyInstance( classLoader, new Class[] { Interface.class }, handler );
```

### 变量
```java
// 映射表：用于维护类装载器对象到其对应的代理类缓存
private static Map loaderToCache = new WeakHashMap(); 

// 标记：用于标记一个动态代理类正在被创建中
private static Object pendingGenerationMarker = new Object(); 

// 同步表：记录已经被创建的动态代理类类型，主要被方法 isProxyClass 进行相关的判断
private static Map proxyClasses = Collections.synchronizedMap(new WeakHashMap()); 

// 关联的调用处理器引用
protected InvocationHandler h;

//代理类实例化构建参数
private final static Class[] constructorParams =
        { InvocationHandler.class };
```
由变量可以猜测，代理的创建过程中是使用了很多的缓存的。

### 静态构造方法
```java
public static Object newProxyInstance(ClassLoader loader, 
            Class<?>[] interfaces, 
            InvocationHandler h) 
            throws IllegalArgumentException { 
    
    // 检查 h 不为空，否则抛异常
    if (h == null) { 
        throw new NullPointerException(); 
    } 

    // 获得与制定类装载器和一组接口相关的代理类类型对象
    Class cl = getProxyClass(loader, interfaces); 

    // 通过反射获取构造函数对象并生成代理类实例
    try { 
        Constructor cons = cl.getConstructor(constructorParams); 
        return (Object) cons.newInstance(new Object[] { h }); 
    } catch (NoSuchMethodException e) { throw new InternalError(e.toString()); 
    } catch (IllegalAccessException e) { throw new InternalError(e.toString()); 
    } catch (InstantiationException e) { throw new InternalError(e.toString()); 
    } catch (InvocationTargetException e) { throw new InternalError(e.toString()); 
    } 
}
```
这个方法很简单，主要是调用getProxyClass方法获取一个cl，然后通过反射创建出代理类的实例。很明显，核心在于getProxyClass到底做了什么？

### getProxyClass方法
这个方法比较长，分为四个主体部分:

【1】对这组接口进行一定程度的安全检查，包括检查接口类对象是否对类装载器可见并且与类装载器所能识别的接口类对象是完全相同的，还会检查确保是 interface 类型而不是 class 类型。这个步骤通过一个循环来完成，检查通过后将会得到一个包含所有接口名称的字符串数组，记为 String[] interfaceNames。总体上这部分实现比较直观，所以略去大部分代码，仅保留留如何判断某类或接口是否对特定类装载器可见的相关代码。

```java
try { 
    // 指定接口名字、类装载器对象，同时制定 initializeBoolean 为 false 表示无须初始化类
    // 如果方法返回正常这表示可见，否则会抛出 ClassNotFoundException 异常表示不可见
    interfaceClass = Class.forName(interfaceName, false, loader); 
} catch (ClassNotFoundException e) { 
}
```
这里的目的是保证代理类创建的时候，和它所有需要代理的接口都在一个ClassLoader上可见。并且禁止传入类（后面解释），禁止重复传入。

【2】从 loaderToCache 映射表中获取以类装载器对象为关键字所对应的缓存表，如果不存在就创建一个新的缓存表并更新到 loaderToCache。缓存表是一个 HashMap 实例，正常情况下它将存放键值对（接口名字列表，动态生成的代理类的类对象引用）。当代理类正在被创建时它会临时保存（接口名字列表，pendingGenerationMarker）。标记 pendingGenerationMarke (其实是一个对象)的作用是通知后续的同类请求（接口数组相同且组内接口排列顺序也相同）代理类正在被创建，请保持等待直至创建完成。

```java
    // 以接口名字列表作为关键字获得对应 cache 值
    Object value = cache.get(key); 
    if (value instanceof Reference) { 
        proxyClass = (Class) ((Reference) value).get(); 
    } 
    if (proxyClass != null) { 
        // 如果已经创建，直接返回
        return proxyClass; 
    } else if (value == pendingGenerationMarker) { 
        // 代理类正在被创建，保持等待
        try { 
            cache.wait(); 
        } catch (InterruptedException e) { 
        } 
        // 等待被唤醒，继续循环并通过二次检查以确保创建完成，否则重新等待
        continue; 
    } else { 
        // 标记代理类正在被创建
        cache.put(key, pendingGenerationMarker); 
        // break 跳出循环已进入创建过程
        break; 
} while (true);
```

【3】动态创建代理类的类对象。首先是确定代理类所在的包，其原则如前所述，如果都为 public 接口，则包名为空字符串表示顶层包；如果所有非 public 接口(package可见)都在同一个包，则包名与这些接口的包名相同；如果有多个非 public 接口且不同包，则抛异常终止代理类的生成。

```java
String proxyPkg = null;     // package to define proxy class in

	/*
	 * Record the package of a non-public proxy interface so that the
	 * proxy class will be defined in the same package.  Verify that
	 * all non-public proxy interfaces are in the same package.
	 */
	for (int i = 0; i < interfaces.length; i++) {
		int flags = interfaces[i].getModifiers();
		if (!Modifier.isPublic(flags)) {
			String name = interfaces[i].getName();
			int n = name.lastIndexOf('.');
			String pkg = ((n == -1) ? "" : name.substring(0, n + 1));
			if (proxyPkg == null) {
				proxyPkg = pkg;
			} else if (!pkg.equals(proxyPkg)) {
				throw new IllegalArgumentException("non-public interfaces from different packages");
			}
		}
	}

	if (proxyPkg == null) {     // if no non-public proxy interfaces,
		proxyPkg = "";          // use the unnamed package
	}
```

确定了包后，就开始生成代理类的类名，同样如前所述按格式“$ProxyN”生成。类名也确定了，接下来就是见证奇迹的发生 —— 动态生成代理类：

```java
/*
 * Choose a name for the proxy class to generate.
 */
long num;
synchronized (nextUniqueNumberLock) {
	num = nextUniqueNumber++;
}

String proxyName = proxyPkg + proxyClassNamePrefix + num;
/*
 * Verify that the class loader hasn't already
 * defined a class with the chosen name.
 */

/*
 * Generate the specified proxy class.
 */
byte[] proxyClassFile = ProxyGenerator.generateProxyClass(proxyName, interfaces);
try {
	proxyClass = defineClass0(loader, proxyName,
	proxyClassFile, 0, proxyClassFile.length);
} catch (ClassFormatError e) {
	/*
 	 * A ClassFormatError here means that (barring bugs in the
 	 * proxy class generation code) there was some other
 	 * invalid aspect of the arguments supplied to the proxy
 	 * class creation (such as virtual machine limitations
 	 * exceeded).
  	 */
 	throw new IllegalArgumentException(e.toString());
}
```
这里通过proxyPkg + proxyClassNamePrefix + num拼接出代理类的名字，一般proxyPkg为空，proxyClassNamePrefix是一个final字符串"$Proxy"，num则是一个递增数字，可以看做是创建的第N个代理类。最后调用ProxyGenerator的generateProxyClass生成代理类的byte[]数组，以一个native方法defineClass0生成最终的代理类class对象。

【4】代码生成过程进入结尾部分，根据结果更新缓存表，如果成功则将代理类的类对象引用更新进缓存表，否则清楚缓存表中对应关键值，最后唤醒所有可能的正在等待的线程。

```java
/*
 * We must clean up the "pending generation" state of the proxy
 * class cache entry somehow.  If a proxy class was successfully
 * generated, store it in the cache (with a weak reference);
 * otherwise, remove the reserved entry.  In all cases, notify
 * all waiters on reserved entries in this cache.
 */
synchronized (cache) {
	if (proxyClass != null) {
		cache.put(key, new WeakReference(proxyClass));
	} else {
		cache.remove(key);
	}
	cache.notifyAll();
}
```

以上四步展现了创建一个代理类的主体，到现在只剩下ProxyGenerator未描述了。
在IBM的原博客中说:
>当你尝试去探索这个类时，你所能获得的信息仅仅是它位于并未公开的 sun.misc 包，有若干常量、变量和方法以完成这个神奇的代码生成的过程，但是 sun 并没有提供源代码以供研读。

但是到我写这篇博客的时候，ProxyGenerator.java已经可以看到了，可以在这里[下载](http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/6-b27/sun/misc/ProxyGenerator.java#ProxyGenerator)。

## ProxyGenerator类
### generateProxyClass方法
```java
public static byte[] generateProxyClass(final String name, Class[] interfaces) {
ProxyGenerator gen = new ProxyGenerator(name, interfaces);
final byte[] classFile = gen.generateClassFile();
//...省略，这里是将byte持久化。
```
继续追踪generateClassFile方法。

### generateClassFile方法
从方法名字可以猜出，这个方法用于生成一个Class对象。这个方法也非常长，分为3步。

【1】收集创建Class对象的所有方法，并创建对应的ProxyMethod对象，以便于后面生成代理代码:

```java
addProxyMethod(hashCodeMethod, Object.class);
addProxyMethod(equalsMethod, Object.class);
addProxyMethod(toStringMethod, Object.class);

for (int i = 0; i < interfaces.length; i++) {
	Method[] methods = interfaces[i].getMethods();
	for (int j = 0; j < methods.length; j++) {
		addProxyMethod(methods[j], interfaces[i]);
	}
}

for (List<ProxyMethod> sigmethods : proxyMethods.values()) {
	checkReturnTypes(sigmethods);
}
```
添加了Object的基础方法和接口的所有方法。在添加方法的时候，会生成一个由方法名字+参数作为Key，List<ProxyMethod>作为值的map，添加完成后会做一次check，即checkReturnTypes。关于这个方法，注释如下：
> For a given set of proxy methods with the same signature, check that their return types are compatible according to the Proxy specification. Specifically, if there is more than one such method, then all of the return types must be reference types, and there must be one return type that is assignable to each of the rest of them.

【2】收集产生类所需要的所有的MethodInfo和FieldInfo对象
代码如下：

```java
methods.add(generateConstructor());

for (List<ProxyMethod> sigmethods : proxyMethods.values()) {
	for (ProxyMethod pm : sigmethods) {
		// add static field for method's Method object
		fields.add(new FieldInfo(pm.methodFieldName,
			"Ljava/lang/reflect/Method;",
			ACC_PRIVATE | ACC_STATIC));

		// generate code for proxy method and add it
		methods.add(pm.generateMethod());
	}
}
```

这里可以看到，method中嗨添加了构造函数和静态初始化函数。

【3】写入最终的类文件
接下去代码的含义是：按照Java类文件编译后的格式，创建一个类出来。我们看一下重点(注意，下面贴的代码在原代码中不是连贯的，为了方便，特意截取几行展示)：

```java
/** name of the superclass of proxy classes */
private final static String superclassName = "java/lang/reflect/Proxy";
    
dout.writeInt(0xCAFEBABE);
dout.writeShort(cp.getClass(superclassName));

for (MethodInfo m : methods) {
	m.write(dout);
}
```

写入的第一个内容就是__dout.writeInt(0xCAFEBABE);__，而0xCAFEBABE就是标记 java class 文件的魔数。之后会写入superclassName，而这个类就是java/lang/reflect/Proxy，即Proxy类，所以所有自动生成的代理类都是Proxy的子类，这也就解释了为什么自动生成的代理类不能代理一个具体类，而只能代理一个接口：Java中没有多继承，后面会有更直接的证据。之后就是写入方法。写入方法这一块比较复杂，我们来采取一些特殊措施看看具体生成的文件是什么。接着最前面的使用例子，我们在main函数中写入如下代码:

```java
Implementation implementation = new Implementation();
Class[] interfaces = implementation.getClass().getInterfaces();
byte[] proxyClassFile = ProxyGenerator.generateProxyClass("MethodSelector", interfaces);
File f = new File("/Users/muzileecoding/Desktop/MethodSelector.class");
try{
	FileOutputStream fos = new FileOutputStream(f);
	fos.write(proxyClassFile);
	fos.flush();
	fos.close();;
} catch (FileNotFoundException e) {
	e.printStackTrace();
} catch (IOException e) {
	e.printStackTrace();
}
```

运行之后就可以在相应的文件夹下面获取到MethodSelector.class文件，将文件反编译（拖入IDE打开即可），下面是完整的代码：

```java
import com.footprint.reflection.SomeMethods;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;

public final class MethodSelector extends Proxy implements SomeMethods {
    private static Method m1;
    private static Method m4;
    private static Method m6;
    private static Method m5;
    private static Method m0;
    private static Method m3;
    private static Method m2;

    public MethodSelector(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return ((Boolean)super.h.invoke(this, m1, new Object[]{var1})).booleanValue();
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final void boring2() throws  {
        try {
            super.h.invoke(this, m4, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void boring3() throws  {
        try {
            super.h.invoke(this, m6, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void interesting(String var1) throws  {
        try {
            super.h.invoke(this, m5, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final int hashCode() throws  {
        try {
            return ((Integer)super.h.invoke(this, m0, (Object[])null)).intValue();
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void boring1() throws  {
        try {
            super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[]{Class.forName("java.lang.Object")});
            m4 = Class.forName("com.footprint.reflection.SomeMethods").getMethod("boring2", new Class[0]);
            m6 = Class.forName("com.footprint.reflection.SomeMethods").getMethod("boring3", new Class[0]);
            m5 = Class.forName("com.footprint.reflection.SomeMethods").getMethod("interesting", new Class[]{Class.forName("java.lang.String")});
            m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);
            m3 = Class.forName("com.footprint.reflection.SomeMethods").getMethod("boring1", new Class[0]);
            m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```
这个就是最终写出的java动态代理类：

1. 动态生成类的时候生成了一个只需要InvocationHandler实例作为参数的构造函数的，结合前面创建出动态代理文件之后创建动态代理实例的反射代码，这里不难理解；
2. 所有的方法体都变成了__super.h.invoke(this, m4, (Object[])null);__这样的调用，而m4代表着接口中的一个方法，从这里可以知道，实际生成的动态代理类，调用了我们创建代理时传入的InvocationHandler实例，并且将我们的返回结果直接返回，这就是动态代理的本质；

至此，我们传入的三个参数如何转化为最终的动态代理类，已经比较清楚了。

## 总结
__动态代理的原理实际上就是：__根据我们要代理的接口，在接口加载的ClassLoader上动态生成一个动态代理类（这个类原来不存在），动态代理类需要我们传入的InvocationHandler作为构造参数，这个代理类中会有所有我们需要代理的接口的方法实现，而这个实现，就是简单的调用我们传入的InvocationHandler的invoke方法。所以如果这样写:

```java
public Object invoke(Object o, Method method, Object[] objects) throws Throwable {
	return method.invoke(o, objects);
}
```
就会造成循环调用，导致栈溢出。

Proxy.newProxyInstance方法的返回值只能强转为它代理的其中一个接口，原因很简单，因为生成的类签名是这样的:

```java
public final class MethodSelector extends Proxy implements SomeMethods
```

以上两个点要注意一下。

最后，正如 IBM 博客中所说，也如前面分析的，Java自带的动态代理是不能实现代理实体类或者抽象类的，这一点美中不足，这意味着我一旦要代理一个类，就必须抽象出响应的方法。但实际上，应该有随意代理一个类的成熟实现了，因为AOP的实现就依赖于此。So:
>不完美并不等于不伟大，伟大是一种本质，Java 动态代理就是佐例。  
>
>——[Java 动态代理机制分析及扩展](https://www.ibm.com/developerworks/cn/java/j-lo-proxy1/)
