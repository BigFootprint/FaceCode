---
title: MVC，MVP，MVVM
date: 2016-03-24 10:14:04
tags: [代码设计]
categories: Android
---

本来想写一篇从MVC -> MVP -> MVVM演变过程的文章，介绍一下各自的不同点和适用常场景。但是查了一些资料之后，觉得对于MVC和MVP的理解是仁者见仁智者见智的，很多文章看上去叙述都是比较靠谱的，但是在一些细节上有所不同。本文不打算深究这些不同，而是思考一下为什么会出现这样的划分。

这里推荐一些文章，可以帮助理解这些这几个模式:

1. [MVP: Model-View-Presenter
   he Taligent Programming Model for C++ and Java](http://www.wildcrest.com/Potel/Portfolio/mvp.pdf)
2. [Model–view–controller](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller)
3. [Flux vs. MVC (Design Patterns)](https://medium.com/hacking-and-gonzo/flux-vs-mvc-design-patterns-57b28c0f71b7#.in4m1jpvu)
4. [Design Patterns - MVC Pattern](http://www.tutorialspoint.com/design_pattern/mvc_pattern.htm) 
5. [Separated Presentation](http://martinfowler.com/eaaDev/SeparatedPresentation.html)
6. [MVC or MVP Pattern – Whats the difference?](http://www.infragistics.com/community/blogs/todd_snyder/archive/2007/10/17/mvc-or-mvp-pattern-whats-the-difference.aspx)

<!--more-->如果一个应用很小，那么以什么样子的方式写问题都是不大的，比如以前上学的时候做的课堂作业，功能简单，不需要后期维护，从没想过设计的问题。

当应用大了以后，功能变得复杂起来：可能需要修改原有的功能，可能新开发的功能与原有的功能有交集，那不管是几个人写，有些问题都需要注意起来。
>不要逞强，觉得一个人维护的应用可以按照自己的性子写，写得多了也扛不住，有血泪教训的。

比如不要重复代码，保持类或者模块的职责单一，高内聚低耦合，一开始可能不觉得有什么，但是一旦复杂起来就会发现注意这些点是多么的可贵。
>遇到过因为Copy代码导致出Bug的情况，因为只记得修改一个地方。也遇到过因为类的职责过于庞大，导致修改的时候小心翼翼，因为连变量名字取起来都觉得要重复了，后期相关功能的开发都需要去修改这个类。

看了一些资料，感觉这几个模式都在试图践行以下两个原则:

1. 保持职责单一: 一个模块一个目的，该干啥干啥；
2. 分离易变和不易变的部分: 不易变化的部分相对稳定，维护起来既不容易分心，也容易复用，变化的部分将以最小的代价变化；

实际上MVC/MVP模式设计还有一个很重要的目的: 方便测试。UI测试比业务逻辑测试要难的多，而且业务逻辑的测试相对而言更加重要。MVC/MVP其实就是建立在这几点基础上，不信我们下面来慢慢分析。

基于以上这些目的，MVC/MVP首先要做的事情就是分离出UI层(因为UI层不不容易进行测试且易变)。那么应该怎么分离呢？

关于这一点，【推荐5】里面说的非常详细，并且给出了一个很好的例子，读者可以去仔细揣摩一下，应该可以很直观的感受到如何做以及这样做的好处。作者还给出了一个分离的标准，这个标准非常非常好，这样可以把View很彻底的拉出去:

__假设我要为这个应用重写一个UI层，使用的也是同样的设计方式，那么新的UI层和旧有的UI层之间会有相同代码么？如果有，那么极有可能这段代码应该放到业务层去。__  

分层之后就会产生一个问题: 两层之间如何交互。我们遵循【推荐5】的叫法，把这两层称为Presentation层和Domain层。交互分为两个方向:  

1. __Presentation层->Domain层__ Presentation层主动触去调用业务层的方法处理业务逻辑，就像【推荐5】中的例子，这样没啥问题；
2. __Domain层->Presentation层__ 要Domain层能主动更新Presentation层，有两个办法，第一个是Domain层持有Presentation层的引用，第二个办法是Presentation层在Domain层设置观察者。

Presentation层->Domain层问题不大，问题主要出现在Domain层-> Presentation层这个方向上，第一种办法肯定不行，这样Domain层依赖于Presentation层，还怎么单独做测试？第二个办法比较靠谱，这种办法和MVP推崇的P和V之间的交互类似，V和P之间是通过接口交互的，V实现该接口，P引用该接口(面向接口编程)。

为什么说这样靠谱呢？因为这样就可以很容易的撇开Presentation层进行单元测试了。通过逻辑分离+面向接口，我们完全可以在测试的时候Mock一个“Presentation”，来对Presentation层调用的所有接口进行测试。

接着我们设想一下真实的开发场景，如果Domain层就这样做成一块，那么随着业务的复杂化，Domain层会变得越来越庞大(Presentation层一般不会，后面会提到如果出现了应该怎么办)。

那么如何维护Domain层呢？我们遵循目标2再进行分层。对于一个应用来说，它的数据基础相对稳定，即数据对象模型比较稳定，可以单独将这块抽出来成一个模块，稳定的为App提供所需的数据支持，比如加载、保存等，暂时称之为数据层。Domain层剥离掉数据层，剩下的就是业务逻辑，暂时称这一层为业务层。
>到这里，其实就已经看到了MVP模型了。

我们可以将业务逻辑独立到另外一个模块中去，这个模块有如下特性：易变而且复杂。这个模块和其余两个模块什么关系呢？

首先我们牢记一点，分离Presentation层是为了测试剩余的业务代码，那么我们决不能将Presentation层的东西、概念引入到数据层和业务层中去。其次数据层是提供较原始数据的地方，它必须经过处理才可以丢给Presentation显示，Presentation层的操作也需要映射成一些方法的调用和逻辑操作才能传到数据层进行相关的保存操作，而这一层处理、映射就是交给业务层来实现的。

因此可以看到MVP模式下，View(UI层)和Model(数据层)之间是没有交互的，一切都是通过Presenter(业务层)来中转的。这样的设计，将最易变的部分放在业务层中，通过业务层隔离Presentation层和数据层，能够较大限度的使这两个部分变得灵活。

这种模式设计也解决了之前我们遇到的问题: 即一个业务层支持多个Presentation层。MVP里面是主张一个View对应一个或者多个Presenter的，这时的既可以重用也可以多样化，为什么呢？因为我们把不变的Model层抽离出去了，如果业务层需要变化，那么一定是不带重复的变化。Presenter变成了仅仅处理业务逻辑的地点，粒度上也更好控制。

举个🌰，比如我写一个Android Activity，这个Activity上有多个Fragment。我可以为这些Fragment配置一个统一的Presenter，但是我也可以为每一个Fragment配置一个Presenter。甚至因为一个Fragment上显示的元素之间相对独立，比如我有一个广告位，这个广告位和页面的其余元素毫无关系，并且广告位还可以在其余页面上重用，那我就可以单独为这个广告位View配置一个Presenter，这样一个Fragment也可以对应多个Presenter了。这种粒度控制变得相当灵活。如下图，View和Model之间可以任意拼接合适的Presenter:

<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/MVP模式.png" width="250" alt="MVP模式"/></div>

应用说白了其实就是通过UI展示数据，修改数据的工具。举个例子: 你下一个外卖单，首先浏览附近的商户，这就是展示数据，最后下单，其实就是为这个账户在数据库中插入一条订单数据。整个应用就是在干这件事。因此数据本身是相对稳定的，而UI呢，很灵活易变（显示内容会变化，形式也会变化，比如以H5的形式显示）。这两块都应当独立出去，剩下的就是业务逻辑，还是以下外卖单为例，创建订单前，可能需要校验配送距离，是否能参加立减活动，这些东西非常易变，理应单独抽离出来。

通过以上职责分配，我们大致可以将应用分为三个模块:

1. __View层__ 负责显示，没有任何的业务逻辑；
2. __Model层__ 负责提供数据，也没有复杂的业务逻辑，只负责简单的操作数据；
3. __Presenter层__ 负责业务逻辑，是View和Model的中间中转层；

这种分层分块，是满足我们的三个目标的。【推荐6】里面也指出，这样的分配有助于各个模块的单独发展。比如Model层的ORM，View层的注解，Presenter层的也有IOC框架支持。

以上，就是为什么一个App可以划分出三个大块的原因。每一个大块本身也可以做出更细粒度的划分，比如数据层也可以划分出Service层，这些内部的划分很大程度上也是为了实现前面所说的两个目标，使得代码更好维护。
>以上是结合日常开发和这三种模式设计的一些思考，后期有所实践再做补充。