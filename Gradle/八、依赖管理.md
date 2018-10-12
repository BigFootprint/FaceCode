---
title: Gradle系列七：依赖管理
date: 2016-05-21 16:38:10
tags: [Gradle]
categories: Android
---

## 一、简介
依赖管理分为两部分: 

1. __dependencies(依赖)__ Gradle 需要知道哪些东西是构建你项目的必需品以便于找到它们，比如我们在进行单元测试的时候就需要依赖 junit jar 包；
2. __publications(发布)__ Gradle 在构建完成项目之后，可能需要上传构建产物，比如我们打包出来的 aar 文件、jar 包；

很多项目都需要依赖别的资源以完成功能，比如以 SSH 架构搭建 J2EE 项目打出 war 包，就必须依赖 spring 等框架提供的 jar 包 —— 这些就构成一个项目的依赖。<!--more-->

Gradle 允许开发者告诉它项目构建依赖哪些资源，它就会负责帮你解决这些依赖，包括下载它们、将它们加入到构建中，其中根据一定的格式去解析依赖资源并寻找它们的过程就称为 __dependency resolution__，即__依赖解析__。
>这一点胜过 Ant，Ant 只能给出需要加载的资源的绝对或者相对路径，否则就要使用 Ivy。

通常来说，一个项目的依赖也会依赖别的资源，Gradle 也需要照顾到这些依赖——这被称为 __transitive dependencies__，即__传递依赖__。

大部分项目是为了构建出一些可以给别的项目使用的文件，将这些文件交给其余项目使用的最简单的方式就是通过发布，Gradle 会为你完成这个工作——帮助你完成构建，得到产出文件，将其上传到某个别人可以获取到的地方。发布的具体位置决定于开发者：可能只是简单的将文件拷贝到一个本地文件夹下面，或者上传到一个远程Maven仓库，或者会在同一个项目的子项目中运用这个输出，这就是 __publication__，即__发布__。

## 二、入门
来看一个脚本:

```Java
apply plugin: 'java'

repositories {
    mavenCentral()
}

dependencies {
    compile group: 'org.hibernate', name: 'hibernate-core', version: '3.6.7.Final'
    testCompile group: 'junit', name: 'junit', version: '4.+'
}
```
这个脚本做了什么呢？首先它声明`Hibernate core 3.6.7.Final`在构建过程中是必须的，当然，该文件依赖的文件也是必须的。其次该脚本声明了大于等于4.0版本的junit包是编译测试时必须的。最后，它指明这些文件应当在Maven的中央仓库去寻找符合条件的文件。

## 三、依赖与Configuration
依赖的本质是什么呢？在 Gradle 中，依赖其实是一组组的 Configuration ，Configutation 这个名字起的非常不友好，它的本意应该是 Dependency-Group ，而一个依赖本身可以看做是一个文件（当然，这个文件也可能依赖别的文件）。

首先来说明一下为什么会有分组的概念，以及从什么维度进行分组呢？举个例子就明白了：比如我开发了一个 Java 项目，在我编译的时候，我有一组需要依赖的外部资源，当我进行单元测试的时候，我还需要依赖另外一组资源，比如 Junit jar 包，Mockito jar 包，但是当我测试完成正式发布的时候，测试所依赖的资源其实是不需要打包进去的。这样一来，我们至少应该把依赖的资源分为两组：正常打包所依赖的资源和测试需要依赖的资源。在 Java 插件中，前者统一归入 "compile" 分组下，后者统一归入 "testCompile" 分组下。

简而言之，所有的依赖最后的用处不一定一致，也不一定需要打包到最后的产物中去，因此我们需要对依赖进行分组，并对组进行管理。

很多的插件都会声明自己的 configuration ，例子如下:

```Java
configurations {
    compile
}
```
使用如下方式可以对一个 Configuration 进行配置:

```Java
configurations {
    compile {
        description = 'compile classpath'
        transitive = true
    }
    runtime {
        extendsFrom compile
    }
}
configurations.compile {
    description = 'compile classpath'
}
```
configuration 的配置有很多可以讲述的地方，限于篇幅和主题，这里不展开细说，相信读者看完[Gradle DO模型](http://www.muzileecoding.com/gradlestudy/gradle-advaced-do.html)是可以自行完成相关学习的。

文档 [Dependency Management 23.5节](https://docs.gradle.org/current/userguide/dependency_management.html#sec:working_with_dependencies) 也讲述了一些 Configuration 的用法，读者可以关注。

## 四、依赖配置
依赖配置分为两部分:

1. 告诉 Gradle 依赖什么；
2. 告诉 Gradle 去哪里寻找这些依赖；

#### 4.1 依赖什么
Gradle 支持的依赖分为以下几类:

| Type                       | Description                              |
| -------------------------- | ---------------------------------------- |
| External module dependency | 对仓库中的某个外部 module 的依赖                     |
| Project dependency         | 对同项目下的某个子项目的依赖                           |
| File dependency            | 对本地文件系统中的一组文件的依赖                         |
| Client module dependency   | 也是对仓库中的某个外部 module 的依赖，不过 module 的原始信息会在 build 文件中重新声明，比如依赖的某个包的版本，是否需要对某个依赖包进行传递依赖解析，当我们需要覆写某个module的原始信息的时候，就使用这类依赖 |
| Gradle API dependency      | 依赖当前 Gradle 版本的 API，一般只有在开发 Gradle 插件或者 Task 类型的时候才会使用该类依赖 |
| Local Groovy dependency    | 依赖当前 Gradle 版本使用的 Groovy 版本，一般只有在开发 Gradle 插件或者 Task 类型的时候才会使用该类依赖 |

一般情况下，我们只会用到前三种配置，而且这种配置是极为简单的，它们的使用例子如下:

```java
dependencies {
	// External module dependency
   runtime group: 'org.springframework', name: 'spring-core', version: '2.5'
   runtime 'org.springframework:spring-core:2.5',
            'org.springframework:spring-aop:2.5'
   
   // Project dependency
   compile project(':shared')
   
   // File dependencies
   runtime files('libs/a.jar', 'libs/b.jar')
   runtime fileTree(dir: 'libs', include: '*.jar')
```
第四种不是很常见，它的例子如下:

```Java
dependencies {
    runtime module("org.codehaus.groovy:groovy:2.4.4") {
        dependency("commons-cli:commons-cli:1.0") {
        	  // 更改传递性
            transitive = false
        }
        module(group: 'org.apache.ant', name: 'ant', version: '1.9.6') {
        	  // 更改依赖包
            dependencies "org.apache.ant:ant-launcher:1.9.6@jar",
                         "org.apache.ant:ant-junit:1.9.6"
        }
    }
}
```

我们大部分时候需要指定依赖什么，但有时候我们还需要指出我们不能依赖什么，有以下两种方式:

```Java
configurations {
    compile.exclude module: 'commons'
    all*.exclude group: 'org.gradle.test.excludes', module: 'reports'
}

dependencies {
    compile("org.gradle.test.excludes:api:1.0") {
        exclude module: 'shared'
    }
}
```
一种是通过 Configuration，一种是通过 Dependency。前者可以指定某一 Confuration 还是所有的 Configuration 都排除对某一资源的依赖。

#### 4.2 去哪里寻找依赖
Gradle 从仓库中寻找依赖的文件，仓库只是一堆文件的集合，通过"group"、"name"和"version"三个维度来组织。Gradle 可以解析多种仓库格式，比如 Maven 和 Ivy，并且提供多种不同的访问仓库的方式，比如使用本地文件系统或者 HTTP。

Gradle 默认情况下不会定义任何的仓库，在使用外部仓库之前，开发者而必须至少配置一个仓库，其中一个选择就是使用 Maven 中央仓库:

```Java
repositories {
    mavenCentral()
}
```
当然也可以使用 JCenter:

```Java
repositories {
    jcenter()
}
```
除了这些默认的可以快捷指定的仓库，我们可以选择用一下方式指定一些其余的仓库:

```Java
repositories {
    maven {
        url "http://repo.mycompany.com/maven2"
    }
    ivy {
        url "http://repo.mycompany.com/repo"
    }
    ivy {
        // URL can refer to a local directory
        url "../local-repo"
    }
}
```
如上配置所示，Gradle 会按照顺序来搜寻一个依赖，直到找到依赖位置。关于Gradle支持的仓库配置方式，同样在 [Dependency Management 23.6节](https://docs.gradle.org/current/userguide/dependency_management.html#sec:repositories) 中有详细的描述，包括对于一些需要密码的仓库应当如何配置都有详细说明。

## 五、依赖版本
#### 5.1 动态版本(Dynamic Versions)和变化模块(Changing Modules)
绝大部分时候，我们可以明确写出我们想要依赖的 Module 的版本，比如:

```Java
runtime group: 'org.springframework', name: 'spring-core', version: '2.5'
```
这就是指定依赖2.5版本的 spring-core。但是有时候我们有一些别的需求：

1. __依赖某个Module最新的版本，或者某个范围内的最新版本。__ 这种最新版本的获得不是通过手动更改版本号来获得的，而是使用"2.+"这样的符号，由依赖管理系统自动获取的，如果这时候最新版本是2.5，那就使用2.5，如果有2.6，就使用2.6，而我的配置始终是"2.+"，这就是 __Dynamic Versions__ 。
2. __依赖某个版本的最新一版。__ 这听上去有点奇怪。最佳的例子其实就是 Maven 的 __SNAPSHOT__，我可以指定依赖某一个版本的 __SNAPSHOT__ 版本，比如"2.5-SNAPSHOT"，这样在这个版本上有任何的修改依赖方都可以尽快得知，同时 Module 的开发者也可以去开发2.6版本，而不会影响到2.5版本，这就是 __Changing Modules__ 。

默认来说，Gradle 缓存这两种版本 24h。

#### 5.2 版本冲突
如果在编译的过程中，我们依赖同一个 Module 的两个不同版本，就会带来版本冲突，版本冲突不但会导致编译不通过，错误的解决方式也会影响产品的功能。Gradle 提供两种依赖冲突解决策略：

1. __Newest__ 使用最新版本的依赖，这是 Gradle 的默认策略，只要依赖的包是向后兼容的，那么这个策略通常就是合适的；
2. __Fail__ 一旦版本冲突，构建就失败。这个策略要求在脚本中解决所有的冲突——可以参照[ResolutionStrategy](https://docs.gradle.org/current/dsl/org.gradle.api.artifacts.ResolutionStrategy.html)提供的解决方案；

尽管上面两项策略已经可以解决大部分的冲突问题，但是 Gradle 仍然做的更多，提供一些更精细的手段来解决版本冲突：可以将第一级或者所有的依赖（包括传递依赖）设置为`forced`来指定一个版本，例子如下:

```Java
apply plugin: 'java' //so that there are some configurations

configurations.all {
  resolutionStrategy {
    // fail eagerly on version conflict (includes transitive dependencies)
    // e.g. multiple different versions of the same dependency (group and name are equal)
    failOnVersionConflict()

    // force certain versions of dependencies (including transitive)
    //  *append new forced modules:
    force 'asm:asm-all:3.3.1', 'commons-io:commons-io:1.4'
    //  *replace existing forced modules with new ones:
    forcedModules = ['asm:asm-all:3.3.1']

    // add dependency substitution rules
    dependencySubstitution {
      substitute module('org.gradle:api') with project(':api')
      substitute project(':util') with module('org.gradle:util:3.0')
    }

    // cache dynamic versions for 10 minutes
    cacheDynamicVersionsFor 10*60, 'seconds'
    // don't cache changing modules at all
    cacheChangingModulesFor 0, 'seconds'
  }
}
```
>前面描述的`exclude`形式也是一种解决冲突的办法，我们可以在根项目下移除对某个资源所有版本的依赖，统一依赖一个最新版本的包，但这种方式比较粗暴，不一定是最佳解决方案。

## 六、高级配置
上面讲述的是一些基本的配置方案，99%的情况下，我们只需要简单的配置一下就可以使用了，但是 Gradle 对配置的支持远不止如此。

#### 6.1 依赖替换
这里举几个例子:

```Java
configurations.all {
    resolutionStrategy.eachDependency { DependencyResolveDetails details ->
        if (details.requested.group == 'org.software' && details.requested.name == 'some-library' && details.requested.version == '1.2') {
            //prefer different version which contains some necessary fixes
            details.useVersion '1.2.1'
        }
    }
}
```
以上是版本替换。

```Java
configurations.all {
    resolutionStrategy.eachDependency { DependencyResolveDetails details ->
        if (details.requested.name == 'groovy-all') {
            //prefer 'groovy' over 'groovy-all':
            details.useTarget group: details.requested.group, name: 'groovy', version: details.requested.version
        }
        if (details.requested.name == 'log4j') {
            //prefer 'log4j-over-slf4j' over 'log4j', with fixed version:
            details.useTarget "org.slf4j:log4j-over-slf4j:1.7.10"
        }
    }
}
```
以上是包替换，直接将一个依赖替换成可兼容的另外一个包。

```Java
configurations.all {
    resolutionStrategy.dependencySubstitution {
        substitute module("org.utils:api") with project(":api")
        substitute module("org.utils:util:2.5") with project(":util")
    }
}
```
以上是用 project 替换 module。

依赖管理有一组解析规则，一组替换规则，还有一组映射规则（可能有多个 module 符合条件），读者知道有这些就足够了，具体的内容可以参考[Dependency Management 23.8节](https://docs.gradle.org/current/userguide/dependency_management.html)。

#### 6.2 依赖缓存
为了加快构建速度，Gradle 会对下载下来的依赖资源进行缓存，避免每次都去下载。

Gradle 的依赖缓存分为两种类型的存储:

1. 下载下来的依赖资源文件(artifacts)，包括二进制文件，比如jar包以及 meta-data，比如 POM 文件和 Ivy 文件。文件的存储路径上包含着 SHA1 校验值，这意味着两个同名不同内容的文件不会可以同时被缓存；
2. 解析出来的 module meta-data 的二进制存储，包括解析出来的动态版本的结果，module 的描述信息，以及文件；

分开存储的主要目的是将下载下来的文件与缓存 metadata 分开。从[官网文档](https://docs.gradle.org/current/userguide/dependency_management.html#sec:dependency_cache)以及实际设备的缓存内容来看，缓存划分的维度很多，包括：Gradle 版本，Repository，Checksum 等，会在设备上留下数量庞大的缓存的信息。
> 缓存的 artifacts 是以 SHA1 校验值作为身份标识的，可以使用该值校验本地资源文件与远程是否一致。
>
> 缓存可能同时被多个进程操作，这个时候 Gradle 使用文件锁来保证一致性。

虽然默认会使用缓存，但是我们可以主动控制这一行为: 

1. __`--offline`__ 告诉 Gradle 只能使用缓存中的资源文件，不需要去检测资源文件是否已经被更新，如果本地缓存中找不到资源文件，则构建失败；
2. __`--refresh-dependencies`__ Gradle 可能因为配置问题或者拉取了错误的资源文件，导致用户需要再次去服务端拉取最新的文件，更新本地缓存，这个时候就可以在命令行中添加该选项，它会使得 Gradle 忽略本地的所有缓存，并更新相关的所有配置、文件。

另外，Gradle 还提供了 ResolutionStrategy 来更精细化的控制缓存，最常用的就是缓存过期时间，默认情况下 Gradle 缓存动态版本 (dynamic version) 的时间是24h，我们可以通过如下代码更改:

```java
configurations.all {
    resolutionStrategy.cacheDynamicVersionsFor 10, 'minutes'
}
```

也可以更改变化模块 (changing module) 的时间:

```java
configurations.all {
    resolutionStrategy.cacheChangingModulesFor 4, 'hours'
}

```

## 七、发布
依赖配置也可以用于配置产物发布，发布的产物被称为 __publication artifacts__。

事实上，插件已经定义好一个项目的产物了，也就是说，具体上传什么，不需要开发者再告诉 Gradle。
>这里提到一个之前没有提过的概念: 插件。可以认为这个就是某一类项目的构建脚本，比如专门把 J2EE 项目构建成 war 包。

剩下的就是告诉 Gradle 把构建产物发布到什么地方了，具体做法就是把一个仓库传递给`uploadArchives`Task:

```Java
uploadArchives {
    repositories {
        ivy {
            credentials {
                username "username"
                password "pw"
            }
            url "http://repo.mycompany.com"
        }
    }
}
```
配置好之后，执行`gradle uploadArchives`命令就可以上传发布产物了—— Gradle 会为你负责其余的一切，比如添加 Maven 需要的 pom.xml文件。
>发布到 Maven 仓库需要使用 Maven 插件，具体见[Maven Publishing](https://docs.gradle.org/current/userguide/publishing_maven.html)。