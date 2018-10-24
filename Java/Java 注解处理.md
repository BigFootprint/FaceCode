---
title: Java 注解处理
date: 2018-10-24
tags: [基础]
categories: Java
---

原文地址：[ANNOTATION PROCESSING 101 by Hannes Dorfmann — 10 Jan 2015](http://hannesdorfmann.com/annotation-processing/annotationprocessing101)

在本文里面，我将介绍如何编写一个注解处理器。首先我将向你介绍注解处理指的是什么，这个强大的工具能做什么，不能做什么，然后我们会一步一步的实现一个简单的注解处理器。

## 基础

一开始我们首先要明确一件重要的事情：我们要讨论的并不是在运行时（runtime）如何使用反射解析注解（运行时：程序运行的时候），而是指发生在源码编译时（compile time）的注解处理（编译时：Java 编译器编译源码的时候）。

注解处理是 javac 内置的一个工具，用于在编译时扫描和处理注解。开发者是可以为指定的注解注册注解处理器的。此处我假设你已经对注解有基本的了解，知道如何声明一个注解，如果你不清楚的话，可以查阅 [official java documentation](http://docs.oracle.com/javase/tutorial/java/annotations/index.html) 获取更多信息。从 Java 5 开始我们就已经可以进行注解处理了，不过实际是在 Java 6（2006 年发布）我们才获得了一些有用的 API。Java 开发者过了一段时间才认识到注解处理的强大之处，随后几年注解处理才开始流行起来。

注解处理器以 Java 源文件（或者编译后的字节码）作为输入，通常会输出一些文件（比如 .java 文件）。这意味着什么？意味着你可以在 .java 文件中生成 Java 代码（但是你不能操作一个已经存在的 class 文件来添加一个方法）！生成的 java 文件会和其他手敲的源文件一样被 javac 编译。

## AbstractProcessor

让我们先来看一下处理器的 API，每一个注解处理器都继承于`AbstractProcessor`：

```java
package com.example;

public class MyProcessor extends AbstractProcessor {

	@Override
	public synchronized void init(ProcessingEnvironment env){ }

	@Override
	public boolean process(Set<? extends TypeElement> annoations, RoundEnvironment env) { }

	@Override
	public Set<String> getSupportedAnnotationTypes() { }

	@Override
	public SourceVersion getSupportedSourceVersion() { }

}
```

它总共有四个方法：

* __**init(ProcessingEnvironment env)**:__ 每一个注解处理器类都必须有一个空构造方法，然而，注解处理工具会使用`ProcessingEnvironment`参数调用注解处理器的`init()` 方法，`ProcessingEnvironment`会提供一些有用的工具类: **Elements**, **Types** 和 **Filer**，稍后我们会用到它们；
* **process(Set<? extends TypeElement> annotations, RoundEnvironment env):** 这个方法类似于注解处理器的`main`函数，这里会编写扫描、分析、处理注解和生成 java 文件的代码，后面我们会看到`RoundEnvironment` 参数可以用于查找源代码中使用了某一类注解的特定元素；
* **getSupportedAnnotationTypes():** 该方法用于指定该处理器用于处理哪些特定的注解，返回值是一个集合，包含该处理能处理的所有注解的全限定名称。换句话说，你需要在这里指定为哪些注解注册该处理器；
* **getSupportedSourceVersion():** 该方法用于指定你使用的哪个 Java 版本，通常我们返回**SourceVersion.latestSupported()**，然而如果你有好的理由，你也可以返回 **SourceVersion.RELEASE_6** 等其他值，我推荐前者；

从 Java 7 开始，开发者也可以不去重载 **getSupportedAnnotationTypes()** 和 **getSupportedSourceVersion()** 方法，如下：

```java
@SupportedSourceVersion(SourceVersion.latestSupported())
@SupportedAnnotationTypes({
   // Set of full qullified annotation type names
 })
public class MyProcessor extends AbstractProcessor {

	@Override
	public synchronized void init(ProcessingEnvironment env){ }

	@Override
	public boolean process(Set<? extends TypeElement> annoations, RoundEnvironment env) { 	  }
}
```

但是考虑到兼容，尤其是考虑到 Android 环境，我推荐还是覆写这两个方法而不是通过 **@SupportedAnnotationTypes** 和 **@SupportedSourceVersion** 注解来解决。

接下来你需要知道的事情是：注解处理器运行在自己的 JVM 上。是的，你没有听错，javac 程序启动了一个独立的 Java 虚拟机来运行注解处理器，所以这对你来说意味着什么呢？意味着你可以在注解处理器里面做任何你在别的 java 程序里面做的事情，比如使用 Guava！只要你想，你也可以使用依赖注入工具 dagger 或者其他任何你想使用的库。但是你也不要忘了，即使只是一个很小的处理器，你也要考虑到算法的效率和设计模式，就像你写其他 Java 程序一样。

## 注册你的处理器

你或许会问：我怎么把 MyProcessor 注册到 javac 呢？你只需要提供一个 .jar 文件就可以了：和其他 .jar 文件一样，将你的处理器代码打包进去，同时你需要在项目的 **META-INF/services** 位置放置一个特殊的文件：**javax.annotation.processing.Processor**。所以你的 .jar 文件目录结构会和下面一样：

```java
MyProcessor.jar
	- com
		- example
			- MyProcessor.class
	- META-INF
		- services
			- javax.annotation.processing.Processor
```

**javax.annotation.processing.Processor** 文件的内容是一个注解处理器全限定名称的列表，一行一个：

```java
com.example.MyProcessor
com.foo.OtherProcessor
net.blabla.SpecialProcessor
```

当 **MyProcessor.jar** 被包含在编译路径下的时候，javac 就会自动检测和读取 **javax.annotation.processing.Processor** 文件的内容，并把 **MyProcessor** 注册为注解处理器。

## 🌰: Factory Pattern

是时候来个实际例子了，在例子中我们会使用 maven 作为构建工具和依赖管工具。如果你不熟悉 maven 那也用不着担心，所有的代码都可以在 [github](https://github.com/sockeqwe/annotationprocessing101) 上找到。

首先我必须要说一句：找一个能使用注解处理器解决问题的简单例子并不容易。在这里我们来实现一个简单的工厂模式（不是抽象工厂模式），例子中会简单介绍一些注解处理的 API。以下问题的陈述可能会有点废话，也不会在实际生活中遇到。再啰嗦一句：这个例子是用于学习注解处理而不是设计模式的。

以下是问题描述：我们想实现一个披萨店，披萨店提供给顾客两种披萨（“Margherita” 和 “Calzone”）以及 Tiramisu 作为甜点。

我们来看一下代码实现：

```java
public interface Meal {
  public float getPrice();
}

public class MargheritaPizza implements Meal {

  @Override public float getPrice() {
    return 6.0f;
  }
}

public class CalzonePizza implements Meal {

  @Override public float getPrice() {
    return 8.5f;
  }
}

public class Tiramisu implements Meal {

  @Override public float getPrice() {
    return 4.5f;
  }
}
```

代码很简单，无需多做解释。为了在 **PizzaStore** 中订购披萨，顾客必须输入披萨的名字：

```java
public class PizzaStore {

  public Meal order(String mealName) {

    if (mealName == null) {
      throw new IllegalArgumentException("Name of the meal is null!");
    }

    if ("Margherita".equals(mealName)) {
      return new MargheritaPizza();
    }

    if ("Calzone".equals(mealName)) {
      return new CalzonePizza();
    }

    if ("Tiramisu".equals(mealName)) {
      return new Tiramisu();
    }

    throw new IllegalArgumentException("Unknown meal '" + mealName + "'");
  }

  public static void main(String[] args) throws IOException {
    PizzaStore pizzaStore = new PizzaStore();
    Meal meal = pizzaStore.order(readConsole());
    System.out.println("Bill: $" + meal.getPrice());
  }
}
```

如你所见，在`order()`方法里面我们有很多的`if`判断，每当我们添加一种披萨的时候就得加一个`if`判断，但实际上我们可以使用工厂模式以及注解处理自动生成`if`判断代码。所以我们实现的代码如下：

```java
public class PizzaStore {

  private MealFactory factory = new MealFactory();

  public Meal order(String mealName) {
    return factory.create(mealName);
  }

  public static void main(String[] args) throws IOException {
    PizzaStore pizzaStore = new PizzaStore();
    Meal meal = pizzaStore.order(readConsole());
    System.out.println("Bill: $" + meal.getPrice());
  }
}
```

**MealFactory** 则会像下面这样：

```java
public class MealFactory {

  public Meal create(String id) {
    if (id == null) {
      throw new IllegalArgumentException("id is null!");
    }
    if ("Calzone".equals(id)) {
      return new CalzonePizza();
    }

    if ("Tiramisu".equals(id)) {
      return new Tiramisu();
    }

    if ("Margherita".equals(id)) {
      return new MargheritaPizza();
    }

    throw new IllegalArgumentException("Unknown id = " + id);
  }
}
```

## @Factory 注解

你知道么，我们想要的是通过注解处理器自动生成 **MealFactory**，更宽泛的说，我们想要一个可以生成工厂类的注解以及处理器。

让我们看看`@Factory`注解：

```java
@Target(ElementType.TYPE) @Retention(RetentionPolicy.CLASS)
public @interface Factory {

  /**
   * The name of the factory
   */
  Class type();

  /**
   * The identifier for determining which item should be instantiated
   */
  String id();
}
```

具体的想法就是：我们使用同样的`type()`注解属于同一个工厂的类，通过`id()`来完成**“Calzone”** 到 **CalzonePizza** 类的映射。先让我们来用用看：

```java
@Factory(
    id = "Margherita",
    type = Meal.class
)
public class MargheritaPizza implements Meal {

  @Override public float getPrice() {
    return 6f;
  }
}
```

```java
@Factory(
    id = "Calzone",
    type = Meal.class
)
public class CalzonePizza implements Meal {

  @Override public float getPrice() {
    return 8.5f;
  }
}
```

```java
@Factory(
    id = "Tiramisu",
    type = Meal.class
)
public class Tiramisu implements Meal {

  @Override public float getPrice() {
    return 4.5f;
  }
}
```

你或许会想我们能不能直接把`@Factory`注解使用在`Meal`接口上，这是不行的，因为注解不会继承（【译者注】通过 @Inheritance 可以指定注解是否可以继承）！在类 `X` 上进行注解，并不意味着继承于 `X` 的类也会自动注解。在开始编写处理器代码之前，我们必须明确以下几点：

1. 只有类可以使用`@Factory`注解，因为接口和抽象类不能通过`new`操作实例化；
2. 使用`@Factory`注解的类必须有一个空构造函数，否则我们无法实例化一个实例；
3. 使用`@Factory`注解的类必须直接或者间接继承于`type()` 方法返回的类（如果是接口，就实现它）；
4. `@Factory`注解的具有相同 `type` 的类会被分为一组并生成一个工厂类，名字会以`Factory`作为后缀，比如 `type=Meal.class`，就会产生`MealFactory`类；
5. `id`只允许传 string 类型的值，而且必须在 type 分组中唯一；

## 注解处理器

下面我会一步步指导你编写代码，并在后面跟上详细的解释。三个点（...）代表因为部分代码因为前面片段已经讨论过或者后面即将加上而被省略了，目的就是为了让代码片段更加可读，前面已经提到，所有代码可以在[github](https://github.com/sockeqwe/annotationprocessing101) 上被找到。好，让我们开始构建 `FactoryProcessor`吧：

```java
@AutoService(Processor.class)
public class FactoryProcessor extends AbstractProcessor {

  private Types typeUtils;
  private Elements elementUtils;
  private Filer filer;
  private Messager messager;
  private Map<String, FactoryGroupedClasses> factoryClasses = new LinkedHashMap<String, FactoryGroupedClasses>();

  @Override
  public synchronized void init(ProcessingEnvironment processingEnv) {
    super.init(processingEnv);
    typeUtils = processingEnv.getTypeUtils();
    elementUtils = processingEnv.getElementUtils();
    filer = processingEnv.getFiler();
    messager = processingEnv.getMessager();
  }

  @Override
  public Set<String> getSupportedAnnotationTypes() {
    Set<String> annotataions = new LinkedHashSet<String>();
    annotataions.add(Factory.class.getCanonicalName());
    return annotataions;
  }

  @Override
  public SourceVersion getSupportedSourceVersion() {
    return SourceVersion.latestSupported();
  }

  @Override
  public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
	...
  }
}
```

代码第一行我们看到了 `@AutoService(Processor.class)`，这是什么呢？这是来自于另外一个注解库的注解，`AutoService`注解处理器是由 Google 开发的，功能就是生成 **META-INF/services/javax.annotation.processing.Processor** 文件。是的，你没有听错，我们可以在注解处理器中使用注解处理器，很方便不是吗？在`getSupportedAnnotationTypes()` 方法里面我们指明了`@Factory`是处理器会处理的注解。

## Elements 和 TypeMirrors

在 `init`方法里面，我们可以得到以下引用：

* __Elements：__ 处理 __Element__ 的工具类（稍后细讲）；
* __Types：__ 处理 __TypeMirror__ 的工具类（稍后细讲）；
* __Filer：__ 顾名思义，你可以使用它创建文件；

在注解处理过程中，我们会扫描 Java 的源文件，源代码的每一个部分都是一种特定类型的 Element，换句话说：Element 代表着程序的某一个元素，比如：包，类或者方法。每一个 Element 代表着一个静态的、语言级别的结构，下面的示例中我通过注释来阐述：

```java
package com.example;	// PackageElement

public class Foo {		// TypeElement
	private int a;		// VariableElement
	private Foo other; 	// VariableElement
	public Foo () {} 	// ExecuteableElement
	public void setA ( 	// ExecuteableElement
	                 int newA	// TypeElement
	                 ) {}
}
```

你必须切换看待源代码的视角：它只是一个结构化的文本，不可执行。你可以把这个过程当做去解析 XML 文本，在 XML 解析过程中会遇到一些 DOM 元素，你可以从某个元素定位到它的父元素或者子元素。

举个🌰，如果你有一个代表类的`TypeElement`实例，你可以像下面这样遍历它的子元素：

```java
TypeElement fooClass = ... ;
for (Element e : fooClass.getEnclosedElements()){ // iterate over children
	Element parent = e.getEnclosingElement();  // parent == fooClass
}
```

如上，Elements 用于表示源码，TypeElements 代表着源码中的类型元素，比如类。然而 TypeElement 并不包含类本身的信息，比如使用 TypeElement 可以获知类的名称，但是不能得知它的父类是什么，这类信息需要通过 `TypeMirror` 获取，可以通过调用 `element.asType()` 来获取元素的 TypeMirror 实例。

## 搜索 @Factory

接下来让我们实现`process()`方法，首先我们要搜索被`@Factory`注解的类：

```java
@AutoService(Processor.class)
public class FactoryProcessor extends AbstractProcessor {

  private Types typeUtils;
  private Elements elementUtils;
  private Filer filer;
  private Messager messager;
  private Map<String, FactoryGroupedClasses> factoryClasses = new LinkedHashMap<String, FactoryGroupedClasses>();
	...

  @Override
  public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {

    // Itearate over all @Factory annotated elements
    for (Element annotatedElement : roundEnv.getElementsAnnotatedWith(Factory.class)) {
  		...
    }
  }
 ...
}
```

这里没什么高端技术，`roundEnv.getElementsAnnotatedWith(Factory.class))`获取了所有被`@Factory`注解的元素。或许你已经注意到我这里并没有说是“返回`@Factory`注解的类列表”，因为它确实只返回了__Element__ 的集合。记住：Element 可以是一个类、方法或者变量等等。所以接下去我们要做的就是检查 Element 是否是一个类：

```java
@Override
public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {

    for (Element annotatedElement : roundEnv.getElementsAnnotatedWith(Factory.class)) {

      // Check if a class has been annotated with @Factory
      if (annotatedElement.getKind() != ElementKind.CLASS) {
    		...
      }
   }

   ...
}
```

这里我们希望确保只有类会被处理。前面我们说到类是__TypeElement__，所以为什么我们不通过`if (! (annotatedElement instanceof TypeElement) )`进行检测呢？因为接口也是 TypeElement 的，因此这样检查是错误的。所以我们不应该通过 instanceof 来进行检测，而应该使用 TypeMirror 的 `ElementKind` 和 `TypeKind` 。

## 错误处理

在 `init()` 方法里面我们可以获得 __Messenger__ 的引用。Messenger 用于注解处理器报告错误信息，警告以及其他提示，它不是一个 logger，它是用于向使用你的注解库的第三方开发者输出信息的工具。在[官方文档](http://docs.oracle.com/javase/7/docs/api/javax/tools/Diagnostic.Kind.html)里面，信息是有不同级别的，最重要的是 [Kind.ERROR](http://docs.oracle.com/javase/7/docs/api/javax/tools/Diagnostic.Kind.html#ERROR) 级别，因为这种类型的信息往往表示注解处理器处理失败了，第三方使用者可能错误的使用了我们的注解（比如使用 @Factory 注解一个接口）。这里和传统的 Java 程序通过抛异常来表示错误有所不同：如果你在`process()` 中抛出异常，那么第三方开发者在遇到问题的时候，注解处理就会崩溃， 开发者会从 javac 得到一堆难以理解的异常，因为它包含的是 FactoryProcessor 的堆栈信息。因此处理器包含 `Messager` 类，它会打印出友好的错误信息，并且可以定位到发生错误的地方。在现代 IDE 比如 Intellij 里面，第三方开发者可以直接点击错误信息，IDE 就会跳转到引发异常的源代码那一行。

回到 `process()`方法，如果我们检测到用户使用 @Factory 注解了非 class 元素，我们会抛出一个错误信息：

```java
public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {

    for (Element annotatedElement : roundEnv.getElementsAnnotatedWith(Factory.class)) {

      // Check if a class has been annotated with @Factory
      if (annotatedElement.getKind() != ElementKind.CLASS) {
        error(annotatedElement, "Only classes can be annotated with @%s",
            Factory.class.getSimpleName());
        return true; // Exit processing
      }

      ...
}

private void error(Element e, String msg, Object... args) {
    messager.printMessage(
    	Diagnostic.Kind.ERROR,
    	String.format(msg, args),
    	e);
  }
}
```

为了让 Messager 正常展示信息，处理器能够正常运行结束不 crash 是很重要的。这就是为什么在调用 `error()` 之后我们要 `return`。就像前面说的，如果我们不`return` 并且继续处理的话，很可能遇到一些异常导致注解处理崩溃，打印一些内部的堆栈信息出来而不是你想要展示的 Messager 错误信息。

## 数据模型定义

在我们继续检测使用 @Factory 注解的类是否符合我们前面提到的五个规则之前，我们需要引入一些数据结构以辅助我们继续开发。有些时候处理器或者问题的解决方案非常简单，开发者会把代码按照处理流程写在一块儿，但是你要知道注解处理器仍旧是一个 Java 程序，我们仍然要面向对象编程，仍然要注意编码风格和后期维护。

我们的 FactoryProcessor 非常简单，但是我们仍然想把一些信息结构化存储起来：

```java
public class FactoryAnnotatedClass {

  private TypeElement annotatedClassElement;
  private String qualifiedSuperClassName;
  private String simpleTypeName;
  private String id;

  public FactoryAnnotatedClass(TypeElement classElement) throws IllegalArgumentException {
    this.annotatedClassElement = classElement;
    Factory annotation = classElement.getAnnotation(Factory.class);
    id = annotation.id();

    if (StringUtils.isEmpty(id)) {
      throw new IllegalArgumentException(
          String.format("id() in @%s for class %s is null or empty! that's not allowed",
              Factory.class.getSimpleName(), classElement.getQualifiedName().toString()));
    }

    // Get the full QualifiedTypeName
    try {
      Class<?> clazz = annotation.type();
      qualifiedSuperClassName = clazz.getCanonicalName();
      simpleTypeName = clazz.getSimpleName();
    } catch (MirroredTypeException mte) {
      DeclaredType classTypeMirror = (DeclaredType) mte.getTypeMirror();
      TypeElement classTypeElement = (TypeElement) classTypeMirror.asElement();
      qualifiedSuperClassName = classTypeElement.getQualifiedName().toString();
      simpleTypeName = classTypeElement.getSimpleName().toString();
    }
  }

  /**
   * Get the id as specified in {@link Factory#id()}.
   * return the id
   */
  public String getId() {
    return id;
  }

  /**
   * Get the full qualified name of the type specified in  {@link Factory#type()}.
   *
   * @return qualified name
   */
  public String getQualifiedFactoryGroupName() {
    return qualifiedSuperClassName;
  }


  /**
   * Get the simple name of the type specified in  {@link Factory#type()}.
   *
   * @return qualified name
   */
  public String getSimpleFactoryGroupName() {
    return simpleTypeName;
  }

  /**
   * The original element that was annotated with @Factory
   */
  public TypeElement getTypeElement() {
    return annotatedClassElement;
  }
}
```

代码很长，但是最重要的部分位于构造函数里面：

```java
Factory annotation = classElement.getAnnotation(Factory.class);
id = annotation.id(); // Read the id value (like "Calzone" or "Tiramisu")

if (StringUtils.isEmpty(id)) {
    throw new IllegalArgumentException(
          String.format("id() in @%s for class %s is null or empty! that's not allowed",
              Factory.class.getSimpleName(), classElement.getQualifiedName().toString()));
    }
```

这里我们获取到了 @Factory 注解实例，并检测 id 是否为空，如果为空就抛出一个异常。你可能会觉得迷惑，因为我们前面提到了我们不会抛出异常，而是使用 __Messager__，实际上两者并不矛盾，因为我们在 `process()`方法里面会捕捉这个异常，稍后就会看到。我们这么做有两个原因：

1. 我想要想你展示在注解处理器的开发和其他 Java 程序并没有很大区别，抛异常在 Java 中是一个很不错的操作；
2. 如果我们想要在 **FactoryAnnotatedClass** 中打印消息，就需要把 Messager 传进去，并且和 "Error Handling" 一节中提到的那样，让 `process()` 正常结束，否则不能正确打印消息，那样的话 **FactoryAnnotatedClass** 必须告知 `process()` 方法一个错误发生了。而做到这样最简单的办法之一就是抛出一个异常，让 `process()` 捕捉到并正常处理掉；

下面我们想要获取 @Factory 的 type 字段：

```java
try {
      Class<?> clazz = annotation.type();
      qualifiedGroupClassName = clazz.getCanonicalName();
      simpleFactoryGroupName = clazz.getSimpleName();
} catch (MirroredTypeException mte) {
      DeclaredType classTypeMirror = (DeclaredType) mte.getTypeMirror();
      TypeElement classTypeElement = (TypeElement) classTypeMirror.asElement();
      qualifiedGroupClassName = classTypeElement.getQualifiedName().toString();
      simpleFactoryGroupName = classTypeElement.getSimpleName().toString();
}
```

这里有点小微妙，因为获取到的变量是 **java.lang.Class** 的，这就意味着这确实是一个类对象了。但是注解处理又是在源码编译之前进行的，所以我们得考虑以下两种情况：

1. **类已经被编译：** 这种情况发生在第三方的 .jar 包里面包含了被 @Factory 注解的 .class 文件，这个时候我们就可以和代码中 try 代码块里面一样直接获取 class；
2. **类没有被编译：** 这种情况发生在使用 @Factory 注解了源码，尝试直接获取 Class 会抛出 **MirroredTypeException** 异常。幸运的是 **MirroredTypeException** 包含了一个代表着我们尚未编译的类的 **TypeMirror** 对象，由于我们知道它一定是 class 类型的（前面检测过），我们可以直接把它转换成 **DeclaredType** ，并获取它的 **TypeElement** 来得到全限定名称；

好了，现在我们需要另外一个叫做 **FactoryGroupedClasses** 的数据结构来对 **FactoryAnnotatedClasses** 进行分组：

```java
public class FactoryGroupedClasses {

  private String qualifiedClassName;

  private Map<String, FactoryAnnotatedClass> itemsMap =
      new LinkedHashMap<String, FactoryAnnotatedClass>();

  public FactoryGroupedClasses(String qualifiedClassName) {
    this.qualifiedClassName = qualifiedClassName;
  }

  public void add(FactoryAnnotatedClass toInsert) throws IdAlreadyUsedException {

    FactoryAnnotatedClass existing = itemsMap.get(toInsert.getId());
    if (existing != null) {
      throw new IdAlreadyUsedException(existing);
    }

    itemsMap.put(toInsert.getId(), toInsert);
  }

  public void generateCode(Elements elementUtils, Filer filer) throws IOException {
	...
  }
}
```

如上，它其实只是一个 `Map<String, FactoryAnnotatedClass>` ，这个 map 是从 @Factory.id() 映射到 FactoryAnnotatedClass 的，我们使用 map 的原因是为了确保 id 是唯一的，map 检索起来很方便。**generateCode()** 将会被调用来产生 Factory 的代码（稍后讨论）。

## 规则检测匹配

让我们继续实现 `process()` 方法。接下来我们需要检测的是：被注解的类是否有一个 public 的无参构造函数，是否不是一个抽象类，是否继承于 type 指定的二类，是否是一个 public 的类：

```java
public class FactoryProcessor extends AbstractProcessor {

  @Override
  public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {

    for (Element annotatedElement : roundEnv.getElementsAnnotatedWith(Factory.class)) {

      ...

      // We can cast it, because we know that it of ElementKind.CLASS
      TypeElement typeElement = (TypeElement) annotatedElement;

      try {
        FactoryAnnotatedClass annotatedClass =
            new FactoryAnnotatedClass(typeElement); // throws IllegalArgumentException

        if (!isValidClass(annotatedClass)) {
          return true; // Error message printed, exit processing
         }
       } catch (IllegalArgumentException e) {
        // @Factory.id() is empty
        error(typeElement, e.getMessage());
        return true;
       }

   	   ...
   }


 private boolean isValidClass(FactoryAnnotatedClass item) {

    // Cast to TypeElement, has more type specific methods
    TypeElement classElement = item.getTypeElement();

    if (!classElement.getModifiers().contains(Modifier.PUBLIC)) {
      error(classElement, "The class %s is not public.",
          classElement.getQualifiedName().toString());
      return false;
    }

    // Check if it's an abstract class
    if (classElement.getModifiers().contains(Modifier.ABSTRACT)) {
      error(classElement, "The class %s is abstract. You can't annotate abstract classes with @%",
          classElement.getQualifiedName().toString(), Factory.class.getSimpleName());
      return false;
    }

    // Check inheritance: Class must be childclass as specified in @Factory.type();
    TypeElement superClassElement =
        elementUtils.getTypeElement(item.getQualifiedFactoryGroupName());
    if (superClassElement.getKind() == ElementKind.INTERFACE) {
      // Check interface implemented
      if (!classElement.getInterfaces().contains(superClassElement.asType())) {
        error(classElement, "The class %s annotated with @%s must implement the interface %s",
            classElement.getQualifiedName().toString(), Factory.class.getSimpleName(),
            item.getQualifiedFactoryGroupName());
        return false;
      }
    } else {
      // Check subclassing
      TypeElement currentClass = classElement;
      while (true) {
        TypeMirror superClassType = currentClass.getSuperclass();

        if (superClassType.getKind() == TypeKind.NONE) {
          // Basis class (java.lang.Object) reached, so exit
          error(classElement, "The class %s annotated with @%s must inherit from %s",
              classElement.getQualifiedName().toString(), Factory.class.getSimpleName(),
              item.getQualifiedFactoryGroupName());
          return false;
        }

        if (superClassType.toString().equals(item.getQualifiedFactoryGroupName())) {
          // Required super class found
          break;
        }

        // Moving up in inheritance tree
        currentClass = (TypeElement) typeUtils.asElement(superClassType);
      }
    }

    // Check if an empty public constructor is given
    for (Element enclosed : classElement.getEnclosedElements()) {
      if (enclosed.getKind() == ElementKind.CONSTRUCTOR) {
        ExecutableElement constructorElement = (ExecutableElement) enclosed;
        if (constructorElement.getParameters().size() == 0 && constructorElement.getModifiers()
            .contains(Modifier.PUBLIC)) {
          // Found an empty constructor
          return true;
        }
      }
    }

    // No empty constructor found
    error(classElement, "The class %s must provide an public empty default constructor",
        classElement.getQualifiedName().toString());
    return false;
  }
}
```

我们在这里添加了一个`isValidClass()` 方法，它检测了我们的规则是否被遵守：

* 类必须是 public 的：`classElement.getModifiers().contains(Modifier.PUBLIC)`
* 类不能是抽象的：`classElement.getModifiers().contains(Modifier.ABSTRACT)`
* 类必须是 **@Factoy.type()** 指明的类的子类：首先我们使用 `elementUtils.getTypeElement(item.getQualifiedFactoryGroupName())` 来创建一个 Class（@Factory.type()）的 Element（译者注：示例中就是 Meal.class）。是的，通过全限定类名就能创建一个 TypeElement。然后我们判断它是一个接口还是一个类：`superClassElement.getKind() == ElementKind.INTERFACE`。所以这里有两种情况：如果是接口，则 `classElement.getInterfaces().contains(superClassElement.asType())`，如果是类，我们需要通过调用 `currentClass.getSuperclass()`来扫描类的继承树。注意：也可以通过 `typeUtils.isSubtype()`来检查；
* 类必须有一个 public 的空构造函数：我们遍历所有的子元素，检查是否满足一下条件的元素：**ElementKind.CONSTRUCTOR**、 **Modifier.PUBLIC** 和 **constructorElement.getParameters().size() == 0**；

如果以上条件全部被满足，**isValidClass()** 会返回 true，否则它会打印错误信息，并且返回 false。

## 注解类分组

完成 `isValidClass` 的检测之后，我们会把 **FactoryAnnotatedClass** 按照如下方式添加到 **FactoryGroupedClasses**：

```java
 public class FactoryProcessor extends AbstractProcessor {

   private Map<String, FactoryGroupedClasses> factoryClasses =
      new LinkedHashMap<String, FactoryGroupedClasses>();


 @Override
  public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {

      ...

      try {
        FactoryAnnotatedClass annotatedClass =
            new FactoryAnnotatedClass(typeElement); // throws IllegalArgumentException

          if (!isValidClass(annotatedClass)) {
          return true; // Error message printed, exit processing
        }

        // Everything is fine, so try to add
        FactoryGroupedClasses factoryClass =
            factoryClasses.get(annotatedClass.getQualifiedFactoryGroupName());
        if (factoryClass == null) {
          String qualifiedGroupName = annotatedClass.getQualifiedFactoryGroupName();
          factoryClass = new FactoryGroupedClasses(qualifiedGroupName);
          factoryClasses.put(qualifiedGroupName, factoryClass);
        }

        // Throws IdAlreadyUsedException if id is conflicting with
        // another @Factory annotated class with the same id
        factoryClass.add(annotatedClass);
      } catch (IllegalArgumentException e) {
        // @Factory.id() is empty --> printing error message
        error(typeElement, e.getMessage());
        return true;
      } catch (IdAlreadyUsedException e) {
        FactoryAnnotatedClass existing = e.getExisting();
        // Already existing
        error(annotatedElement,
            "Conflict: The class %s is annotated with @%s with id ='%s' but %s already uses the same id",
            typeElement.getQualifiedName().toString(), Factory.class.getSimpleName(),
            existing.getTypeElement().getQualifiedName().toString());
        return true;
      }
    }

    ...
 }
```

## 代码生成

至此，我们已经收集了所有被 **@Factory** 注解的类，并转换成 **FactoryAnnotatedClass** 存储，同时也通过 **FactoryGroupedClasses** 进行了分组。现在我们就要为每一个 Factory 生成 java 文件了：

```java
@Override
public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {

	...

  try {
        for (FactoryGroupedClasses factoryClass : factoryClasses.values()) {
          factoryClass.generateCode(elementUtils, filer);
        }
    } catch (IOException e) {
        error(null, e.getMessage());
    }

    return true;
}
```

写 Java 文件的过程和在 Java 中写其他文件差不多，我们使用 **Filer** 提供的 **Writer** 来进行。我们可以像字符串拼接一样来生成代码，但是幸运的是，Square，一个开源了很多优质项目的公司，为我们提供了一个生成 Java 代码的库：[JavaWriter](https://github.com/square/javawriter)：

```java
public class FactoryGroupedClasses {

  /**
   * Will be added to the name of the generated factory class
   */
  private static final String SUFFIX = "Factory";

  private String qualifiedClassName;

  private Map<String, FactoryAnnotatedClass> itemsMap =
      new LinkedHashMap<String, FactoryAnnotatedClass>();

	...

  public void generateCode(Elements elementUtils, Filer filer) throws IOException {

    TypeElement superClassName = elementUtils.getTypeElement(qualifiedClassName);
    String factoryClassName = superClassName.getSimpleName() + SUFFIX;

    JavaFileObject jfo = filer.createSourceFile(qualifiedClassName + SUFFIX);
    Writer writer = jfo.openWriter();
    JavaWriter jw = new JavaWriter(writer);

    // Write package
    PackageElement pkg = elementUtils.getPackageOf(superClassName);
    if (!pkg.isUnnamed()) {
      jw.emitPackage(pkg.getQualifiedName().toString());
      jw.emitEmptyLine();
    } else {
      jw.emitPackage("");
    }

    jw.beginType(factoryClassName, "class", EnumSet.of(Modifier.PUBLIC));
    jw.emitEmptyLine();
    jw.beginMethod(qualifiedClassName, "create", EnumSet.of(Modifier.PUBLIC), "String", "id");

    jw.beginControlFlow("if (id == null)");
    jw.emitStatement("throw new IllegalArgumentException(\"id is null!\")");
    jw.endControlFlow();

    for (FactoryAnnotatedClass item : itemsMap.values()) {
      jw.beginControlFlow("if (\"%s\".equals(id))", item.getId());
      jw.emitStatement("return new %s()", item.getTypeElement().getQualifiedName().toString());
      jw.endControlFlow();
      jw.emitEmptyLine();
    }

    jw.emitStatement("throw new IllegalArgumentException(\"Unknown id = \" + id)");
    jw.endMethod();

    jw.endType();

    jw.close();
  }
}
```

Update：替换成了[JavaPoet](https://github.com/square/javapoet) 库。

## 注解处理轮次

注解处理可能不止一轮，官方文档指出的处理流程如下：

> 注解处理会有好多轮，在每一次的处理过程中，注解处理器都会用来处理前一轮产生的注解。第一轮注解处理输入的文件是整个工具的初始输入，初始输入可以当做注解处理的第 0 轮注解处理的输出。

【译者注】这句话的意思比较绕，简单来说就是，每一轮注解处理完成之后，可能会产生新的注解文件（比如我们产生的 Factory Java 代码可能也会带有注解），那么 javac 会继续调用注解处理器去处理这些生成的文件。

简单定义如下：一轮处理指的是调用一次注解处理器的`process()`方法。在我们的例子中：**FactoryProcessor** 只会被实例化一次，但是如果新的源文件被创建，则 `process()` 方法会被调用多次。听起来有点奇怪是不是？背后的原因就是，生成的源文件可能也会包含 **@Factory** 注解，同样需要 **FactoryProcessor** 再次处理。

举个🌰，**PizzaStore** 的例子中就会被处理三轮：

| Round | Input                                    | Output           |
| ----- | ---------------------------------------- | ---------------- |
| 1     | CalzonePizza.java Tiramisu.javaMargheritaPizza.javaMeal.javaPizzaStore.java | MealFactory.java |
| 2     | MealFactory.java                         | --- none ---     |
| 3     | --- none ---                             | --- none ---     |

我在这里解释处理次数还有一个原因，如果你看我们的 **FactoryProcessor** 代码，你会发现我们收集的数据都被存储在私有字段 **Map<String, FactoryGroupedClasses> factoryClasses** 里面，第一轮里面我们检测到 MagheritaPizza, CalzonePizza 和 Tiramisu 并产生了 MealFactory.java 文件，在第二轮我们把 MealFactory 当成输入，由于在 MealFactory 中没有使用 **@Factory** 注解，所以不会收集到数据，这时我们不会想要产生一个错误，但实际上我们遇到了：**Attempt to recreate a file for type com.hannesdorfmann.annotationprocessing101.factory.MealFactory**。

产生这个问题的原因就是我们没有去清理 **factoryClasses**，这意味着在第二轮中 `process()` 方法依然持有第一轮收集到的数据，并想产生和第一轮一样的文件。在这个例子中，我们知道只有第一轮会检测到 **@Factory** 注解的类，因此可以像下面这样进行简单的修复：

```java
@Override
public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
	try {
      for (FactoryGroupedClasses factoryClass : factoryClasses.values()) {
        factoryClass.generateCode(elementUtils, filer);
      }

      // Clear to fix the problem
      factoryClasses.clear();

    } catch (IOException e) {
      error(null, e.getMessage());
    }

	...
	return true;
}
```

我知道有其他办法来解决这个问题，比如设置标志位，关键在于记住：__注解处理是有多轮的，你不能去覆盖或者重新创建已经存在的源文件。__

## 注解和处理器分离

如果你已经看了 [git 仓库](https://github.com/sockeqwe/annotationprocessing101/tree/master/factory) 中 Factory 处理器的实现，你会发现我们已经把代码分成了两个 maven 的 module，这样做的原因是让我们示例的使用者可以在自己的项目依赖里面只包含 annotation，而处理器的依赖只在编译期。觉得困惑？如果我们只有一个产物（可以理解为一个 module，最终产生一个 jar 包），那么使用 Factory 注解处理器的开发者就必须在项目中同时包含 **@Factory** 注解和 **FactoryProcessor** 的代码。我非常确信别人是不想把处理器编译进去的（【译者注】这里涉及到依赖的 scope 概念，可以参见：[Dependency Scope](http://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html)）。如果你是一个 Android 开发者，你可能听过 65K 方法限制（android 的 .dex 文件只能包含 65000 个方法）。如果你在 FactoryProcessor 中使用了 guava，并且把注解和处理器打包在一起，那么 android 的 apk 就会包含 guava 这个库，而 guava 包含越 20000 个方法，因此注解和注解处理器的分离意义重大。

## 生成类的实例化

如前面在 **PizzaStore** 例子中所见，类 **MealFactory** 是一个和其他手写的类一样的 java 类。与此同时，你需要手动实例化它（和其他 Java 对象一样）：

```java
public class PizzaStore {
  private MealFactory factory = new MealFactory();

  public Meal order(String mealName) {
    return factory.create(mealName);
  }
  ...
}
```

如果你是一个 android 开发者，你应该会很熟悉一个注明的注解处理器 [ButterKnife](http://jakewharton.github.io/butterknife/)，使用 [ButterKnife](http://jakewharton.github.io/butterknife/) 你可以通过  **@InjectView** 注解 android 的 View 属性，ButterKnifeProcessor 会产生一个 **MyActivityViewInjector()** 类，开发者可以使用 **Butterknife.inject(activity)** 来进行生效关联，在 ButterKnife 内部会使用反射来实例化 **MyActivity$$ViewInjector()**:

```java
try {
	Class<?> injector = Class.forName(clsName + "$$ViewInjector");
} catch (ClassNotFoundException e) { ... }
```

但是反射不是很慢吗？我们使用注解来生成代码不会遇到很多反射性能问题吗？是的，反射会带来性能问题，但是注解加快了开发速度，因为开发者不需要再手动实例化对象。ButterKnife 使用一个 HashMap 来缓存初始化后的对象，所以当 **MyActivityViewInjector** 需要创建的时候就会先从 HashMap 中检索。

[FragmentArgs](https://github.com/sockeqwe/fragmentargs) 和 BuffterKnife 类似，它使用反射来初始化原本开发者需要手动实例化的对象。FragmentArgs 在注解处理过程中会生成一个特殊的 ”lookup“ HashMap 类，所以整个 FragmentArgs 库只会在一开始的时候运行一次反射来实例化这个特殊的 HashMap 类，之后的运行就是简单的原生 Java 代码了。

总的来说，在反射和注解带来的易用性之间寻找权衡完全取决于你（注解处理器的开发者）。

## 结论

我希望你现在对注解处理已经有了一个比较深的了解，我必须再次声明：注解处理是一个非常强大的工具，可以大大减少写模板代码的数量。另外我还想说的是使用注解处理器你可以做比例子中更多更复杂的事情，比如泛型擦除。因为注解处理发生在类型擦除之前。如你所见，通常注解处理器需要处理两个问题：第一，如果你想在其他类里面使用 ElementUtils, TypeUtils 和 Messager，你必须把他们当做参数传递过去，在 [AnnotatedAdapter](https://github.com/sockeqwe/AnnotatedAdapter) （我为 android 写的一个注解处理器）里面，我尝试使用 Dagger（依赖注入）来解决它，对于一个简单的处理器来说，可能有点小题大做，但它确实有效；第二，你必须查询  **Elements**，和前面说的一样，处理元素的过程可以看成是解析 XML 或者 HTML，对于 HTML 而言你可以使用 JQuery，而在注解处理这块如果有类似 JQuery 的库将会非常牛逼，如果你知道有这样的库，欢迎在下面评论。

__请注意：FactoryProcessor 的部分代码是不完善的，如果你想基于FactoryProcessor 来写你自己的注解处理器，不要复制粘贴这些问题，你应该从一开始就避免他们。 __

> PS：作者原本是打算写 Annotation Processing 102 的，内容是注解处理器的单元测试，但是很遗憾，到现在博客上也没有该篇文章。