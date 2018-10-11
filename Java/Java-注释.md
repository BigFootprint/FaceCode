---
title: Java 注释
date: 2016-12-22 16:41:06
tags: Java
categories: Java
---


算起来自己正儿八经写 Java 代码也有两年多了，每每看到 Android Framework 和成熟开源项目的代码，都有一种高大上的气息油然而生。这些代码除了本身写的好，注释通常也非常完善。而平日里接触到的公司代码以及自己写的代码都不是非常注意注释这块儿，写的非常随意甚至不写。

> 因此往往通过注释就能区别代码是来自开源项目还是公司内部自己写的。T T

其实 Java 在关于代码注释的写法这块，是有比较完善的规范的：[How to Write Doc Comments for the Javadoc Tool](http://www.oracle.com/technetwork/articles/java/index-137868.html)。<!-- More -->

## 简介

通常我们在学习一个 系统/SDK 的时候会遇到两类文档：API文档 & 开发指导文档，有的 系统/SDK 会把两类文档混成一个，但是官方并不建议这么做。

API 文档通常是由 Javadoc 从代码注释生成，它所应该标注的应当是与平台无关、实现无关的关于这个方法/类/属性的声明，以达到 Java 的 "write once, run anywhere" 的目标；应当是调用者和实现者之间的一个协议。

开发指导文档一般包括示例、开发术语定义、概念解释和存在的Bug、使用环境。毫无疑问这些描述也有助于开发者理解和使用 系统/SDK，但它并不是接口声明，而是根据接口声明进行的具体实现，在接口文档中不是必须的。Java 本身就将 API 文档和开发指导文档分开维护——JDK 文档包中包含 API 文档以及示例代码和开发指导文档，通过链接来关联两者。

## 标准写法

文档注释必须在类/属性/方法前面使用 HTML 格式书写（代码的任何地方都可以写注释，但是只有这些地方的注释会被生成到 API 文档中），标准格式包括两块：描述块 + 标记块。比如下面这个例子：

```java
/**
 * Returns an Image object that can then be painted on the screen. 
 * The url argument must specify an absolute {@link URL}. The name
 * argument is a specifier that is relative to the url argument. 
 * <p>
 * This method always returns immediately, whether or not the 
 * image exists. When this applet attempts to draw the image on
 * the screen, the data will be loaded. The graphics primitives 
 * that draw the image will incrementally paint on the screen. 
 *
 * @param  url  an absolute URL giving the base location of the image
 * @param  name the location of the image, relative to the url argument
 * @return      the image at the specified URL
 * @see         Image
 */
 public Image getImage(URL url, String name) {
        try {
            return getImage(new URL(url, name));
        } catch (MalformedURLException e) {
            return null;
        }
 }
```

前面的两段文字就是描述块，后面四行以`@`开头的就称为标记块。生成的 API 文档如下：

<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/API文档示例.png"  alt="API文档示例"/></div>

### 基本规范

比照上面的例子，文档注释的基本规范如下：

1. 使用`/** */`包裹；
2. 从 Javadoc 1.4 开始，每一行的开头 __*__ 可选（但是现在的 IDE 在格式化的时候都保持了这个习惯，注释因此看上去也更加整齐）；
3. 第一句话应该是方法的总结，Javadoc 在生成文档的时候会把这句话作为方法总结呈现在方法概要列表里面；
4. 如果注释描述不止一段，每一段之间需要加上`<p>`标记符；
5. 当遇到第一个以`@`字符开头的行的时候，就意味着描述块的结束；

### 细节技巧

#### 第一句描述

第一句描述会被作为总结呈现在 API 文档适当的地方，例子中展示的是方法，但对于类、属性、接口以及包都是适用的。第一句话通常以 blank，tab，换行符或者是第一个标记块结束。

> 99%的情况都是一个正常的英文句子，在中文中，以句号可以达到同样的效果。养成习惯，第一句写一个总结句即可。

#### 实现无关

写 API 注释的时候，官方家建议应当尽量与实现无关。实际生产中，像 Android 确实有部分代码是和 J2EE 服务端公用的，也有代码会有多份实现，因此这一点需要多加注意：__注释必须确保有足够的信息保证实现的一致，保证能够写出一致的测试 case，包括边界条件，参数范围，边界 case 等，说明清楚哪些细节可以自由实现__。

> 这一点很像 Java 虚拟机规范，列出了实现的很多参数和要求，也有很多方面未有涉及，由实现者自行决定。

如果必须写明一些实现相关的行为，建议可以另开一个段落，并以清晰的开头表明接下去的描述是实现相关的，下面是一个例子：

> On Windows systems, the path search behavior of the `loadLibrary` method is identical to that of the Windows API's `LoadLibrary` procedure.

这段注释就清晰的表明它描述的是 Windows 平台相关的内容。

#### 方法注释的自动复用

有三种情况，方法的注释可以自动从别的地方复用：

1. 子类覆写父类的方法；
2. 子接口覆写父接口的方法；
3. 子类实现接口的方法；

这三种情况下，都会自动复用被覆写/实现方法的注释（如果你自己写了注释，那就不会复制了，但使用[`{@inheritDoc}`](http://docs.oracle.com/javase/7/docs/technotes/tools/windows/javadoc.html#inheritDoc)可以做到）。

#### 对于关键字以及名字，使用`<code>`

对于以下在注释中引用到的字符串，建议使用`<code>`包裹：

1. Java 关键字；
2. 包、类、方法、接口、属性、参数名字；
3. 代码示例；

还有一种更加优雅的写法：[`{@code}`](http://docs.oracle.com/javase/7/docs/technotes/tools/windows/javadoc.html#code)。

> 实际使用时发现生成的文档上并没有明显的标记，对于长串的代码也没有格式化，不过在阅读注释的时候显得更加清晰。

#### 注意点

除了上述提到的，还有一些注意点：

1. 如果一个方法有很多重载形式，那么当需要表达一个通用形式的时候，一定不要再方法名后面加上括号，比如`add()`和`add(int, Object)`，直接用`add`代表这两种形式即可；
2. 使用第三人称而不是第二人称；
3. 通常方法是用来实现一个行为的，因此建议使用动词开头（看方法而异吧）；
4. 使用 this 而不是 the 来代替当前对象；
5. 注释不要只是复述方法名，要添加一些额外的描述（有时候这个很艰难）；

### Tag（标记）

Tag，就是前面例子中看到的`@param`、`@return`、`@see`这样的标记，它是大小写铭感的，必须写在某一行的开头（前面可以有空格或者可选的星号），否则就会被当做普通文本处理。

__Tag 分为两类，块 Tag (Block Tag)和行内 Tag(Inline Tag)。__前者只能出现在描述后面的 Tag 部分，形式是`@tag`；后者可以出现在描述块或者块 Tag 后面的描述中，形式是`{@tag}`。目前 Tag 包括：`@author`，`{@code}`，`{@docRoot}`，`@deprecated`，`@exception`，`{@inheritDoc}`，`{@link}`，`@{linkplain}`，`{@literal}`，`@param`，`@return`，`@see`，`serial`，`@serialData`，`@serialField`，`@since`，`@throws`，`{@value}`，`@version`。

下面简单介绍下用法，详细可见 [JAVADOC TAGS](http://docs.oracle.com/javase/7/docs/technotes/tools/windows/javadoc.html#link)。

#### @autohor name-text

可以使用多个`@author`表示多个作者，也可以使用`,`在一个`@author`中分隔多个作者。

#### @deprecated deprecated-text

deprecated-text 会被移到描述块前面，使用斜体表示并前缀一个加粗的告警提示 "Deprecated"。

这个 Tag 可以用在所有的可注释元素上，使用这个 Tag 的时候，至少在第一句话要说明 API 是什么时候被废弃，替代方案是什么，后续描述可以阐述为什么废弃这个 API。示例：

```java
/**
 * @deprecated  As of JDK 1.1, replaced by {@link #setBounds(int,int,int,int)}
 */
```

#### {@docRoot}

表示到当前生成文档根目录的相对的路径，一般用于指向存在于根目录下的指定文件，比如 Logo 或者版权声明。它不但可以在所有的可注释元素中使用，也可以用在命令行中。

命令行使用示例：` javadoc -bottom '<a href="{@docRoot}/copyright.html">Copyright</a>'`。在注释中：

```java
/**
 * See the <a href="{@docRoot}/copyright.html">Copyright</a>.
 */
```

#### {@inheritDoc}

这个标记会从继承树中最近的位置复制注释，从而允许在继承树的高层中进行一些通用的注释，在低层中进行更为详细的描述。这个标记可以在下面两个地方使用：

1. 在方法的描述块中。这种情况下，会从父类/接口中拷贝描述；
2. 在@return，@param和@throws标记的文本参数中。这种情况下，会从父方法的对应标记中进行拷贝；

> 详细的寻找过程如下：
>
> **Algorithm for Inheriting Method Comments** - If a method does not have a doc comment, or has an {@inheritDoc} tag, the Javadoc tool searches for an applicable comment using the following algorithm, which is designed to find the most specific applicable doc comment, giving preference to interfaces over superclasses:
>
> 1. Look in each directly implemented (or extended) interface in the order they appear following the word implements (or extends) in the method declaration. Use the first doc comment found for this method.
> 2. If step 1 failed to find a doc comment, recursively apply this entire algorithm to each directly implemented (or extended) interface, in the same order they were examined in step 1.
> 3. If step 2 failed to find a doc comment and this is a class other than Object (not an interface):
> 4. 1. If the superclass has a doc comment for this method, use it.
>    2. If step 3a failed to find a doc comment, recursively apply this entire algorithm to the superclass.
>
> 简而言之：会一直往继承树高层找，直到找到对应的注释或者到达 Object 类。

#### **{@link**  package.class#member  label}

一个非常强大的标记，可以链接到源码中的任何一个包、类、方法或者属性，从而使得注释可以点击跳转：

```java
Use the {@link #getComponentAt(int, int) getComponentAt} method.
```

上面的注释会产生如下效果：

```html
Use the <a href="Component.html#getComponentAt(int, int)">getComponentAt</a> method.
```

它可以使用在注释的任何地方。如果 label 不是代码（不是一个方法、类名或者属性名等，与代码无关），可以使用 `{@linkplain  package.class#member  label}`代替。

#### {@literal text}

可以避免一段文字被当做 HTML 标记或者 Javadoc 解析，比如：

```Java
{@literal A<B>C}
```

`<B>`就不会被当做加粗来解析，实际上，`{@code text}`就等于`<code>{literal text}</code>`。

#### @param

在方法的注释上很常见，但实际上也可以用来标注类，因为它可以用于标记泛型。

#### @see

这个标记会生成 "See Also" （中文：另请参阅）的注释用于指向一个引用，可以再任何可注释的对象上。它有三种用法：

1. `@see "string"`。例如`@see "The Java Programming Language"`，string 代表的是一本书或者无法通过 URL 获取的资料，必须要加上双引号才行；
2. `@see <a herf="URL#valie>label</a>">`。这里给出的是一个 URL；
3. `@see package.class#member label`。这个用法和`{@link}`很像，可以链接到源码中的任何包/类/方法/属性。label 是可选的，具体显示可以参见[How a name is displayed](http://docs.oracle.com/javase/7/docs/technotes/tools/windows/javadoc.html#shortened)。

具体写法可以参见下面的表格：

<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/@see写法.png" height="360" alt="API文档示例"/></div>

#### @since since-text

可以用于标记某个 API /特性是在哪个版本被加上的，可以用在所有可注释的元素上，比如`@since 1.5`。

#### @throws

和`@exception`一样，用于表示一个方法可能会抛出的异常，用法如下：

```java
/**
 * @throws IOException  If an input or output 
 *                      exception occurred
 */
```

实际上如果方法签名（注意：不是方法体）throw 了一个异常，而没有使用 @throws 标记，Javadoc 工具会自动生成一个没有描述的 HTML 插入到文档中。

#### @{value package.class#field}

如果`{@value}`用于修饰一个静态的 final 变量，那么就会展示这个变量的值，比如：

```java
/**
 * The value of this constant is {@value}.
 */
public static final String SCRIPT_START = "<script>"
```

否则它就可以用来表示某个类的常量值：

```java
/**
 * Evaluates the script starting with {@value #SCRIPT_START}.
 */
public String evalScript(String script) {
}
```

注意：变量必须是静态的。

## 总结

文章提到的两篇文档中还有很多注释相关的整理，尤其是关于生成文档的命令选项，如果真的需要生成 API 文档，可以仔细研究一下。这里整理的目的主要是为了日常开发所用，一般只限于代码注释而不用生成 API 文档，因此这里就不再继续研究了。

希望这篇文章能够提高一下自己和读者写注释的规范性，让注释能更好的帮助别人理解我们的代码。