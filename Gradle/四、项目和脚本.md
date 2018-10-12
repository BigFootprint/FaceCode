---
title: Gradle系列三：项目和脚本
date: 2016-06-13 10:19:51
tags: [Gradle]
categories: Android
---

## 一、初见Gradle脚本
读者可以在任意目录下面建立一个`build.gradle`文件，将以下内容拷贝进去:

```Java
task compile << {
    println 'compiling source'
}
```
在本目录下面执行命令`gradle -q compile`，就可以看到命令行输出"compiling source"字符:<!--more-->

	>gradle -q compile
	compiling source

`build.gradle`就是一个 Gradle 脚本文件，可以独立运行。虽然例子和构建没有任何关系，但是这就是一个非常简单的，由单个脚本文件组成的构建项目。

>这里的 __构建项目__ 可能有歧义：Gradle 其实是内嵌在开发项目中的一组文件。Gradle 构建工具通过读取这些文件，了解开发者对项目的配置，完成项目的构建，这里讲 Gradle 的这组文件称为“构建项目”。

## 二、Project 和 Task
我们在脚本文件中写入的内容是什么意思呢？它是一个 __Task__，它的语法以及用法会在[Gradle Task](http://www.muzileecoding.com/gradlestudy/gradle-task.html)中细说，这里只需要知道它是一个类似方法的代码片段就 OK 了。

借助这个脚本文件，我们先重点看一下 Gradle 相关的几个重要概念：

<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/Gradle项目概念模型.png" height="400" alt="Gradle项目概念模型"/></div>

上图展示了一个 Gradle 构建项目的核心概念模型（其余概念将在 [Gradle DO 模型](http://www.muzileecoding.com/gradlestudy/gradle-advaced-do.html)中描述）。这个图中有三个重要的概念：__Project__、__Task__ 和 __Action__。Gradle 的构建核心就是基于 Project 和 Task 的（Action 在讲述 Task 的时候会详细解释，这里不影响理解，略过）。

每一个 Gradle 构建项目包含一个或者多个 Project —— 这个 Project 是一个很不明确的概念，它和一个`build.gradle`文件一一对应，它可以什么都不做，比如我们前面举的例子，只是打印一个字符串，也可以完成类似编译一个 Java 应用——打 Jar 包——上传到 Maven 仓库这样复杂的操作，这完全取决于你在`build.gradle`文件中实现的功能。

每一个 Project 又由一个或者多个 Task 组成，前面的例子就是由一个叫做`compile`的 Task 组成的，一个 Task 代表一小段可以执行的构建代码，在实际的应用中，一个 Task 可能完成编译一组类、打一个 Jar 包、生成 Javadoc 文档、上传归档文件到仓库等功能。

>有 Android 经验的小伙伴可以将 Project 映射到一个 Android 项目，子 Project 映射到 Module 去理解。
>
>如果把Project看成是一个类，而Task看成是一个个方法，理解起来会更加形象。

## 三、多项目文件结构
前面举得例子非常简单，实际生产环境中多 Project 形式的项目是非常普遍的，比如 Android 中一个主项目往往有多个 Module ，最后的 APK 打包其实就是将这些 Module 的产物打包在一起的过程。虽然项目的嵌套形式不一定一样，但是它的 Gradle 配置应当具备以下特征：

1. 在项目根目录/主目录下面有一个`settings.gradle`文件，同时也有一个`build.gradle`文件 —— 对应着上图中最顶层的 Project；
2. 每一个子项目也有自己的`build.gradle`文件；

`settings.gradle`是用于告诉 Gradle：构建项目由哪些开发项目组成，它们是如何组织的。在这样一个复杂的目录结构下面，我们可以直接使用`gradle projects`命令来查看项目结构。

在 "GRADLE_HOME/samples/java" 目录下面，找到 "multiproject" 目录，在该目录下面执行`gradle -q projects`，就会得到如下输出:

```Java
> gradle -q projects

------------------------------------------------------------
Root project
------------------------------------------------------------

Root project 'multiproject'
+--- Project ':api'
+--- Project ':services'
|    +--- Project ':services:shared'
|    \--- Project ':services:webservice'
\--- Project ':shared'

To see a list of the tasks of a project, run gradle <project-path>:tasks
For example, try running gradle :api:tasks
```
这说明:

1. `multiproject`项目包含三个直系子Project：`api`, `services`和`shared`；
2. 子项目`services`又包含`shared`和`webservice`两个子项目；

对比实际文件目录结构，很容易理解这些项目的组织过程。而它的`settings.gradle`文件如下:

```java
include "shared", "api", "services:webservice", "services:shared"
```

>"multiproject" 下面其实还有一个`buildSrc`目录，里面也有`build.gradle`文件，但是并没有在命令行中打印出来，这是因为`settings.gradle`文件中没有配置该项目。

根目录下面的`build.gradle`文件通常会有一些子项目共有的配置，比如共享的依赖，在这个文件里面，也可以对子项目进行配置，具体可以查看[组织构建逻辑](https://docs.gradle.org/current/userguide/organizing_build_logic.html)。
>`build.gradle`文件不一定要叫这个名字，可以以子模块的名字命名。

## 四、多项目构建
上面已经看到一个多项目构建是如何配置的了，下面进一步阐述。

### 4.1 项目定位
多项目构建是以一棵树的形式展现的，树中的每一个节点都代表着一个 project（是不是很类似前面的结构图？），每一个 project 都有一个代表自己位置的路径。大部分情况下，这个位置和项目所在文件系统中的相对位置是一致的，然而，这是可配置的。 project 树是根据 "settings.gradle" 文件来创建的，这个文件默认是放在根 project 目录下面，但是实际上根目录的位置也可以在该文件中重新定义。

换句话说， __只要有 "settings.gradle" 文件，只要可以定义出一个 project 树出来就好__ 。

### 4.2 构建 Project 树
在 settings.gradle 文件中，有一组方法可以用于构建 project 树。主要有 "Hierarchical layouts" 和 "flat physical layouts" 。

#### 4.2.1 Hierarchical layout
举个例子，在 "settings.gradle" 中写入如下脚本:

```java
include 'project1', 'project2:child', 'project3:child1'
```
`include`方法使用路径作为参数，路径是根据相对路径得出的，只不过路径分隔符变成了`:`。我们只需要标注出叶子节点项目即可，也就是说，如果我们包含项目 'services:hotels:api'，那构建的时候就会创建三个 projects: 'services', 'services:hotels' 和 'services:hotels:api'。

#### 4.2.2 Flat physical layouts
举个例子，在 "settings.gradle" 中写入如下脚本:

```java
includeFlat 'project3', 'project4'
```
`includeFlat`方法以目录作为参数，这些目录必须是根 project 的兄弟目录，也就是说和项目根目录平级。在多 project 树中，这些目录的位置是作为根 project 的子项目存在。

### 4.3 项目信息修改
多项目构建树是由 "project descriptors" 构成的，可以在 settings 文件中随时更改这些 descriptors，包括更改 project 名称，project 目录，以及 project 构建名称。例子如下:

```java
//获取属性
println rootProject.name
println project(':projectA').name

//更改属性
rootProject.name = 'main'
project(':projectA').projectDir = new File(settingsDir, '../my-project-a')
project(':projectA').buildFileName = 'projectA.gradle'
```
>初始化脚本的代理对象 Settings 的`project`方法返回的是 ProjectDescriptor 对象。

## 五、总结
以上就是一些关于项目结构以及重要文件配置的说明。