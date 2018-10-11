---
title: Gradle系列一：Groovy语法
date: 2016-05-18 13:56:53
tags: [Gradle, Groovy]
categories: Android
---

<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/groovy-icon.png" height="250" alt="One Piece"/></div>

## 一、简介
[Groovy](http://www.groovy-lang.org/) 是 Java 平台上设计的面向对象编程语言。它拥有类似 Python、Ruby 和 Smalltalk 的一些特性，可以作为 Java 平台的脚本语言使用。<!--more-->

Groovy 的语法与 Java 非常相似，多数的 Java 代码同时也是正确的 Groovy 代码，两者可以混编，__Groovy 代码最终还是会被编译为字节码运行在 JVM 上__。它与 Java 的关系可以描述如下：

<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/groovy&java.png" height="160" alt="Groovy和Java的关系"/></div>

简而言之，__两者是互补的而非替代的__。

## 二、组成
<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/Groovy组成.png" alt="Groovy组成"/></div>

如图，是 Groovy 语言主要的组成部分:

1. __GDK__  主要是针对 Java 原有的库的功能扩充，是的一些操作更加的方便；
2. __Library__  扩展功能，为常用功能开发了一组工具类，比如读取XML，SQL查询等；
3. __Language__  最核心的部分，优化了 Java 原有的语法，比如`switch`可以运用在任何类型的对象上；添加新型的语法内容，比如闭包；

## 三、环境搭建
关于环境搭建，可以参考这篇文章[Groovy安装和Eclipse插件安装](http://www.muzileecoding.com/groovy/Groovy-install-and-ide-plugin.html)。

## 四、常见语法
__【约定】__ 以下示例中给出的`assert`语句全部都为 true。

### 4.1 类声明
```Java
class Book{
	private String title
	
	Book (String theTitle){
     	title = theTitle
	}

	String getTitle(){
		return title
	}
}
```
和 Java 很像，只是语句后面不需要再添加分号。

### 4.2 方法和变量
```Java
Book gina = new Book('Groovy Study')
def hello = 'Hello, '

assert gina.getTitle() == 'Groovy Study'
assert hello + getTitleBackwards(gina) == 'Hello, ydutS yvoorG'

def getTitleBackwards(book){
	String title = book.getTitle()
	title.reverse()
}

//方法调用形式
println ('Groovy is great!')
println 'Groovy is great'
```
这里可以看到类的初始化和 Java 并无不同(先忽略 String ，后面细说)，和 Java 不一样的是，定义/声明变量和方法使用`def`关键字，不需要再指明类型了。

### 4.3 GroovyBean
```Java
class BookBean{
	String title
}

def groovyBook = new BookBean()

groovyBook.setTitle('Groovy conquers the world')
assert groovyBook.getTitle() == 'Groovy conquers the world'
```
Groovy 不需要手动添加 getter 和 setter ，会为类自动生成 —— 声明 Bean 成为一件简单清爽的事情。

### 4.4 GString
__这个是重点！！__

```Java
def nick = 'ReGina'
def book = 'Groovy Study'
assert "$nick is $book" == 'ReGina is Groovy Study'

def language = 'groovy'
def improvedSentence = "${language.capitalize()} is awesome!”
assert improvedSentence == 'Groovy is awesome!'

assert "abc" - "a" == "bc"
```
String 的定义既可以使用单引号，也可以使用双引号，操作也变得更加丰富：添加了变量替换、方法调用（值计算）、加减操作等强大形象的功能。

其实 Groovy 中的 String 不止以上两种形式，以下是汇总表格:

<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/groovy-string.png" alt="Groovy字符串形式"/></div>

如图，除了单引号，双引号，还有三个单引号，三个双引号的形式，三个引号是用于多行展示的，比Java要方便很多。其中`Placeholder resolved`表示是否支持变量计算替换，`Backslash escape`表示是否支持转义。

### 4.5 Number
```Java
def x = 1
def y = 2
assert x + y == 3

assert x.plus(y) == 3
assert x instanceof Integer
```
数字除了相加，还能调用方法。这是因为__在 Groovy 中，一切都是对象，没有基本类型__，基本类型都会升级成包装类型运行。

### 4.6 Range
```Java
def x = 1..10
assert x.contains(5)
assert x.contains(15) == false
assert x.size() == 10
assert x.from == 1
assert x.to == 10
assert x.reverse() == 10..1
```
一个新的类，用于表达一个范围。它具备很多神奇的功能，下面的一些例子中可以看到它的一些用法，具体可以参考[官方Doc](http://www.groovy-lang.org/documentation.html)、

### 4.7 List
```Java
def num = ['一', '二', '三', '四', '五', '六', '七', '八', '九', '十']
assert num[4] == '五'

num[10] = '十一'
assert num.size() == 11

def list = [0, 1, 2, 3, 4]
list.each() { item ->
	assert item == list[item]
}

List list2 = [1, 2, *list, 3, 4]
assert list2 = [1, 2, 0, 1, 2, 3, 4, 3, 4]

def myList = ['a', 'b', 'c', 'd', 'e', 'e', 'f']

assert myList[0..2] == ['a', 'b', 'c']
assert myList[0, 2, 4] == ['a', 'b', 'e']

myList[0..2] = ['x', 'y', 'z']
assert myList == ['x', 'y', 'z', 'd', 'e', 'f']

myList[3..5] = []
assert myList == ['x', 'y', 'z']

myList[1..1] = [0, 1, 2]
assert myList == ['x', 0, 1, 2, 'z']

assert myList[-1] == 'z'

List myList = ['a', 'b', 'c']
assert ['a', 'x', 'c'].grep(myList) == 'a'

def kings = ['Dierk', 'Paul']
kings = kings.sort{ item -> 
	item.size()
}
assert kings = ['Paul', 'Dierk']

def odd = [1, 2, 3].findAll {item ->
	item % 2 == 1
}
assert odd == [1, 3]
```
List 是我们首先介绍的一个集合 ，可以看到它的声明、使用方式和我们熟悉的数组非常类似，配合 range 使用，更加炫酷噢(看不懂的地方语法可以后面再看)！

这是 GDK 对 Java 增强的一个很好的例子。

### 4.8 Map
```Java
def http = [
	100 : 'CONTINUE',
	200 : 'OK',
	400 : 'BAD REQUEST'
]

assert http[200] == 'OK'
assert http.200 == 'OK'
http[500] = 'INTERNAL SERVER ERROR'
assert http.size() == 4
```
这里展示的是 Map 的增强使用，非常符合人的直觉思维，比 Java 的阅读性也更高。

### 4.9 Control Structure
```Java
for(index in 1..10){
	println index
}

def list = [0, 1, 2, 3, 4]
for(index in list){
	println index
}

def age = 36
switch(age){
	case 16..20 : break;
	case 21..50 : break;
	default: throw new IllegalArgumentException()
}
```
控制结构也得到进一步增强，比如例子中，for 可以与 range 配合使用，`switch...case...`也可以和 range 结合。

### 4.10 Other
```Java
assert code = '1 + 49.0'
assert 50.0 == evaluate(code)

value ?: default

switch(candidate){
	case classifier1: handle1() ; break;
c	ase classifier2: handle2() ; break;
}
```
这些是一些额外的例子，比如计算表达式的值，三目操作符的简化（空则用 default 值），主要目的是展现一下 Groovy 所做的优化以及一些"奇技淫巧"。

### 4.11 总结
Groovy 对原 Java 的语法以及集合等做了大量的扩充，不但增强了原有功能，使用起来更加快速自然，也添加新的语法，使得语言的表达能力更强。

上面的例子并不系统，主要目的是为了让读者对 Groovy 这门语言有大概的认知。系统学习的话还是建议看[官方文档](http://www.groovy-lang.org/documentation.html)。

## 五、Closure
本节所要介绍的 __Closure(闭包)__ 是语法的一个重要组成部分，因为非常重要，因此单独拉出来作为一节阐述。
>要学习 Gradle，闭包的概念必须理解。

### 5.1 Closure是什么 ？
对于 Java 程序员来说，Closure 是一个比较陌生的概念: 

1. A closure is a piece of code wrapped up as an object. 
2. It acts like a method in that it can take parameters and it can return a value. 
3. It's a normal object in that you can pass a reference to it around just as you can a reference to any other object. 

这就是 Closure，中文称为闭包。从概念上类比一下，闭包就是 C++ 中的函数（可以通过函数指针引用），OC 中的 Block，JavaScript 中的 Closure。

通俗的说：__闭包就是一段代码，可以接受参数，可以在程序中像变量一样传递。__

### 5.2 声明和使用
先来个最简单的例子：

```Java
Closure printer = {line -> println line}
```
以上就是一个简单的闭包声明，它接受一个参数 line。使用例子如下：

```Java
Closure printer = {line -> println line}
printer("Hello world")
```
执行以上代码，就可以看到命令行中打印出 "Hello world"。可以看到闭包的使用和方法调用极其类似。下面来展示更多的例子：

```Java
def adder = {x, y -> return x+y} //闭包adder
assert adder(4, 3) == 7
assert adder.call(2,6) == 8
```
一目了然。

```Java
def benchmark(int repeat, Closure worker) {
	def start = System.nanoTime()
	repeat.times{ worker(it) }

	def stop = System.nanoTime()
	return stop - start
}

def slow = benchmark(10000,  {(int)it/2})
def fast = benchmark(10000) {it.intdiv(2)}
assert fast * 15 < slow
```
这个例子比较复杂一些:

1. 首先定义了一个方法`benchmark`，它接受一个闭包作为参数；
2. 调用的时候直接传递进去两个闭包: `{(int)it/2}`和`{it.intdiv(2)}`；
3. 在方法内部，通过`worker(it)`的方式调用闭包；

### 5.3 闭包要点
关于闭包，还有几个要点是必须要了解的：

1. 当看到 {} 的时候，应当认为等同于:new Closure(){}；
2. 任何一个 closure 都有一个默认的参数：it，它指向传递给 closure 的第一个参数，如果没有传递参数，则 it 为 null；
3. closure 总是有返回值，如果没有写 return，则返回最后一行代码的左值；如果没有值，则是 null；

下面给一个例子解释一下第二点和第三点：

```Java
//定义闭包，使用默认参数it
def odd = { it % 2 == 1 }

//将List中的数传递进入闭包，如果闭包返回true，则放到结果List中去
assert [1, 2, 3].grep(odd) == [1, 3]

//将switch的参数传给闭包，如果为true，则执行case
switch(10){
	case odd : assert false
}

if (2 in odd)
	assert false
```
这个例子展示的用法比较偏，但正展示了 Groovy 的强大功能，其中闭包 odd 的返回值就是`it % 2 == 1`。

### 5.4 与方法的关系
Closure 确实是一种新的语法机制，但是 Groovy 提供了一种办法，可以将一个方法转换成闭包:

<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/grrovy-closure&method.png" height="188" alt="Groovy方法转换成闭包"/></div>

其中`receiver`代表的是一个对象实例，`.&`是固定格式，`someMethod`是对象的一个方法名。来看个例子:

```Java
class MultiMethodSample{
	int mysteryMethod (String value) {
		return  value.length()
	}
	
	int mysteryMethod(List list){
		return list.size()
	}
	
	int mysteryMethod(int x, int y){
		return x + y;
	}
}

MultiMethodSample instance = new MultiMethodSample()
Closure multi = instance.&mysteryMethod

assert 10 == multi ('string arg') 
assert 3 = multi (['list', 'of', 'values'])
assert 14 = multi (6, 8)
```
如图，通过`instance.&mysteryMethod`就可以生成一个闭包，这个闭包与一般定义的还不同，它可以根据参数的不同执行到不同的方法，保留了方法的重载机制 —— 从这个角度说：__闭包和方法本质上就是一样的__。

### 5.5 Scope
这里要讲一个很重要的东西: __Scope__，什么是 Scope 呢？
>The environment available inside a closure is called its __scope__.

具体来说，Scope 就是考察:

1. this指向什么？
2. 可以获取哪些外部变量和方法? 

#### 5.5.1 `this`指针
来看个例子:

```Java
def x = 0
10.times {
	x ++
}

assert x == 10
```
`times`方法本身并不知道 x 的存在，但 Closure 却可以对`x`进行读写，这是怎么做到的呢？

这里又要引入一个新的概念 __Birthday Context__ :

```Java
The closure somehow remembers the context at the point of its declaration and carries it along throughout its lifetime. That way, it can work on that original context whenever the situation calls for it. This is called the __birthday context__ of the closures. 
```
即：闭包会"记住"它声明的时候的上下文环境，在它的生命周期期间内一直引用着这个环境，这样，闭包就能随时操作原来的环境(包括变量)，这个上下文环境就称为闭包的 __Birthday Context__ 。

还是以上面的例子为参考，引用关系如下:

<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/闭包birthday-context.png" height="150" alt="Groovy的Birthday Context引用关系"/></div>

如图，Script 就是代表脚本文件本身，`x`就是声明在脚本文件中的，同时脚本文件中有声明了闭包`{x++}`，这个闭包就持有对外部环境的引用，这里可以看到就是`x`，当把闭包传给`times`函数的时候，闭包依然可以获得`x`的引用。

在 Java 中，在一个方法中调用`this`指向的是当前对象本身，那么在闭包中使用`this`也是指向当前对象么？非也:
>its owner: the object in whose scope the closure was declared.

__闭包中的`this`指向的是它的 owner，即创建它的对象__。还是来看个例子:

```Java
class Mother{
	int field = 1
	int method () { return 2 }
	Closure birth(param){
		def local = 3
		def closure = {caller ->
     		[ self : this, field:field, method: method(),
     		local : local, param : param, caller : caller,
     		owner : owner,  delegate: delegate
     		]
		}
		return closure
	}
}

Mother julia = new Mother()
def closure = julia.birth(4)
def context = closure.call(this)

assert context.self == julia
assert context.field == 1
assert context.method == 2
assert context.local == 3
assert context.param == 4
assert context.caller == this
assert context.owner == julia
```
读者可以自己跑一下看看：例子中的 closure 中的 this 指向的就是它的创建者 mother。
>【注意】方法不是对象，因此例子中的 closure 的创建者不是`birth`方法，而是 julia 对象。

#### 5.5.2 外部变量和方法
还是从前面的例子看，`method()`方法和 field 变量是如何解析到的呢？

在 Closure 中，类似 method() 和 field 这样的类成员称为 __vanilla reference__ 。vanilla reference 解析的时候，默认前面是加上`this.`引用的，即它们的解析是针对 owner 的。

#### 5.5.3 delegate
__delegate__ 是从例子中看到的，它是闭包的一个默认属性，它有什么作用呢？

```Java
//假设List中有这个方法
class List{
	def with(Closure doit) {
		doit.delegate = this
		doit()
	}
}

def list = []
def expected = [1, 2]
list.with{
	// birth context 中并没有 add 方法
	add 1
	add 2
	assert delegate == expected
}
```
假设在`List`类中有一个`with()`方法，闭包传递给它之后，先修改 delegate 为 this，然后再执行闭包。那么上面的 assert 就成立。为什么呢？

因为 __delegate 实际是用于更改 vanilla reference 的解析对象的__。前面说了，在闭包中调用方法，默认前面是加上 this 的，但是 delegate 可以修改这个默认值，在前面的例子中，我们实际把 delegate 修改指向了 this，这个 this 实际指向的是 List 本身，因此这个`add()`方法调用的是 List 本身的`add()`方法，所以最后`assert delegate == expected`是成立的。

#### 5.5.4 Vanilla Reference解析
前面说了 __Vanilla Reference__ 的解析方式，有以下三种：

1. The closure itself : 比如例子中的 local 变量；
2. The owner : 比如第二个例子中的 expected；
3. The delegate : 比如`add()`方法的解析；

既然有三个解析途径，那么解析顺序是如何的呢？是不是设置了 delegate 就一定会改变 __Vanilla Reference__ 的解析对象呢？

首先肯定是本地变量解析优先的，剩下的 owner 和 delegate 的解析顺序，则是决定于`resolveStrategy`，总共有四种情况:

1. delegate first // 优先针对 delegate 解析
2. delegate only // 只对 delegate 解析
3. owner first // 优先针对 owner 解析
4. owner only // 只针对 owner 解析

修改例子如下：

```Java
closure.resolveStrategy = Closure.DELEGATE_ONLY
```
通过更改解析策略，就可以控制解析对象和顺序。

## 六、总结
Groovy 作为一门脚本语言，与 Java 保持极大的相似性和兼容性，学习成本很低，并且极大的扩展了 Java 语言的表达能力！

这里分析 Groovy，主要是为学习 Gradle 提供一个基础，因此这里的列举既不系统也不深入，如果要深入学习，推荐以下资料:

1. [官方文档](http://www.groovy-lang.org/documentation.html)，Groovy的官方资料还是很全面的；
2. 《Groovy In Action》，这个系列书对于入门以及稍微深入一门技术都还挺不错的；