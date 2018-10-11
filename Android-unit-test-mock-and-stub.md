---
title: Android单元测试(二)——Mock & Stub
date: 2015-11-18 13:16:19
tags: [测试]
categories: Android
---

本文翻译总结自Martin Fowler（不知道是谁？！！）的[《Mocks Aren't Stubs》](http://martinfowler.com/articles/mocksArentStubs.html)

Mock对象是在XP社区出现的，后面越来越多的出现在XP的开发测试流程中。Mock对象的概念一直很模糊，尤其经常会和stub——一个常见的测试环境辅助概念混淆起来，Martin Fowler在很长一段时间内也认为它们（mock和stub）是差不多的，但是通过和mock开发者的交流使得Martin Fowler意识到两者是有区别的。<!--more-->

两者的区别其实包含两部分:

* 如何验证测试结果：状态验证(state verification) VS 行为验证(behavior verification)；
* 测试和设计如何协作：经典形式(classical)和Mock形式(mockist)（Martin Fowler的分类）；

## 普通测试
Martin Fowler在文章中举了个🌰，这里原封不动的阐述。🌰是这样的：我们想要创建一个Order对象，这个对象会从一个Warehouse对象中获取资源（简单说就是Warehouse对象会根据Order对象分配物资），Order对象很简单，只有一个Product对象以及数量，Warehouse对象则包含着不同Product的存货清单。当我们要求Warehouse对象根据Order对象分配物资的时候，有两种情况:

1. 物资数量充足，可以正确分配，订单完成，相应的数量被从Warehouse对象的存货清单中减去；
2. 物资数量不足，订单不能完成，Warehouse的存货清单不变动；

这两种行为可以进行一些测试，下面是一种传统的JUnit测试：

```java
public class OrderStateTester extends TestCase {
	private static String TALISKER = "Talisker";
	private static String HIGHLAND_PARK = "Highland Park";
	private Warehouse warehouse = new WarehouseImpl();

	protected void setUp() throws Exception {
		warehouse.add(TALISKER, 50);
		warehouse.add(HIGHLAND_PARK, 25);
	}
	public void testOrderIsFilledIfEnoughInWarehouse() {
		Order order = new Order(TALISKER, 50);
		order.fill(warehouse);
		assertTrue(order.isFilled());
		assertEquals(0, warehouse.getInventory(TALISKER));
	}
	public void testOrderDoesNotRemoveIfNotEnough() {
		Order order = new Order(TALISKER, 51);
		order.fill(warehouse);
		assertFalse(order.isFilled());
		assertEquals(50, warehouse.getInventory(TALISKER));
	}
}
```
这个测试按照经典的四步进行：

1. __setup__  即setUp方法中执行的代码，还有部分在test方法中，主要是Order和Warehouse对象的建立；
2. __exercise__ order.fill是属于这个阶段的，对象开始执行一些我们需要测试的行为；
3. __verify__ assert代码属于验证阶段，来检测exercise的执行结果是否正确；
4. __teardown__ 这个测试中没有显示的teardown阶段，垃圾回收器悄悄地把这事儿给干了；

>【一句话】测试的流程是这样的：首先创建对象，准备数据(setup)，让我们测试的对象在这种上下文环境中运行(exercise)，验证结果是否符合预期(verify)，最终清理测试环境(teardown)。

在setup阶段，我们创建了两个对象——Order和Warehouse——Order是我们需要测试的类，但是在调用fill方法的时候，我们需要一个Warehouse对象，因此必须创建一个它的对象。Testing-oriented的人喜欢使用object-under-test或者system-under-test来称呼他们集中精力测试的类，在这里，就是我们的Order类，两个称呼在Martin Fowler看来都很丑陋，不过Martin Fowler仍然打算在接下来的描述中使用其中一个称呼，因为大家都这么用——System Under Test，缩写 __SUT__ 。

所以在这个测试中我们需要一个SUT(Order)和一个Collaborator(可理解为辅助者、协作者，这里保持原汁原味儿)(Warehouse)。需要Warehouse有两个原因：一个是为了使得测试能够正常运行(Order.fill方法需要调用到Warehouse的方法)，第二个是为了验证(Order.fill会改变Warehouse的内部状态，因此需要通过验证Warehouse的状态来确认Order执行的正确性)。在接下来的讨论中，我们会看到SUT和Collaborator之间是有很多不同的(在这篇文章的之前版本中，Martin Fowler分别将两者成为primary object和secondary object，即主对象和次对象)。

上面的测试中使用的是状态验证方式，即：我们通过验证SUT和Collaborator在exercised之后的状态来确认测试的结果，后面我们将看到，mock对象给出了一种不同的验证方式。

## Mock对象测试
现在我们将使用mock对象进行相同的测试，在下面的代码中我们使用jMock框架来完成mock。jMock是一个Java上的mock库，也有很多其他类似的库(EasyMock、Mockito、powermock)，但是这个库是这项技术的创始人所写的最新库(看它的官网，目前至少有5位主要贡献者，排在最前面的是Steve Freeman)，所以选择它是合适的。

```java
public class OrderInteractionTester extends MockObjectTestCase {
	private static String TALISKER = "Talisker";

	public void testFillingRemovesInventoryIfInStock() {
		//setup - data
		Order order = new Order(TALISKER, 50);
		Mock warehouseMock = new Mock(Warehouse.class);
		//setup - expectations
		warehouseMock.expects(once()).method("hasInventory")
				.with(eq(TALISKER),eq(50))
				.will(returnValue(true));
		warehouseMock.expects(once()).method("remove")
				.with(eq(TALISKER), eq(50))
				.after("hasInventory");
		//exercise
		order.fill((Warehouse) warehouseMock.proxy());
		//verify
		warehouseMock.verify();
		assertTrue(order.isFilled());
	}
	
	public void testFillingDoesNotRemoveIfNotEnoughInStock() {
		Order order = new Order(TALISKER, 51);    
		Mock warehouse = mock(Warehouse.class);
		warehouse.expects(once()).method("hasInventory")
				.withAnyArguments()
				.will(returnValue(false));
		order.fill((Warehouse) warehouse.proxy());
		assertFalse(order.isFilled());
	}
}
```
首先我们来看testFillingRemovesInventoryIfInStock：setup阶段已经有明显的不同，它分为两部分：data和expecttions（见注释），即设置数据和期待行为。data部分用于建立我们需要用到的对象，这里和传统的srtup区别并不大，区别在于我们创建的对象，SUT是一样的，但是Collaborator完全不一样——它是一个Mock对象，一个Mock的Warehouse！！天呐！！技术上来说它是一个Mock类的实例。

setup的第二部分，也就是expecttions部分，是用于在Mock对象上创建期待行为的，这个部分在描述了SUT在exercised阶段，哪些方法会被调用，以及返回结果是什么样子的。实际的期待描述可以比这个更复杂，前面说的比较抽象，说白一点：这里可以定制一些行为，设置一些限制，比如方法调用次数，序列，返回值等，通过这些设置，可以描述测试者对测试流程的期待，创建一个稳定的环境，也便于后期认证。

一旦expectations建立好，我们就可以进入exercise阶段(这个阶段干啥的？看前面)。之后我们进入verify阶段，包括两个方面：和传统测试一样验证SUT，但同时又去验证Mock对象——验证我们的调用符合我们的expections(即setup阶段的第二部分的预设)。

这里与普通测试的最大不同就在于我们如何验证SUT和Collaborator的交互。通过状态验证，我们使用assert来确认Warehouse的状态符合预期，但是Mock对象则是使用行为验证——验证Order是否正确调用了Warehouse，验证的是调用这个行为！我们通过在setup阶段告诉Mock对象我们期望的调用行为来在最后让Mock对象进行验证，只有Order对象是进行assert验证，即状态验证的，如果整个exercise阶段没有改变Order对象的状态，那么这里没有任何的asserts。

在第二个测试(testFillingDoesNotRemoveIfNotEnoughInStock)中，我们做了一些不同的事情，首先我们使用mock而不是new方法来创建Mock对象，这是一个快捷方法，使用它可以不用显示的调用verify方法，任何通过这种方式创建的对象在方法执行后都会被自动验证。

第二个不同点在于我们通过使用withAnyArguments方法放宽了expectations的限制。这样在Order的逻辑被改变之后，这个test就不会失败，从而减轻了维护test的成本。实际上这里可以省略这个方法的调用，因为是默认的。

>【PS】Martin Fowler在文章中还举了一个EasyMock的例子，但是鉴于例子本身和我们理解本文核心阐述的概念不太相关，而我这么晚了也懒得翻译，因此省略。

## Mock和Stub之间的区别
当这两个概念刚刚出现的时候，很多人都会对它们感到迷惑，后来才逐渐了解到两者之间的区别。想要彻底理解什么是mock，理解mock和其他test double(一个术语，后面解释)的区别是很重要的。

当我们做上面那样的一个测试的时候，我们每次只能集中关注其中一个元素组件(unit)，这就是我们常说的单元测试。而问题在于在我们测试这单一组件的时候，我们经常需要其他的组件进行辅助，就像测试Order类需要Warehouse类一样。

在上面提到的两种测试风格中，第一种风格使用了一个真的Warehouse对象，第二种风格则使用了一个mock的Warehouse对象。通过Mock对象是来进行测试是不使用真实对象进行测试的一种方法，实际上还有很多别的方式可以替代真实对象。

用于描述这种情况的词汇很快变得混乱起来——各种词汇都被用到了：stub、mock、fake、dummy。在这里，Martin Fowler使用了Gerard Meszaros书中的词汇定义。

Meszaros使用 __Test Double__ 来表示所有的替代真实对象进行测试的对象，这个名词来自电影中的Stunt Double(他的目的就是取一个没有被广泛使用的名字)。Meszaros随后又定义了四类Double:

1. __Dummy对象__ 是那些被到处传递，但是从不被使用的对象。通常来说只是用来填充参数列表；
2. __Fake对象__ 有实际实现，但是经常会采用一些简单的方式，从而使得它们不能在最终的产品中使用；
3. __Stub对象__ 封装了一些测试中需要用到的调用响应数据。Stub也有可能会记录一些调用信息，比如一个email gate stub就会记录它发送的信息以及发送信息的次数；
4. __Mock对象__ 是提前编写有详细的关于收到调用方法的expectations的对象，好拗口，还是来句人话：这个对象用于描述自己期望得到什么样子的方法调用，还不了解的话，看看前面JMock示例中的Warehouse Mock对象，它就有expectations描述。

在这几类对象中，只有mock对象需要行为验证，其余的几类只能做状态验证。Mock对象通常在exercise阶段也会扮演一些别的对象的角色，因为它需要是的SUT相信它自己正在和一个真正的Collaborator打交道——但是Mock对象在setup阶段和verify阶段和其余对象是绝对不同的。

为了更好的理解test double，我们需要延伸一下我们的例子。很多人只有在真实的对象用起来不方便的时候才会选择使用test double，一个更加经典的需求是：我们希望在Order不能完成的时候发一封邮件。这个时候我们在测试阶段并不希望真正的发送一封邮件出去，所以在这个时候我们希望为邮件系统创建一个test double——一个我们可以操控的邮件系统。

这里，我们可以看到mock和stub的区别，如果我们为邮件行为创建一个测试，我们可以使用如下这个简单的stub来实现：

```java
public interface MailService {
	public void send (Message msg);
}
public class MailServiceStub implements MailService {
	private List<Message> messages = new ArrayList<Message>();
	public void send (Message msg) {
		messages.add(msg);
	}
	public int numberSent() {
		return messages.size();
	}
}     
```
然后我们可以在Stub上如下使用状态验证：

```java
class OrderStateTester...
public void testOrderSendsMailIfUnfilled() {
    Order order = new Order(TALISKER, 51);
    MailServiceStub mailer = new MailServiceStub();
    order.setMailer(mailer);
    order.fill(warehouse);
    assertEquals(1, mailer.numberSent());
}
```
当然啦，这只是一个很简单的例子——只测试了一个消息被发送了，而没有测试邮件被发送到了正确的人，内容正确与否，但是这并不妨碍我们讲述清楚我们的关注点。

使用Mock的话，测试看上去会很不一样：

```java
class OrderInteractionTester...
public void testOrderSendsMailIfUnfilled() {
    Order order = new Order(TALISKER, 51);
    Mock warehouse = mock(Warehouse.class);
    Mock mailer = mock(MailService.class);
    order.setMailer((MailService) mailer.proxy());

    mailer.expects(once()).method("send");
    warehouse.expects(once()).method("hasInventory")
      .withAnyArguments()
      .will(returnValue(false));

    order.fill((Warehouse) warehouse.proxy());
  }
}
```
在两种情况下，我们都使用了一个test double来代替真正的mail服务——前者使用的是Stub，后者使用的是Mock，这里可以看出区别——前者进行的是状态验证，后者进行的是行为验证！

Mock对象总是使用行为验证，Stb也可以，Meszaros将使用行为验证的Stub称为Test Spy，区别就在于test double如何运行和验证。

## Classical和Mockist测试
现在我们来看一下第二个不同点：classical和Mockist两种风格，区别点在于 __什么时候使用Mock或者别的double__ 。

classical测试驱动开发者主张尽可能的使用真实的对象，只有在使用真是对象遇到问题的时候才会使用double。所以classical测试驱动开发者会使用一个真实的Warehouse对象和一个double的Mail服务对象。

Mockist测试驱动开发者则相反，他们总是会使用mock对象，也就是说，他们会使用一个Mock的Warehouse。Mockist的一个重要分支是BDD(Behavior Driven Development)，这里不赘述。

## 选择
在这篇文章中，阐述了两种区别：状态验证和行为验证，Classical和Mockist。那么到底应该怎么选择呢？首先我们来说一下状态验证和行为验证的选择。

首先要考虑的是上下文环境：我们面对的是一个很好创建的协作关系，比如Order和Warehouse，还是一个复杂难用的，比如Order和Mail服务？

如果是一个简单的，那么选择也是简单的，Classical测试者不会使用mock、stub或者其他任何的double，我直接使用一个真实的Object并进行状态验证；而Mockist也可以选择使用mock对象并进行行为验证。这个可以参考前面的例子，因为成本很小，选择什么都可以，因此没有太多需要考虑的。

如果是一个复杂的协作关系，name对于Mockist来说也没什么需要考虑的——使用mock和行为验证。但对于Classical来说，就需要做出一些选择了，但问题不大，通常来说他们会根据实际情况，选择最方面快捷的方式。

所以结论是：状态验证和行为验证并不是一个大问题，真正的问题在于Classical和Mockist之间的选择，也就是什么时候使用Mock对象的问题。但是状态和行为验证确实会影响到这个讨论，后面会集中阐述。Martin Fowler在这里抛出了另外一个情况：假设测试者遇到了一个非常难以进行状态验证的情况，即使他面对的是一个简单的写作关系，极佳的例子就是Cache，因为你不能从他的状态来确认Cache是否命中或者不命中，这个时候行为验证反而是一个非常明智的选择。Martin Fowler认为还有很多别的类似情况。

在我们进行决策讨论之前，我们先来讨论一下我们做出决策的时候需要考虑的因素。

### Driving TDD
Mock对象诞生于XP社区，而XP的一个重要原则就是TDD——系统的迭代进化是由测试驱动进行的。因此mockists强调mockists对于设计的作用是很正常的，尤其是他们宣传一种称作需求驱动的开发方式，在这种方式下，开发者首先通过写测试来开发一个user story，理清楚SUT、对collaborators的expectations以及SUT和collaborator之间的交互关系，这样就可以有效地设计出SUT的外围接口。

一旦你的第一个测试运行起来，mock中携带的expectations就为下一步的开发和测试提供了一个出发点。开发者可以将每一个expectations转换成collaborator上的test，这样循环迭代，逐步开发建立SUT。这种风格也被称为outside-in，描述的非常到位。在分层系统中这种方式工作的很好，首先你可以开发UI，底层使用Mock实现，然后你可以为更底层写测试，逐步往下延伸，这是一个高度结构化的开发方式，很多人相信这对于OO和TDD的初学者都很有裨益。

经典的TDD给出的指导略有不同。你可以使用相同的开发步骤，但是是使用的stub，而不是mock。这样当你需要collaborator的协助的时候，你只需要将测试的需要的返回结果硬编码进去就好了，当你编码到这一段的时候，你可以将结果替换成实际运行的逻辑代码。

经典的TDD还可以做别的事情，一个流行的形式就是middle-out。在这种风格下面你需要考虑你开发的功能以及这个功能所需要的domain(不知道怎么翻译更加合适)，你可以首先获取到一个满足你需求的domain对象，一旦它可以工作了，你就可以在此之上进行开发。使用这种风格不需要你造假任何东西。因为一开始我们可以将精力集中在domain模型上，可以防止domain逻辑泄露到UI更上层，因此很多人喜欢这种风格。（看上去像是一种变种）

有很多的想法都是想要一层层构建应用，并且不是直到另外一层完成才开发下一层。Classicist和Mockist都有着敏捷背景并且青睐细粒度的迭代，因此他们都不会选择一层层开发，而是一个功能一个功能的去开发。

>【有话说】不使用TDD的团队是否就不需要考虑这个了？不过通过Stub或者Mock的方式实现快速迭代开发是一个不错的思路，可以短时间内让精力集中在某一个点上。

### Fixture Setup
使用classic TDD，测试的时候不但需要创建SUT，还需要创建所有相关的collaborator，例子中我们只创建了很少几个对象，但是实际的测试中，我们会创建很多的collaborator，通常来说这些对象会随着一个test的执行被创建和销毁。

Mockist测试则不一样，它只需要创建SUT和它的直接关联对象，这能减少创建一些复杂对象带来的工作量。

实践中classic测试者会尽量复用复杂的collaborator，最简单的方式就是将这类collaborator放到xUnit的setup方法中，更为复杂的collaborator则可能需要被多个test类使用，所以在这种特殊情况下，你需要创建一个collaborator生成类。Martin Fowler将这中生成类称为Object Mothers。使用Object Mothers在大型的classic测试中非常重要，但是Object Mothers为整个测试添加了额外的需要维护的代码，任何的改动都会在整个测试中造成严重的影响。下面Martin Fowler所说的省略。

两种风格之间时常有所争辩。Mockists觉得创建真是的collaborator太过复杂，但是classicists认为这些collaborator都可以复用，但是每次测试都需要创建Mock对象。

>【有话说】看实际情况吧，对一个较完备的系统做测试，应该是classic方式比较快速，正如Martin Fowler所说，大部分情况下创建实际对象都是cheap的。但是不排除需要创建复杂对象的情况。

### Test Isolation
在Mockists测试情况下，如果在系统中引入了一个bug，通常只会导致包含这个Bug的SUT所在的test执行失败。但是如果使用classic方式，则会导致所有使用到这个bug对象的测试失败。

Mockists认为这种情况有很大的问题：会导致找到根本的错误需要很多的时间。但是classicists则不这么认为：通常来说，定位问题的发生点是比较简单的事情，而且如果经常测试的话，你是知道什么改动引起这种问题的（PS：现代的IDE以及版本控制工具使得这个更加简单）。

这里一个可能很重要的点在于测试的粒度。classic测试需要一次exercise很多个真实对象，经常会出现一个单独的primary test用于测试很多歌对象，而不是一个。如果这个test包含着很多个对象，找到bug的原因就比较困难了——原因就是测试太粗粒度了。

mockist测试则比较不容易出现这个问题，因为除了SUT，其余的对象都是Mock的。这里比较好的做法是：保证为了一个类创建细粒度的测试。但是比较大型集中的测试也是有存在意义的，但是只限于测试很少的一些对象。而且一旦你发现因为粒度太粗，导致测试的时候很难定位问题，你就应该创建更细粒度的测试区追踪问题。

其实经典的xUnit测试并不只是简单的单元测试，同时也是一个最小集成测试。因此很多人都期望单元测试可以发现一些单独对某个类进行测试时发现不了的问题，尤其是类之间交互时的问题。Mockist测试就缺乏这方面的能力，同时你还得关注你的expectations是否有错误，从而导致一些实际存在的错误漏掉。

不管你使用什么风格的测试，你必须综合使用粗粒度的验收测试，Martin Fowler遇到过一些因为验收测试太晚而追悔莫及的情况。

>【有话说】我倾向于classicists方式，为了创建细粒度的测试，感觉需要花费的精力是在太大，我的想法是真实对象 > Stub对象 > Mock对象。

###Coupling Tests to Implementations
当使用mockist测试的时候，我们测试的是SUT是否和它的collaborator正常交互。classic则只关心最终的状态，而不关系状态是如何产生的。因此mockist测试和方法的实现耦合度就更高——更改collaborator的调用通常会导致mockist测试跑不过。

这种耦合引起了一些关注。最重要就是TDD。使用mockist是的你需要考虑行为的实现——实际上mockist测试者认为这是一个优点。而Classicists认为在测试的时候，只去考虑外部接口是很重要的，而具体的实现应该放在测试之后再去考虑。

这种情况在使用mock工具的时候变得更加糟糕，因为mock工具通常需要你很详细的描述方法的调用和参数，即使与特定的这个test无关。jMock工具的一个目标就是使得在描述expectation的时候变得更加灵活，使得它(expectation)在不关心的地方可以描述的更加宽泛。

>【有话说】论耦合的话两者都有，只是如Martin Fowler所说，mockist不仅需要考虑一些具体的实现（方法调用），还需要考虑返回值，而Classicists只耦合了最终的状态，后者相对而言更加轻便。

### Design Style
忽略。

>【有话说】忽略了还有话说？有。实际上这一小节是Martin Fowler最关注的部分，即对测试方式的选择会如何影响你的代码设计。但我认为这需要对这两种理念有足够的了解和实践经历才会有深刻的理解，从而去思考Martin Fowler的话。而我本身不具备这样的条件。

## 到底该用哪一个？
这个问题比较难回答，Martin Fowler是一个classic测试者，他认为目前没有任何理由改变成一个mockist测试者。他在观察到mockist编码者之后，这种想法就更加坚定了，因为在写expectation的时候需要殚精竭虑去思考实现和调用方式显得非常不自然。

同时Martin Fowler也说道：在很深入的使用一项技术之前，评价它很困难。所以如果读者对Mockist有兴趣，可以尝试一把。

## 结论
xUnit测试框架和TDD越来越火热，人们通常会不求甚解的去来哦接使用一个框架，而不知道其背后设计的理念和含义。不管你一开始学习到什么框架，从不同的角度去理解它和别的框架的不同都很重要——了解背后指导它们的理念非常重要！这篇文章的目的就是指出这些理念的区别以及如何在它们之间做出权衡。

## 我言
当初也是因为要对一个Android项目做单元测试，因此开始看一些单元测试方面的知识，从Google官网上看到了对Mockito的引用推荐，从而看到其官网上的示例，进而对expectations产生了疑问——我之前并不知道还有行为验证这种东西，因此对我而言行为验证的那段代码非常难以理解，在翻阅资料的时候无意中看到了Martin Fowler的这篇文章，遂译之。

读完这篇文章，最大的收获如下：

1. 理解了行为验证 & 状态验证的区别；
2. 了解了classic和mockist两种测试风格的区别；
3. 了解了在这些区别之间做出决策需要考虑的一些因素；


我对于决策因素的一些考虑也已经通过【有话说】写在下面了。后期我的单元测试应该会侧重于【状态验证+真实对象+Stub】的方式，诚如Martin Fowler所言，mockist的方式对我而言需要考虑维护的东西太多。