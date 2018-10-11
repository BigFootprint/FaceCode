---
title: Proguard
date: 2016-05-12 15:49:01
tags: [性能优化, 工具]
categories: Android
---

## 简介
__Proguard__ 主要提供以下功能：

1. 创建更加紧凑的代码，让最终的代码更加小巧，可以更快的在网络上传输、加载，运行时可以消耗更小的内存；
2. 混淆代码，使得程序逆向工程起来更加困难；
3. 列出无用的代码，以便于可以移除它们；
4. 重新定位以及校验 Java 6 以及以上版本的 class 文件：添加预校验信息到class文件中去，减轻 ClassLoader 的校验负担——详细解释可见[What is preverification?](http://proguard.sourceforge.net/FAQ.html#preverification)；

<!--more-->__Proguard__ 执行起来的速度非常快，效果显著，并且在 Ant、Gradle 等工具上已有插件实现。通过简单的模板化的配置加上简单的命令行选项，通常就足够对我们的代码做出出色的优化。

>1. 具体的优化结果可以查看[这里](http://proguard.sourceforge.net/results.html)；
>2. 通过删除无用的代码(包括优化库文件)以及将变量名类名简化，可以大幅度减小包的体积；
>3. Proguard 专门为 Android 定制了一个优化和混淆版本: __DexGuard__。它专注于 App 的保护，另外提供诸如资源混淆、字符串加密、类加密、dex切分等功能。它直接创建 Dalvik 字节码。

__Proguard__对代码的处理过程如下:

![Proguard优化过程](http://7xktd8.com1.z0.glb.clouddn.com/Proguard优化过程.png)
如图所示，__Proguard__ 读取 Input jars，经过一系列处理，最终得到一个 Output jars。优化过程是可以多次进行的。

Proguard 要求指明 Input jars 的 Library jars —— 它们是你用来编译代码的库文件。库文件始终保持不变。

### Entry points
为了确定哪些代码应该被保留，哪些应该被丢弃或者混淆，开发者应该为代码指定若干__`entry points`__。这些 entry points 通常是拥有 main 方法的类，或者是 applet，midlets，activities 等等。

1. 在压缩阶段，__Proguard__ 从这些 entry points 出发，查找确认哪些类和变量是被使用的，所有其余的类以及变量都会被丢弃；
2. 在优化阶段，__Proguard__ 进一步优化代码，那些不是 entry points 的类和方法可以被设置成 private、final 或者 static 的，没有被用到的参数会被移除，一些方法也会被内联；
3. 在混淆阶段，__Proguard__ 重命名那些非 entry points 的类和成员变量，在整个过程中，都会保持它们的名字不变；
4. 预验证阶段是唯一一个不需要知道 entry points 的阶段；

### 反射
自动处理代码在面对反射和自省时都会出现一些问题。在使用__Proguard__ 的时候，那些会被动态调用（即通过名字调用）的类和变量都应该被标记为 entry points。举个🌰 ：`Class.forName()`可能会指向任何的运行时的类，不太可能知道哪些类应该保持原来的名字以供这个方法调用（类名可能来源于任何地方）。因此开发者就必须在配置文件中通过`-keep`指定这个类。

但是，__Proguard__会自动探测并处理以下情况：

```java
//以下类名或者变量名都是通过字符串写死，而不是一个随时可以改变的变量
Class.forName("SomeClass")
SomeClass.class
SomeClass.class.getField("someField")
SomeClass.class.getDeclaredField("someField")
SomeClass.class.getMethod("someMethod", new Class[] {})
SomeClass.class.getMethod("someMethod", new Class[] { A.class })
SomeClass.class.getMethod("someMethod", new Class[] { A.class, B.class })
SomeClass.class.getDeclaredMethod("someMethod", new Class[] {})
SomeClass.class.getDeclaredMethod("someMethod", new Class[] { A.class })
SomeClass.class.getDeclaredMethod("someMethod", new Class[] { A.class, B.class })
AtomicIntegerFieldUpdater.newUpdater(SomeClass.class, "someField")
AtomicLongFieldUpdater.newUpdater(SomeClass.class, "someField")
AtomicReferenceFieldUpdater.newUpdater(SomeClass.class, SomeType.class, "someField")
```
类名或者变量的名字当然可能不同，但是这种构建模式__Proguard__是可以识别的，因此通过以上方式引用的类以及变量在压缩阶段会被保留，而String参数(指的是这些方法的参数)会在混淆阶段被替换。
>即__Proguard__会识别一些模式下对类和变量的引用，并机智的判断出哪些类和变量需要保留或者替换。

另外，__Proguard__会对保留哪些类是有必要的给出意见。举个🌰：__Proguard__会注意到如下的构建过程:`(SomeClass)Class.forName(variable).newInstance()`，这有可能意味着`SomeClass`这个类/接口以及它的实现都应该被保留（注意，这里并不属于前面说到的__Proguard__可以自动识别的模式）。
>带有反射的代码还是要多加小心。

## 使用
使用__Proguard__通常需要一个配置文件，针对以上每一个步骤，__Proguard__都提供了丰富的配置项，具体的配置选项[这里](http://proguard.sourceforge.net/manual/usage.html)可以查询，同时也有很多[例子](http://proguard.sourceforge.net/manual/examples.html)可以参考。另外__Proguard__还有一个GUI组件可以使用，[这里](http://proguard.sourceforge.net/manual/gui.html)可查看详细。

遇到问题可以在[官网的问题列表](http://proguard.sourceforge.net/manual/troubleshooting.html)里面搜索一下。

#### 关于keep
keep选项，总共有以下几种:

| Keep                                     | From being removed or renamed | From being renamed          |
| ---------------------------------------- | ----------------------------- | --------------------------- |
| Classes and class members                | -keep                         | -keepnames                  |
| Class members only                       | -keepclassmembers             | -keepclassmembernames       |
| Classes and class members, if class members present | -keepclasseswithmembers       | -keepclasseswithmembernames |

如果不确认使用哪一个命令，可以使用`-keep`：这会保证指定的类以及类成员不会在压缩阶段被移除掉，在混淆阶段不会被重命名。
>【注意！！】  
>1. 如果指定了一个类而没有指定keep它的类成员，那么__Proguard__仅仅会保留类以及它的无参构造方法作为entry points，它仍然可能会移除、优化或者混淆它的类成员；
>2. 如果指定了一个方法，那么__Proguard__只会保留方法为entry points，它的代码仍然可能被优化或者修改；

#### Class Specifications
`keep`后面肯定跟的是一个类或者类成员的描述符，从而表达`keep`应该应用在哪些地方。这个描述符是有规范的（或者称为模板），即Class Specifications。

它的格式如下:

```java
[@annotationtype] [[!]public|final|abstract|@ ...] [!]interface|class|enum classname
    [extends|implements [@annotationtype] classname]
[{
    [@annotationtype] [[!]public|private|protected|static|volatile|transient ...] <fields> |
                                                                      (fieldtype fieldname);
    [@annotationtype] [[!]public|private|protected|static|synchronized|native|abstract|strictfp ...] <methods> |
                                                                                           <init>(argumenttype,...) |
                                                                                           classname(argumenttype,...) |
                                                                                           (returntype methodname(argumenttype,...));
    [@annotationtype] [[!]public|private|protected|static ... ] *;
    ...
}]
```
格式里面涉及到一些符号，解释如下:

| 符号   | 含义               |
| ---- | ---------------- |
| []   | 可选               |
| ...  | 前面列出的选项可能以任意数量出现 |
| \|   | 隔离一组可选项          |
| ()   | 把某一块说明组合到一块      |

`class`关键字指向任何的接口或者类，`interface`只指向接口。`enum`只指向枚举类，`interface`或者`enmu`加上`!`表示非接口或者非类。

每一个类都必须以全限定名出现，比如`java.lang.String`，内部类则以__$__分割，比如`java.lang.Thread$State`，类名的表达可以带有正则描述:

| 符号   | 含义                                       |
| ---- | ---------------------------------------- |
| ?    | 匹配任何一个单一字符，但是不包括包名分割符，即`.`，比如`mypackage.Test?`匹配`mypackage.Test1`，但是不匹配`mypackage.Test12` |
| \*   | 匹配类名的任何一部分，但不包括包名分隔符，即`.`，比如`mypackage.*Test*`匹配`mypackage.Test`和`mypackage.YourTestApplication`，但是不匹配`mypackage.mysubpackage.MyTest`，或者更简单的说，`mypackage.*`匹配`mypackage`下面的所有类，但不包括子package下面的类 |
| \*\* | 匹配类名的任何一个部分，可以包含任何数量的包名分隔符，即`.`，比如`**.Test`匹配除了根包下面的所有的包中的`Test`类，`mypackage.**`匹配`mypackage`包以及子包下面的所有类。 |

另外，单独的`*`可以表示任何类。

`extends`和`implements`通常用于限定类，目前它们是等效的，用于表述只有继承/实现某个类/接口的类，但是这个接口/类本身不在这个描述以内。

`@`描述符用于限定哪些使用特定的注解描述的类/类成员，`annotationtype`的描述和类名一致。

变量和方法的描述和Java中的很像，除了方法的参数列表不包括参数名（类似于Java Doc中的表述），描述符中可以包含以下通配符：

1. __`<init>`__ 匹配任何一个构造函数；
2. __`<fields>`__ 匹配任何一个变量；
3. __`<methods>`__ 匹配任何的方法；
4. __`*`__ 匹配任何的变量和方法； 

以上这些描述不包含任何的返回值，只有`<init>`有一个参数列表。变量和方法名字可以使用正则表达式，可以包含`?`和`*`，含义和前面一致，描述符中的类型也可以包含以下通配符：

1. __`%`__ 匹配任何一个原始类型（boolean、int等，但是不包括void）；
2. __`?`__ 在类（比如参数、返回值）名中匹配任何一个字符；
3. __`*`__ 也是用于类名，和前面类名的描述中关于`*`的描述一致；
4. __`**`__ 也是用于类名，和前面类名的描述中关于`**`的描述一致；
5. __`***`__ 匹配任何类型：原始类型、非原始类型、数组等；
6. __`···`__ 匹配任何数量的任何类型参数；

要注意: `?`、`*`和`**`永远不会匹配原始类型，只有`***`可以匹配上任何维度的数组。举个例子：`** get*()`匹配`java.lang.Object getObject()`，但是不会匹配`float getFloat()`或者`java.lang.Object[] getObjects()`。

构造函数是可以直接只用类名或者全限定类名进行描述的，因为在Java中，构造函数有参数列表，却没有返回值。

类和变量访问控制符通常是用来给更严格的限制通配符——类和变量必须有一致的访问控制符，同样，也可以使用`!`来表示不能设置这样的访问控制符。

Proguard甚至支持只有编译器才可以设置的控制符：`synthetic`，`bridge`和`varargs`。

>【注】以上翻译自[官网Usage](http://proguard.sourceforge.net/manual/usage.html)。如有错误，欢迎指出。

## Android
在Android上，可以使用如下方式启用__Proguard__:

```java
android {
    buildTypes {
        release {
            minifyEnabled true
            shrinkResources true
            proguardFiles getDefaultProguardFile(‘proguard-android.txt'),
                    'proguard-rules.pro'
        }
    }
    ...
}
```
通过`minifyEnabled`的设置，可以对代码进行压缩优化；通过`shrinkResources `的设置，可以对代码进行资源优化；`proguardFiles`设置则用于提供压缩规则：

1. `getDefaultProguardFile(‘proguard-android.txt')`方法会从Android SDK下的`tools/proguard/`目录读取__Proguard__的默认配置，在这个目录下面还有一个文件`proguard-android-optimize.txt`，它不仅包含相同的规则，而且会在字节码层级对APK进行分析，进一步优化APK文件，可以尝试；
2. `proguard-rules.pro`文件是开发者自定义规则的地方，该文件默认与`build.gradle`文件同级；

资源优化在代码压缩之后，因为只有移除不需要的代码之后才可以判断哪些资源无用。
>目前资源优化对于`values/`文件夹下面的资源不做移除，因为AAPT（Android Asset Packaging Tool）不允许。

以下这种方式可以为某种特殊的build variant添加新的混淆规则：

```java
android {
    ...
    buildTypes {
        release {
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android.txt'),
                   'proguard-rules.pro'
        }
    }
    productFlavors {
        flavor1 {
        }
        flavor2 {
            proguardFile 'flavor2-rules.pro'
        }
    }
}
```
>Build Type  + Product Flavor = Build Variant
>
>注意，这里在flavor2中设置的配置文件与buildTypes中的是追加关系，不是替代关系。

### 输出文件
混淆后，会在`<module-name>/build/outputs/mapping/release/`目录下输出下面的文件：

1. __dump.txt__ 描述apk文件中所有类文件间的内部结构；
2. __mapping.txt__ 提供了原始的类，方法，和字段名与混淆后代码之间的映射；
3. __seeds.txt__ 列出了未被混淆的类和成员；
4. __usage.txt__ 列出了从apk中删除的代码；

### 定义代码保留
某些情况下，使用默认的__Proguard__配置文件`proguard-android.txt`就足够了，__Proguard__仅仅会移除所有的未使用代码，但是有些情况__Proguard__很难判断，可能会移除一些不该被移除的代码，比如:

1. 一个仅仅在`AndroidManifest.xml`文件中使用的类；
2. 从JNI调用的方法；
3. 运行时操作的代码（比如反射和自省）；

这部分代码需要特别注意，可以使用以上提到的输出文件辅助排查。可以使用一下配置语句进行代码保留:

```java
-keep public class MyClass
```
还有一种办法就是使用`@Keep`注解，在类上使用这个注解，整个类都会保持原样，在方法/变量上使用该注解，则被注解的方法/变量以及它们的所在类都会保持原封不动，不过这个使用这个注解需要添加`Annotations Support Library`依赖。

关于资源的优化，[官网](http://developer.android.com/intl/zh-cn/tools/help/proguard.html)上有比较详细的描述，比较复杂，这里不整理了。同代码一样，也可以对资源进行保留，但是对于重复的资源、多语言资源、通过Id引用的资源都有可选的配置。

下面给出一个配置例子:

```java
-injars      bin/classes
-injars      libs
-outjars     bin/classes-processed.jar
-libraryjars /usr/local/java/android-sdk/platforms/android-9/android.jar

-dontpreverify
-repackageclasses ''
-allowaccessmodification
-optimizations !code/simplification/arithmetic
-keepattributes *Annotation*

-keep public class * extends android.app.Activity
-keep public class * extends android.app.Application
-keep public class * extends android.app.Service
-keep public class * extends android.content.BroadcastReceiver
-keep public class * extends android.content.ContentProvider

-keep public class * extends android.view.View {
    public <init>(android.content.Context);
    public <init>(android.content.Context, android.util.AttributeSet);
    public <init>(android.content.Context, android.util.AttributeSet, int);
    public void set*(...);
}

-keepclasseswithmembers class * {
    public <init>(android.content.Context, android.util.AttributeSet);
}

-keepclasseswithmembers class * {
    public <init>(android.content.Context, android.util.AttributeSet, int);
}

-keepclassmembers class * extends android.content.Context {
   public void *(android.view.View);
   public void *(android.view.MenuItem);
}

-keepclassmembers class * implements android.os.Parcelable {
    static ** CREATOR;
}

-keepclassmembers class **.R$* {
    public static <fields>;
}

-keepclassmembers class * {
    @android.webkit.JavascriptInterface <methods>;
}
```

### 反混淆
对于打印出的Crash Log等信息，因为被混淆的原因，导致这些信息阅读起来非常困难。因此我们需要一个工具__翻译__一下这些信息，可以使用如下命令进行:

```java
retrace.bat|retrace.sh [-verbose] mapping.txt [<stacktrace_file>]
```
`retrace`是一个工具，位于`<sdk-root>/tools/proguard/`目录下面(在Windows上是`retrace.bat`，在Mac/Linux上是`retrace.sh`。)。

[官网](http://proguard.sourceforge.net/)也有相关的文档，点击Retrace Manual下面的三个Tab就可以看到介绍、使用方式以及例子。

>Google Play自动在支持反混淆，可见[帮助中心](https://support.google.com/googleplay/android-developer/answer/6295281)。

## 限制
使用__Proguard__的时候，需要注意一些技术问题，它们都很容易规避或者解决:

1. __Proguard__的优化算法会假设被处理的代码不会故意抛出NPE/ArrayIndexOutOfBoundsExceptions/OutOfMemoryErrors/StackOverflowErrors来达到某个目的。举个例子，如果`myObject.myMethod()`这样的方法调用完全无效，它可能会移除它，它忽略`myObject`是null的可能性（也即：忽略开发者通过这种方式抛出NPE错误的意图）。某些时候这是一件好事，优化后的代码会抛出更少的异常。但如果这个假设是错误的，开发者应该使用`-dontoptimize`关闭优化功能；
2. __Proguard__的优化算法也会假设被处理的代码不会有`busy-waiting loops without at least testing on a volatile field`(翻译不好，大概是说通过一个无限循环去检测一个非volatile变量)，因此它也会移除此类代码。如果这项假设是错误的，开发者应该使用`-dontoptimize`关闭优化功能；
3. 如果Input jars和Library jars在有class在同一个包下面，则混淆后的Output jars有可能会包含一些与Library jars同名的类文件，尤其当Library jars之前就被混淆过。因此Input jars和Library jars不应该共享一个包；

## 常见问题
具体见[官网](http://proguard.sourceforge.net/FAQ.html)。这里整理几个重要的结论：

1. __Proguard__会自动处理`Class.forName("SomeClass")`和`SomeClass.class`的处理情况，在压缩阶段，这些类会被保留，在混淆阶段，这些类也会被替换掉；
2. 对于资源、String以及流程控制的混淆，直接看官网；
3. __Proguard__支持增量式混淆——即提供一个之前的mapping文件进行一次新的混淆；
4. __Proguard__允许自定义混淆字典；