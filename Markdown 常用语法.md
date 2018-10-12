---
title: Markdown常用语法
date: 2015-10-17 10:58:26
tags: [工具, 效率, MarkDown]
categories: 工具
---

Markdown是一种标记语言，可以方便的转化为HTML，比如本文就是以Markdown编写，最终转化为网页展现在读者面前。这种标记语言相对于HTML来说功能有限，但是对于专心写文章的人来说，却非常足够(Markdown兼容HTML)。目前Github，StackOverFlow以及很多的博客网站、静态博客系统（Octopress, Hexo...）都支持Markdown语言，因此对于有兴趣的读者来说，学习markdown不仅成本小，而且实用性很大。<!--more-->

本文主要是是整理一些我常用到的且不容易记住的Markdown语法。这里先推荐一个Mac下的Markdown写作工具：Mou。而本文则是在另外一个编辑器"MacDown"下编写完成，MacDown基于Mou，两个我都用过，都挺不错。Windows下也有相应的编辑器，读者可以安装一枚。

下面内容随时更新。

###列表
无须列表前面加上*，+，-都可以表示，比如：

	*  AAAA
	*  BBBB
	*  CCCC
展示出来如下：

*  AAAA
*  BBBB
*  CCCC

有序列表则直接以数字表示：  

	*  AAAA
	*  BBBB
	*  CCCC
展示出来如下：

1.  AAAA
2.  BBBB
3.  CCCC

但是注意：列表和前面文段之间需要加上一个空行，否则Mackdown语法不能识别。

###分割线
可以这样写（还有别的语法，但是学会一种就够啦）：  

	***
 展示出来如下:
***

###代码
代码块有两种写法:
1）选中代码块，按Tab键，如下:

		AAA
		BBB
展示出来如下:

	AAA
	BBB

还有一种就是如下写:  

	​```java
	public static main(){}
	​```
效果则是这样的:
```java
public static main(){}
```

###图片
```
![封面](http://7xktd8.com1.z0.glb.clouddn.com/cover.png)
```
实际效果就是:
![封面](http://7xktd8.com1.z0.glb.clouddn.com/cover.png)

###链接
```
[木子李的博客](http://www.muzileecoding.com/)
```
实际效果是:
[木子李的博客](http://www.muzileecoding.com/)

###表格
```
|文章 | 作者 |
|:---|:--- |
|Markdown常用语法|木子李|
```
实际效果是:

| 文章           | 作者   |
| :----------- | :--- |
| Markdown常用语法 | 木子李  |