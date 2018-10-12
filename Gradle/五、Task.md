---
title: Gradle系列四：Task
date: 2016-05-19 10:42:42
tags: [Gradle]
categories: Android
---

## 一、简介
__Task__，是一小段可以执行的脚本代码，它是组成 Gradle 脚本文件的重要内容。

类比起来，可以将 Gradle 脚本文件看成一个类，将 Task 看成这个类中的一个方法。
>实际上脚本文件以及 Task 确实映射到 Java 类，这一点会在[Gradle DO模型](http://www.muzileecoding.com/gradlestudy/gradle-advaced-do.html)中说明。<!--more-->

## 二、入门
当我们在一个目录下执行`gradle`命令的时候，`gradle`命令会首先在当前目录下寻找`build.gradle`文件，也就是 Gradle 脚本文件——通过这个文件，我们可以定义一个 Project 和一系列 Task。

比如我们在一个`build.gradle`文件中写入如下内容:

```Java
task hello {
    doLast {
        println 'Hello world!'
    }
}
```
然后在命令行中执行`gradle -q hello:`，就会有如下输出:

    > gradle -q hello
    Hello world!

这个例子非常简单，我们定义了一个叫做"hello"的 Task，在这个 Task 中，我们插入了一个 Action —— Action也是一小段可执行的 Groovy 脚本文件，它是组成 Task 的基本元素，以闭包的形式声明。为什么要这样实现 Task 呢？这样做主要是为了更加灵活的控制 Task 的行为。
>不了解闭包的话可以看文章: [Groovy 语法](http://www.muzileecoding.com/gradlestudy/groovy.html)。
>
>官网上说 Groovy 的 Task 和 Ant 的 Target 在概念上是一致的。有 Ant 经验的小伙伴可以类比理解。

## 三、Task模型
__Task 是一组 Action 的组合，这组 Action 存储在一个链表中，当我们执行 Task 的时候，其实是在顺序执行这组 Action。__

Task 提供了一组高级功能，可以让我们在 Action 链表前后插入 Action ，我们来看一个例子:

```Java
task hello << {
    println 'Hello A'
}
hello.doFirst {
    println 'Hello B'
}
hello.doLast {
    println 'Hello C'
}
hello.doFirst {
    println 'Hello D'
}
hello << {
    println 'Hello E'
}
```
我们执行以上代码，可以得到如下输出:

	Hello D
	Hello B
	Hello A
	Hello C
	Hello E
这个顺序是怎么产生的呢？解释一下语法就明白了:

__Task 可以执行两种方法: `doLast`和`doFirst`，其中`<<`符号和`doLast`等效。前面说了，可以想象Task内部有一个 Action 链表，`doLast`负责在这个链表的最后面插入元素，`doFirst`负责在链表的最前面插入元素。最后执行的时候，按顺序执行 Action 链表。__

对比以上代码，输出是符合预期的。

## 四、Task基础语法
>Task 内部是可以使用任何的 Groovy 语法的，关于 Groovy 语法，可以查看 [Groovy 语法](http://www.muzileecoding.com/gradlestudy/groovy.html)

### 4.1 依赖
当我们写一个类，并在其中写了诸多方法的时候，我们是需要一个地方组合这些方法实现目标功能的，这就像 Java 中的`main`函数，给整个程序一个入口，从而便于其余的类和方法按照一定的逻辑运行起来。那么在Gradle 脚本文件中，是如何组织这些 Task 的联系的呢？答案是 __依赖__ 。

```Java
task hello << {
    println 'Hello world!'
}
task intro(dependsOn: hello) << {
    println "I'm Gradle"
}
```
如代码所示，我们定义了两个 Task ，并通过`dependsOn`关键字声明`intro`依赖`hello`，执行一下`gradle -q intro`命令，输出如下:

	> gradle -q intro
	Hello world!
	I'm Gradle

虽然执行的是`intro`Task，但是实际先执行的是`hello`，因为前者依赖后者的执行。下面这种方式和上面等效:

```Java
task hello << {
    println 'Hello world!'
}
task intro() << {
    println "I'm Gradle"
}

intro.dependsOn hello
```

除了依赖已知的 Task，依赖还能以如下形式进行:

```Java
task taskX(dependsOn: 'taskY') << {
    println 'taskX'
}
task taskY << {
    println 'taskY'
}
```
一个 Task 可以依赖一个还未声明的 Task，只需要以字符串形式表示 Task 名字即可。

下面是一个比较复杂的例子:

```Java
task compile << {
    println 'compiling source'
}

task compileTest(dependsOn: compile) << {
    println 'compiling unit tests'
}

//依赖两个Task
task test(dependsOn: [compile, compileTest]) << {
    println 'running unit tests'
}

task dist(dependsOn: [compile, test]) << {
    println 'building the distribution'
}
```
在命令行中执行`gradle dist test`，可以得到如下输出:

	> gradle dist test
	:compile
	compiling source
	:compileTest
	compiling unit tests
	:test
	running unit tests
	:dist
	building the distribution
	
	BUILD SUCCESSFUL
	
	Total time: 1 secs

这里我们可以得出一个结论: __无论一个Task在一次执行中如何被依赖，它都只会执行一次__ 。

### 4.2 创建
直接使用`task`关键字声明一个Task是最常见的方式，但实际上我们还可以动态创建 Task ，比如在脚本文件中写入:

```Java
4.times { counter ->
    task "task$counter" << {
        println "I'm task number $counter"
    }
}
```
执行`gradle -q task1`，就可以得到如下输出:

	> gradle -q task1
	I'm task number 1

动态生成的 Task 也能以上面的任意一种依赖形式进行 Task 依赖声明。

### 4.3 获取
这个点其实前面已经用到了，可能因为比较符合思维，因此不一定能注意到，这里特意提一下:

```Java
task hello << {
    println 'Hello world!'
}
hello.doLast {
    println "Greetings from the $hello.name task."
}
```
如上，在往`hello`中插入 Action 的时候，直接可以使用`hello`去引用 Task ，实际上，每一个声明出来的 Task 都被当成 Gradle 脚本文件的一个属性，因此可以直接通过这种方式存取（就像类中声明的一个变量）。
>还有一种办法是通过`Project.getTasks().findByPath()`或者`Project.getTasks().getByPath()`方法拿，但是我们这里没有讲到 Domain Object，因此略过。

### 4.4 额外属性
在写一个方法的时候，我们常常会去声明一些本地变量，Task 也具备这些功能。

```Java
task myTask {
    ext.myProperty = "myValue"
}

task printTaskProperties << {
    println myTask.myProperty
}
```
在 Task 中可以声明一个变量，保存到 ext 中去，之后在关于这个 Task的任何操作中就都可以使用它了，如上，执行命令`gradle -q printTaskProperties`就有如下输出:

	> gradle -q printTaskProperties
	myValue

>细心的读者可能比较奇怪，这里的`myTask`声明比较奇怪，声明的时候没有使用`doLast`，`doFirst`，`<<`中的任何一种，在闭包里面也只是简单的一行 Groovy 代码，这是一种什么声明方式？
>
>这个其实和脚本的生命周期相关，会在[Gradle 生命周期](http://www.muzileecoding.com/gradlestudy/gradle-lifecycle.html)讲解，这段代码在配置阶段执行。这里只需要知道__以这种方式声明的 Task 中的代码会先于其余 Action 执行__就好了。

### 4.5 默认Task
前面我们执行一个 Task 的时候，都需要指定 Task 名字，其实 Gradle 脚本文件中是可以通过`defaultTasks`指定默认 Task 的:

```Java
defaultTasks 'clean', 'run'

task clean << {
    println 'Default Cleaning!'
}

task run << {
    println 'Default Running!'
}

task other << {
    println "I'm not a default task!"
}
```
现在只需要执行`gradle -q`命令，就可以得到如下输出:

	> gradle -q
	Default Cleaning!
	Default Running!

这种情况下改名了就等价于` gradle -q clean run`。

## 五、Task 高级语法
官方文档中有一章是专门为深入 Task 做讲解的: [More about Tasks](https://docs.gradle.org/current/userguide/more_about_tasks.html)。这里因为没有讲到 DO，不能很好的解释一些概念，这里就略过了，等讲完[Gradle DO模型](http://www.muzileecoding.com/gradlestudy/gradle-advaced-do.html)读者可以自行学习这一章节。