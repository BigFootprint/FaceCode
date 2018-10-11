---
title: Log4j
date: 2016-11-30 14:43:48
tags: [Java, 日志]
categories: 后端
---

## 简介
简单来说，[Log4j](https://logging.apache.org/log4j/1.2/manual.html) 是一个专门的日志工具，可以可靠、快速的进行日志输出。

> Inserting log statements into code is a low-tech method for debugging it. It may also be the only way because debuggers are not always available or applicable. This is usually the case for multithreaded applications and distributed applications at large.

按照官网的说法，日志在某些情况是唯一进行调试的手段，比如多线程状态下，或者是分布式应用中。按照我的理解，日志有以下好处：

1）我们可以选择输出我们关注的信息到文件或者我们任何想要的地方，靠谱快速；  
2）日志系统可以做快照，在发生异常情况时，可以记录此时的上下文环境；

<!--more-->Log4j 主要包含三个方面的东西：Logger、Appender 和 Layout，这三个部分定义了 Log 的内容、格式、级别以及输出目标。开始学习前，先给一个例子直观感受一下：

```java
Logger  logger = Logger.getLogger("com.foo");//为 Logger 取一个名字
logger.setLevel(Level.INFO);// 设置 Logger 的级别
logger.warn("Low fuel level."); // 打印一个warn级别的日志
logger.debug("Starting search for nearest gas station.");// 打印一个debug级别的日志，但是这里实际不会打印，下面会解释原因
```

## Logger 的继承

### 继承和级别

Logger 根据 `System.out.println`的设计，也对日志进行空间划分，也就是 Category 的概念，简单来说，日志的输出是分组，可以把某些日志分为一组，从而可以在组的层面上控制日志的输出。Category 的概念在 Log4j 1.2之后就被取代了，这就是 Logger 类。官网上关于分组的说法是：

> A logger is said to be an *ancestor* of another logger if its name followed by a dot is a prefix of the*descendant* logger name. A logger is said to be a *parent* of a *child* logger if there are no ancestors between itself and the descendant logger.

这个和包以及子包的概念一样，名字叫做"java.util"的 Logger 是名字叫做"java"的 Logger 的子，是名字叫做"java.util.Vector"的 Logger 的父，是名字叫做"java.util.X.Y"的 Logger 的祖先。__换句话说：Logger类通过名字（详见前面例子）来标记身份，名字之间如果按照规范有父子关系，则称作 Logger 之间有继承关系__。

同时 Logger 也有一个 root logger，它比较特殊：

1）这个 Logger 总是存在的；

2）不能通过名字获取实例，只能通过方法`Logger.getRootLogger`获取；

下面是 Logger 常用的一些方法：

```java
package org.apache.log4j;

  public class Logger {

    // Creation & retrieval methods:
    public static Logger getRootLogger();
    public static Logger getLogger(String name);

    // printing methods:
    public void trace(Object message);
    public void debug(Object message);
    public void info(Object message);
    public void warn(Object message);
    public void error(Object message);
    public void fatal(Object message);

    // generic printing method:
    public void log(Level l, Object message);
}
```

从 API 就可以看出，日志是有级别的，官方的级别定义在 [Level](https://logging.apache.org/log4j/1.2/apidocs/org/apache/log4j/Level.html) 类中，如果一个 Logger 没有被赋予日志级别，那它就会从最近的祖先那里继承一个级别。

> 这里可能有点模糊了，日志的级别是区分大小的，比如常见的几种级别的大小比较就是：DEBUG < INFO < WARN < ERROR < FATAL。
>
> Logger 本身也是有日志级别的，它表示一个 Logger 可以输出什么级别的日志，比如 Logger A 定义的级别是 warn，那么使用这个 Logger 输出 warn 级别的日志就会被过来掉，而 warn 和更高级别的 error 日志就可以被输出。因此日志级别对于 Logger 来说相当于一个过滤器。

为了保证所有的 Logger 都会有一个级别，root logger 总是会有一个默认级别。如下表所示：

| Logger name | Assigned level | Inherited level |
| :---------: | :------------: | :-------------: |
|    root     |     Proof      |      Proof      |
|      X      |      none      |      Proof      |
|     X.Y     |      Pxy       |       Pxy       |
|    X.Y.Z    |      none      |       Pxy       |

Logger X 没有设置日志级别，则从 root logger 继承一个，因此也是 Proof，而 Logger X.Y 设置了日志级别为 Pxy，则最终的日志级别也为 Pxy，Logger X.Y.Z 没有设置日志级别，则默认从最近的祖先，也就是 Logger X.Y 里继承一个，即 Pxy。

通过同一个名字获取的 Logger 总是指向同一个 Logger 实例。通常在开发中我们会使用当前类的 class 来获取一个 Logger 。比如：

`private static final Logger logger = Logger.getLogger(MonitorController.class);`

这是一种简便有效的获取 Logger 的方法，当然 Logger4j 并不排斥你用别的方式为 Logger 命名。父 Logger 和子 Logger 可以以任意的顺序进行配置和初始化，并不影响它们的继承关系。

### 小结

通过前面的介绍，我们已经知道：

1）日志本身是有级别的，级别也是有序的；

2）Logger 也是有级别的，只有当打印的日志级别大于等于 Logger 的日志级别，才会输出日志；

3）Logger 的日志级别可以继承；

由此可以看出，日志的有序级别是 Log4j 控制日志输出的一个核心概念。

> Log4j建议只使用四个级别，优先级从高到低分别是ERROR、WARN、INFO、DEBUG。

## Appenders和Layouts

在输出日志的时候，除了需要根据级别来控制日志输出，还需要控制日志输出的目的地和格式，这就是Appender和Layout的作用。

### Appender

一个 Logger 可以配置多个 Appender，简而言之，Logger 可以把同一份日志输出到多个地方，比如console、文件。一个有效的日志请求（指的是级别大于等于 Logger 的日志级别）不但会被当前 Logger 的所有 Appender 处理，而且会被 Logger 的祖先 Logger 所配置的所有 Appender 处理，比如 root logger 配置了一个 console Appender，则所有的 Appender 都至少会在控制台输出日志。也就是说 __Appender 也具有继承性__，但不同于日志级别的继承，Appender 的继承是可以控制的：只需要将 Logger 的 additivity 属性设置为false，该 Logger 将不会继承父 Logger 的 Appender。

### Layout

Logger 允许为 Appender 设置一个 Layout 来控制日志的输出格式。