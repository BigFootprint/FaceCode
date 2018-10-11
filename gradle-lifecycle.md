---
title: Gradle系列八：生命周期
date: 2016-06-18 12:20:02
tags: [Gradle]
categories: Android
---

Gradle 的核心是一门以依赖为基础的 DSL。开发者通过定义 Task 以及 Task 之间的依赖来完成任务。Gradle 确认会根据依赖关系来执行 Task，并且每一个 Task 都只会被执行一次，Gradle 会在执行 Task 之前先构建一个依赖关系图。
>构建脚本就是用于配置这个依赖关系图，因此构建脚本实际上是配置脚本。

<!--more-->
## 一、构建步骤
Gradle 构建有三个步骤。

| 步骤                 | 说明                                       |
| ------------------ | ---------------------------------------- |
| __Initialization__ | Gradle 支持单项目或者多项目构建，在该阶段，Gradle 确认哪些项目会参与构建，然后为每一个项目创建 Project 对象 |
| __Configuration__  | 这个阶段就是配置 Initialization 阶段创建的 Project 对象，所有的配置脚本都会被执行 |
| __Execution__      | 这个阶段 Gradle 会确认哪些在 Configuration 阶段创建和配置的 Task 会被执行，哪些 Task 会被执行取决于`gradle`命令的参数以及当前的目录，确认之后便会执行 |

## 二、生命周期示例
如果我们在一个目录下面执行`gradle init`命令，那么标准的 gradle 构建项目下是会创建 "settings.gradle" 文件的。对于单项目构建，这个文件是可选的，但对于多项目构建，这个文件是必须的。这个文件会在 initialization 阶段被执行，这个文件会根据命名规范列出需要参与构建的项目。

在一个 "settings.gradle" 文件中写入如下内容:

```java
println 'This is executed during the initialization phase.'
```

在 "build.gradle" 文件中写入如下内容:

```java
println 'This is executed during the configuration phase.'

task configured {
    println 'This is also executed during the configuration phase.'
}

task test << {
    println 'This is executed during the execution phase.'
}

task testBoth {
    doFirst {
      println 'This is executed first during the execution phase.'
    }
    doLast {
      println 'This is executed last during the execution phase.'
    }
    println 'This is executed during the configuration phase as well.'
}
```
运行命令`gradle test testBoth`，就会得到如下输出:

```java
> gradle test testBoth
This is executed during the initialization phase.
This is executed during the configuration phase.
This is also executed during the configuration phase.
This is executed during the configuration phase as well.
:test
This is executed during the execution phase.
:testBoth
This is executed first during the execution phase.
This is executed last during the execution phase.

BUILD SUCCESSFUL

Total time: 1 secs
```
根据文案提示，具体的动作发生的生命周期阶段都是很清楚的。

>对于构建脚本(即 "build.gradle")来说，属性获取和方法调用都被代理到 Project 对象，而同样的，在 "settings.gradle" 中进行的属性获取和方法调用都会被代理到 Settings对象。
>
>这一点会在在[Gradle DO模型](http://www.muzileecoding.com/gradlestudy/gradle-advaced-do.html)中详细解释。

## 三、Initialization
Gradle 如何知道自己在构建的是一个单项目构建项目还是一个项目构建项目呢？如果它是在一个有 "settings.gradle" 的文件夹下面运行，事情就变得很简单，但是 Gradle 允许在任何一个子项目下面执行命令，所以这涉及到一个寻找 "build.gradle" 文件的过程，搜寻路径如下:

1. It looks in a directory called master which has the same nesting level as the current dir.
2. If not found yet, it searches parent directories.
3. If not found yet, the build is executed as a single project build.
4. If a settings.gradle file is found, Gradle checks if the current project is part of the multiproject hierarchy defined in the found settings.gradle file. If not, the build is executed as a single project build. Otherwise a multiproject build is executed.

>这个搜索过程很奇怪，因为实际上 Gradle 的构建项目并不一定只嵌套两层。

为什么要进行这样的搜索呢？Gradle 在执行构建的时候是需要确认是不是在多项目下的子项目中。如果是在子项目中，那么只有子项目以及它们依赖的子项目会被构建，但是一些配置以及依赖关系是需要从多项目构建配置中读取的，因此还是需要确认究竟是不是多项目。开发者可以使用`-u`命令来告诉 Gradle 不要进行搜索，那么当前项目就会被当做单项目进行构建，如果当前项目包含 "settings.gradle" 文件，则`-u`无效。

另外，以上的搜索方法只适合于 physical hierarchical 或者 flat layout 类型的构建项目。
>不了解这两个概念的，可以在文章[Gradle 项目和脚本](http://www.muzileecoding.com/gradlestudy/gradle-project-and-script.html)中查询到。

## 四、生命周期
Gradle 的构建分为三个阶段，在这些阶段之间，开发者是可以收到一些回调的，回调分为两类: 

1. 实现某一类型的监听接口；
2. 提供一个回调闭包以供调用；

下面都是以闭包的形式举得例子，并且也不全面，建议直接阅读 API 文档获取更多的方法。

#### 4.1 Project evaluation
在一个 Project 被 evaluate(这个词始终没有找到好的翻译，实践中它是指向 Configuration 之后，Execution 之前的一个阶段，所以应该是指Project被配置好) 之后，会收到一个回调，通过这个回调可以添加额外的配置，或者打一些 Log:

```Java
allprojects {
    afterEvaluate { project ->
        if (project.hasTests) {
            println "Adding test task to $project"
            project.task('test') << {
                println "Running tests for $project"
            }
        }
    }
}
```
如上，在有`hasTests`属相的Project中创建一个名为`test`的 Task。这就是通过提供闭包的形式在生命周期的某个节点上添加动作的效果。

但有时候 evaluate 会失败，如果这个时候需要收到消息，就应该使用另外一个回调:

```Java
gradle.afterProject {project, projectState ->
    if (projectState.failure) {
        println "Evaluation of $project FAILED"
    } else {
        println "Evaluation of $project succeeded"
    }
}
```

也可以对 Gradle 对象添加 `ProjectEvaluationListener` 来监听这些事件。

#### 4.2 Task creation
在一个 Task 被添加到 Project 的时候（也就是创建的时候），也可以收到回调，这可以被用来设置一些默认值或者添加行为:

```Java
tasks.whenTaskAdded { task ->
    task.ext.srcDir = 'src/main/java'
}

task a

println "source dir is $a.srcDir"
```
当然也可以通过在`TaskContainer`中添加 Action 来达到目的。

##### 4.2.1 Task execution graph ready
前面说了，开发者定义完 Task 以及它们的依赖关系之后，在执行前会生成 Task execution graph。在生成完这张图（Configuration 阶段完成）之后也会收到一个回调，利用这个回调，我们可以检查是否有我们需要的 Task，以及整个 Graph 的状态是否符合我们的预期，比如:

```Java
task distribution << {
    println "We build the zip with version=$version"
}

task release(dependsOn: 'distribution') << {
    println 'We release now'
}

gradle.taskGraph.whenReady {taskGraph ->
    if (taskGraph.hasTask(release)) {
        version = '1.0'
    } else {
        version = '1.0-SNAPSHOT'
    }
}
```
执行`gradle -q distribution`之后，可以得到如下输出:

	>gradle -q distribution
	We build the zip with version=1.0-SNAPSHOT

执行`gradle -q release`之后，输出则如下:

	>gradle -q release
	We build the zip with version=1.0
	We release now

这样我们就可以在一个 Task 被执行之前根据环境对它进行控制。

另外也可以通过添加`TaskExecutionGraphListener`监听到`TaskExecutionGraph`来接收事件。

##### 4.2.2 Task execution
在 Task 执行前后也可以接受回调事件，并且无论是否执行成功都能收到:

```Java
task ok

task broken(dependsOn: ok) << {
    throw new RuntimeException('broken')
}

gradle.taskGraph.beforeTask { Task task ->
    println "executing $task ..."
}

gradle.taskGraph.afterTask { Task task, TaskState state ->
    if (state.failure) {
        println "FAILED"
    }
    else {
        println "done"
    }
}
```
执行一下`gradle -q broken`，可以得到如下输出:

	> gradle -q broken
	executing task ':ok' ...
	done
	executing task ':broken' ...
	FAILED

## 五、总结
Gradle 执行构建的三个阶段非常清楚，提供了多种方式让开发者有机会监听、更改构建状态，这也是 Gradle 强大、灵活的原因之一。

除了上面提到的这些方法，监听声明周期的方法有很多，主要集中在`TaskExecutionGraph`、`Gradle`和`Project`三个类中，全部列出来意义不大，建议读者在有需求的时候可以将相关方法都研究一下，注释还是比较全面的。
