---
title: Gradle系列六：命令行
date: 2016-06-18 10:39:50
tags: [Gradle]
categories: Android
---

当看完 Task 入门，我们已经可以写一些简单的脚本了。下面来系统学习一下 Gradle 命令行相关内容。

以下面脚本为例：<!--more-->

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

依赖关系如下:
![脚本依赖关系](https://docs.gradle.org/current/userguide/img/commandLineTutorialTasks.png)

### 1. Excluding tasks
可以使用`-x`选项剔除一个 Task 的执行，比如针对以上脚本，我们运行`gradle dist -x test`，会得到如下输出:

	> gradle dist -x test
	:compile
	compiling source
	:dist
	building the distribution
	
	BUILD SUCCESSFUL
	
	Total time: 1 secs

只有 compile 和 dist 两个任务执行了，test 以及被 test 依赖的 compileTest 都没有执行。另外虽然 test 没有执行，但是依赖 test 的 dist 还是执行完成了，这说明`dependOn`其实并不是一个强依赖。

### 2. Task name abbreviation
执行Task的时候可以使用 Task 名字缩写，比如执行 dist 可以使用`gradle -q di`，执行 compileTest 可以使用`gradle compTest`或者`gradle cT`。

### 3. Selecting which build to execute
一般我们执行`gradle`命令的时候，Gradle 会在当前目录下寻找脚本，我们可以使用`-b`选项来指定其余路径下面的脚本文件，比如：

```Java
gradle -q -b subdir/myproject.gradle hello
```
一旦使用了`-b`选项，那么 "settings.gradle" 文件将不会被使用。对于多项目脚本，可以使用`-p`指定目录：

```Java
gradle -q -p subdir hello
```

### 4. 获取脚本文件信息
通过`gradle tasks`命令能看到当前可执行的所有 Task。每一个 Task 后面都会跟上 description 用于描述 Task 的具体功能（等讲到脚本的生命周期的时候，会提到如何为一个 Task 添加描述）。在我的脚本中执行`gradle -q tasks`，可以获得如下输出:

```Java
------------------------------------------------------------
All tasks runnable from root project
------------------------------------------------------------

Build Setup tasks
-----------------
init - Initializes a new Gradle build. [incubating]
wrapper - Generates Gradle wrapper files. [incubating]

Help tasks
----------
components - Displays the components produced by root project 'gradle'. [incubating]
dependencies - Displays all dependencies declared in root project 'gradle'.
dependencyInsight - Displays the insight into a specific dependency in root project 'gradle'.
help - Displays a help message.
projects - Displays the sub-projects of root project 'gradle'.
properties - Displays the properties of root project 'gradle'.
tasks - Displays the tasks runnable from root project 'gradle'.

To see all tasks and more detail, run gradle tasks --all

To see more detail about a task, run gradle help --task <task>
```
可以打印出所有可执行的 Task，包括子 Project 中的 Task。每一个 Task 后面都跟着这个 Task 的功能描述。

其中`Help tasks`中的 Task 可以获取关于本脚本的很多信息，比如`gradle dependencies`可以列出根项目所依赖的项目。