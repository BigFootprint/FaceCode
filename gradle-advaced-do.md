---
title: Gradle系列九：DO模型
date: 2016-06-13 10:21:03
tags: [Gradle]
categories: Android
---

## 一、Domain Object
Gradle 脚本实际上是配置脚本，在脚本执行的时候，Gradle会配置一些特定类型的对象，这些对象就被称为脚本的 delegate 对象，也是构建领域的领域对象，下文简称 __DO__。

>要理解delegate，可以查看文章[Groovy 语法](http://www.muzileecoding.com/gradlestudy/groovy.html)。

__DO 的意义就在于脚本可以使用被代理的对象的方法__——这一点非常重要，是脚本中可调用方法的重要来源。<!--more-->

## 二、顶级DO
Gradle 有三种脚本类型，如下表所示:

<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/顶级DO.png" width="480" alt="顶级DO"/></div>

我们最常配置的脚本文件就是`build.gradle`和`settings.gradle`，这两类脚本文件会分别被 delegate 到 Project 实例和 Settings ，。了【实例。

>"delegate到"的意思就是脚本中的配置以及方法调用都会被转移到代理对象上。

但是实际上每一个脚本文件最直接对应的 DO 是 Script 对象，Script 对象根据脚本文件类型来判断将脚本中的配置代理到什么类型的对象上去。

### 2.1 DO创建过程
这里简单描述一下一个脚本执行过程中相关顶级 DO 的创建过程:

1. 为构建创建一个 Settings 对象；
2. 搜寻 settings.gradle 脚本文件，如果有，通过该脚本配置Settings对象；
3. 根据配置好的 settings.gradle 对象创建 Project 树；
4. 根据每一个 build.gradle 文件配置对应的 Project 对象；

这里省略了 Script 对象和 Gradle 对象的创建，前者在扫描到脚本文件的时候就被创建了，后者则是在构建过程初始化的时候被创建。

## 三、Project进阶
这里讲到 DO 之后，就建立了 Gradle 这门 DSL 语言与 Java 类的关联，从这个角度，可以进一步解释很多的问题。

### 3.1 方法和属性
这里以一个 Android 项目的`build.gradle`文件内容为例:

```java
buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:2.0.0'

        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}

allprojects {
    repositories {
        jcenter()
    }
}
```
从 Groovy 的角度看，这样的语法就是一个方法调用，应该是一个类似如下定义的方法:

```java
def void buildscript(Closure closure){
	//method body
}

def void allprojects(Closure closure){
	//method body
}
```
那么这个方法是来自哪里的呢？实际上就是来自 Project，打开[Project的API文档](https://docs.gradle.org/current/javadoc/org/gradle/api/Project.html)，就可以看到这两个方法:  

<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/project-api.png" width="4800" alt="顶级DO"/></div>

属性也是如此。

### 3.2 搜寻路径
在实际的脚本文件以及来自网上的很多脚本案例中，除了以上可以从 Project 的 API 文档中查询到的方法之外，还有一些方法是无法找到的，比如之前介绍的很重要的声明属性的 ext，它的形式也表明它是一个方法，但是 Project 中确实没有这个方法。

>在实践中我使用反射来寻找这个方法，但是并没有找到：它不属于 Project 或者 Task 类。

那么这个方法来自哪里呢？当我们在脚本中调用一个方法的时候，它是有一个搜索路径的:

1. Project 本身
2. build file
3. 添加进来的 extension 产生的同名方法
4. convention methods
5. 添加进来的 Task 产生的同名方法
6. 父 Project 的方法，直到根 Project 为止

这里是官网上列出的六个搜寻点，但__在实践中发现，最初的搜索点应该是 Script 对象，接着才是 Project 本身__。

对于属性，也有类似的搜索路径:

1. Project 本身
2. extra property
3. 添加 extensions 产生的同名属性
4. convention 添加的属性
5. 添加 Task 产生的同名属性
6. 从父 Project 继承的 extra 属性和 convention 属性，直到根项目

鉴于两种原因，导致方法和属性的定位非常不容易:

1. Groovy 可以轻松地动态创建类，并为类添加属性和方法。举个栗子，虽然文档中写明`build.gradle`中的配置会代理到 Project 对象，但在实际运行的时候，Gradle 创建了一个 Project 的 Default 实现类来作为代理对象，这个类的具体实现在文档中是不可查的（但是可以利用反射进行探索）；
2. 如搜寻路径中写明的，Gradle 允许通过 extension、convention等多种手段来添加方法和属性，这会随着 Gradle 对 DSL 的定义和实现、插件的使用等变化，这部分也很难查；

因此，除了阅读 API 文档之外，还需要阅读 User Guide 等文档来寻找一些特殊的用法。

### 3.2 Container
Project 允许配置很多的内容，比如声明一组 Task、创建一组扩展、定义一组配置等等，在 Gradle 中，有一组 Container 来管理这些元素。

#### 3.2.1 TaskContainer
负责管理所有的 Task 实例，通过 Project 的 task 方法即可创建 Task 实例放入其中。

[Project](https://docs.gradle.org/current/javadoc/org/gradle/api/Project.html) 类有一个方法`getTasks()`，它返回的就是[TaskContainer](https://docs.gradle.org/current/javadoc/org/gradle/api/Project.html)对象，读者可以自行查阅这个类，可以发现很多有关 Task 的新玩法。

#### 3.2.2 ConfigurationContainer
1. A Configuration represents a group of artifacts and their dependencies.
2. Configuration is an instance of a FileCollection that contains all dependencies but not artifacts. If you want to refer to the artifacts declared in this configuration please use getArtifacts() or getAllArtifacts().

关于 Configuration，在[Gradle 依赖管理](http://www.muzileecoding.com/gradlestudy/gradle-advaced-dependency-management.html)有描述。另外在官方例子中也有一些与此相关，比如[Definition of a configuration](https://docs.gradle.org/current/userguide/dependency_management.html#defineConfiguration)。读者可以自行查阅。

#### 3.2.3 ExtensionContainer
与其余两个 Container 不同，ExtensionContainer 是一个顶级接口，无父接口。特性是会在 Project 中创建同名的属性和方法。重要的子接口 —— Convention。

关于 Extension 的用法，在[Writing Custom Plugins](https://docs.gradle.org/current/userguide/custom_plugins.html)一章中正好有例子：

```Java
apply plugin: GreetingPlugin

greeting {
    message = 'Hi'
    greeter = 'Gradle'
}

class GreetingPlugin implements Plugin<Project> {
    void apply(Project project) {
    	// 可以创建扩展，从而在外部获得配置
        project.extensions.create("greeting", GreetingPluginExtension)
        project.task('hello') << {
            println "${project.greeting.message} from ${project.greeting.greeter}"
        }
    }
}

class GreetingPluginExtension {
    String message
    String greeter
}
```
project 有一个 extensions 属性（也有一个`	getExtensions()`），它的类型就是 ExtensionContainer ，同样，读者也能在这里找到各种玩法。这其中是有一个非常特别的方法的: `getExtraProperties()`，它返回的对象类型为[ExtraPropertiesExtension](https://docs.gradle.org/current/javadoc/org/gradle/api/plugins/ExtraPropertiesExtension.html)，前面说了`ext()`方法不可查，但是读者查看这个类的相关文档之后，会有意外收获的。

它的子类 Convention 更加牛逼，官方解释如下：
>A Convention manages a set of convention objects. When you add a convention object to a Convention, and the properties and methods of the convention object become available as properties and methods of the object which the convention is associated to. A convention object is simply a POJO or POGO. Usually, a Convention is used by plugins to extend a Project or a Task.

这个目前在官方文档中没有发现相关使用指导(坑)，但是网上有[大牛](http://aetomation.aestasit.com/posts/injecting-utility-methods-in-gradle-projects-using-plugin-conventions/#.V3e1TZN95TY)探索出了用法：

```Java
// This line will enable our plugin for
// the build that imports this script.
apply plugin: UtilitiesPlugin

/**
 * This is a plugin.
 */
class UtilitiesPlugin implements Plugin {
	def void apply(Project project) {
		project.convention.plugins.utilities =
			new UtilitiesPluginConvention(project)
	}
}

/**
 * Plugin convention class.
 */
class UtilitiesPluginConvention {
	private Project project

	public UtilitiesConvention(Project project)  {
		this.project = project
	}

	/* DEFINE YOUR UTILITY METHODS HERE */
	def printMessage() {
		println "Message from " + project.name
	}
	...
}
```
这个使用完全符合 Convention 的接口定义。

## 四、DO总结
根据以上分析，Gradle 的 DO 树如下:

<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/gradle-do.png" width="480" alt="Gradle DO 树"/></div>

这些 DO 共同搭建了一个构建框架，开发者通过在脚本中对这些 DO 做出配置来控制构建过程。当然这里只是列出了一些重要的 DO，在 官网的 Java Doc 中，还有许许多多的 DO 有待探索，但这里构建的是整个 Gradle DO 基础，建立于此，读者可以自行学习其余的用法。