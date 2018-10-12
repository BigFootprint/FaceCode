---
title: Gradle系列五：属性
date: 2016-06-13 10:19:51
tags: [Gradle]
categories: Android
---

## 一、简介
无论是配置构建环境、控制构建过程还是记录构建中间信息，我们都需要用到属性。Gradle为我们提供了多种属性声明方式，下面详细阐述。

## 二、ext
在Gradle中最常见的定义属性的方式就是通过ext定义，比如在`build.gradle`中写入：<!--more-->

```java
ext {
	developer_name = "bigfootprint"
}

println developer_name
```
执行`gradle -q`命令，就可以看到如下输出:

```java
bigfootprint
```
在ext中声明的属性，我们是可以直接使用的，不用带任何的前缀。

>读者可能比较感兴趣，为什么这里可以通过ext来定义这样的属性？这个将在[Gradle DO模型](http://www.muzileecoding.com/gradlestudy/gradle-advaced-do.html)中阐述，这里只需要知道这种写法是可以声明属性的就好。

不仅仅可以使用 ext 为构建项目添加属性，还可以用 ext 为 Task 添加属性，比如在`build.gradle`文件中可以写入如下内容:

```java
task taskExt{
	ext{
		taskowner = "bigfootprint"
	}
}

println taskExt.taskowner
```
执行`gradle -q`可以看到如下输出:

```java
bigfootprint
```

## 三、其余方式
ext 是声明属性最方便的方式，但是 Gradle 还提供了别的方式进行属性声明。

#### 3.1 gradle.properties
在这类文件中添加的属性配置，在构建中也可以使用。该文件有两个地方可以放置：

1. 环境变量"GRADLE_USER_HOME"指向的位置，如果没有设置，默认是在"USER_HOME/.gradle"下面；
2. 项目根目录下面——对于多项目的构建来说，该文件可以存放在任意的子项目文件夹下；

如果以上两个位置都有该文件且定义有相同的属性，则 "USER_HOME" 下面的定义具有更高的优先级。

>项目中如果有多个目录的 "gradle.properties" 文件中有同样的属性，且 "USER_HOME" 下面没有该属性定义，则优先查找命令执行的当前目录下的属性文件，找不到则去父项目中寻找。

#### 3.2 命令行
使用`-D`命令行选项，可以给运行Gradle进程的JVM传递系统属性，在执行`gradle`使用`-D`选项也有同样的效果。

除了使用`-D`命令行选项，还可以使用`-P`选项。使用见后面的示例。

使用命令行传递的属性优先级最高。

#### 3.3 特殊命名的系统属性或者环境变量
Gradle 可以根据特殊命名的系统属性或者环境变量来为构建设置属性。这个特性在我们没有 CI 服务器的管理权限，同时又需要为构建设置一些不能轻易暴露的属性的时候非常有用。

这种情况下，不能使用`-P`选项，也不能更改系统级别的配置文件。正确的策略是修改 CI 服务器的构建任务，添加一个符合某种模式的变量——这个变量对系统内的普通用户不可见：

1. 环境变量的格式如下: `ORG_GRADLE_PROJECT_prop=somevalue`;
2. 系统属性的格式如下: `org.gradle.project.prop`;

>比如签名信息，就可以使用这种方式进行属性设置。

#### 3.4 设置系统属性
严格来说，这一项不属于该主题。它是通过特殊的命名规范，在`gradle.properties`文件中设置特殊格式的属性来设置系统属性——这种方式只有在项目根目录下的`gradle.properties`文件中才有效，它的格式是`systemProp.propName`。

#### 3.5 示例
比如在`gradle.properties`中设置如下属性: 

```java
gradlePropertiesProp=gradlePropertiesValue
sysProp=shouldBeOverWrittenBySysProp
envProjectProp=shouldBeOverWrittenByEnvProp
systemProp.system=systemValue
```

在`build.gradle`文件中创建如下Task:

```
task printProps << {
    println commandLineProjectProp
    println gradlePropertiesProp
    println systemProjectProp
    println envProjectProp
    println System.properties['system']
}
```

执行如下命令: `gradle -q -PcommandLineProjectProp=commandLineProjectPropValue -Dorg.gradle.project.systemProjectProp=systemPropertyValue printProps`，则可以得到如下输出:

```Java
commandLineProjectPropValue
gradlePropertiesValue
systemPropertyValue
envPropertyValue
systemValue
```

>【注意】`-P`命令直接使用键值对的形式就好，但是`-D`的属性名前面则要加上`org.gradle.project.`前缀。

## 四、属性查看
我们可以使用`gradle properties`命令查看某个project的属性，下面是在某个构建项目根目录下执行命令的输出片段:

```java
> gradle -q api:properties

------------------------------------------------------------
Project :api - The shared API for the application
------------------------------------------------------------

allprojects: [project ':api']
ant: org.gradle.api.internal.project.DefaultAntBuilder@12345
antBuilderFactory: org.gradle.api.internal.project.DefaultAntBuilderFactory@12345
artifacts: org.gradle.api.internal.artifacts.dsl.DefaultArtifactHandler_Decorated@12345
asDynamicObject: org.gradle.api.internal.ExtensibleDynamicObject@12345
baseClassLoaderScope: org.gradle.api.internal.initialization.DefaultClassLoaderScope@12345
buildDir: /home/user/gradle/samples/userguide/tutorial/projectReports/api/build
buildFile: /home/user/gradle/samples/userguide/tutorial/projectReports/api/build.gradle
```
在脚本中，如果使用了一个未定义的属性，则会抛出异常。如果构建过程依赖于一个可选的属性，那么可以使用`hasProperty('propertyName')`来判断某个属性是否被设置，未设置的情况下不要使用。
