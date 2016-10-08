---
layout: post
title: scala type system
categories: [scala]
keywords: scala, type
---

## FAQ

1: 讲一下多态, 分别从技术上和设计上讲解

技术上可以分成 编译时刻的多态和运行时刻的多态, 而 scala 还有另外一种多态

设计上, 多态有什么好处

2: CanBuildFrom

两个意义, 第一个意义是是否可以转换, 第二个意义是期望转化成什么样子


## 里氏替换原则

派生类（子类）对象能够替换其基类（超类）对象被使用

其他原则 SOLID

* 单一功能原则 (S)
* 开闭原则 (O)
* Liskov代换原则 (L)
* 接口隔离原则 (I)
* 依赖反转原则 (D)

发生替换的位置可能是函数的接受者, 也可能是函数参数

**单一功能原则**（Single responsibility principle）

规定每个类都应该有一个单一的功能，并且该功能应该由这个类完全封装起来。
所有它的（这个类的）服务都应该严密的和该功能平行（功能平行，意味着没有依赖）。

马丁把功能（职责）定义为：“改变的原因”，并且总结出一个类或者模块应该有且只有一个改变的原因。一个具体的例子就是，
想象有一个用于编辑和打印报表的模块。这样的一个模块存在两个改变的原因。第一，报表的内容可以改变（编辑）。
第二，报表的格式可以改变（打印）。这两方面会的改变因为完全不同的起因而发生：一个是本质的修改，一个是表面的修改。
单一功能原则认为这两方面的问题事实上是两个分离的功能，因此他们应该分离在不同的类或者模块里。把有不同的改变原因的事物耦合在一起的设计是糟糕的。

保持一个类专注于单一功能点上的一个重要的原因是，它会使得类更加的健壮。继续上面的例子，如果有一个对于报表编辑流程的修改，
那么将存在极大的危险性，打印功能的代码会因此不工作，假使这两个功能存在于同一个类中的话。

**开闭原则**

在面向对象编程领域中，开闭原则规定“软件中的对象（类，模块，函数等等）应该对于扩展是开放的，但是对于修改是封闭的”

开闭原则被广泛的重新定义由于抽象化接口的使用，在这中间实现可以被改变，多种实现可以被创建，并且多态化的替换不同的实现。

哪些设计模式都会为了开闭原则的, 好像都是吧。命令模式不算。

**接口隔离原则 interface-segregation principles**

接口隔离原则(ISP)拆分非常庞大臃肿的接口成为更小的和更具体的接口，这样客户将会只需要知道他们感兴趣的方法。
这种缩小的接口也被称为角色接口（role interfaces）。[2]接口隔离原则(ISP)的目的是系统解开耦合，从而容易重构，更改和重新部署。
接口隔离原则是在SOLID (面向对象设计)中五个面向对象设计(OOD)的原则之一，类似于在GRASP (面向对象设计)中的高内聚性。

在面向对象设计中，接口(interface)提供了便于代码在概念上解释的抽象层，并创建了避免依赖的一个屏障。

**依赖反转原则 Dependency inversion principle, DIP**

高层次的组件可能是汽车, 低层次组件是轮子

是指一种特定的解耦(传统的依赖关系创建在高层次上，而具体的策略设置则应用在低层次的模块上)形式，使得高层次的模块不依赖于低层次的模块的实现细节,
依赖关系被颠倒(反转), 从而使得低层次模块依赖于高层次模块的需求抽象。

**该原则规定:**

1. 高层次的模块不应该依赖于低层次的模块，两者都应该依赖于抽象接口。
2. 抽象接口不应该依赖于具体实现。而具体实现则应该依赖于抽象接口。

该原则颠倒了一部分人对于面向对象设计的认识方式。如高层次和低层次对象都应该依赖于相同的抽象接口

在传统的应用架构中，低层次的组件设计用于被高层次的组件使用，这一点提供了逐步的构建一个复杂系统的可能。
在这种结构下，高层次的组件直接依赖于低层次的组件去实现一些任务。这种对于低层次组件的依赖限制了高层次组件被重用的可行性。

依赖反转原则的目的是把高层次组件从对低层次组件的依赖中解耦出来，这样使得重用不同层级的组件实现变得可能。
把高层组件和低层组件划分到不同的包/库（在这些包/库中拥有定义了高层组件所必须的行为和服务的接口，并且存在高层组件的包）中的
方式促进了这种解耦。由于低层组件是对高层组件接口的具体实现，因此低层组件包的编译是依赖于高层组件的，这颠倒了传统的依赖关系。
众多的设计模式，比如插件，服务定位器或者依赖反转，则被用来在**运行时**把指定的低层组件实现提供给高层组件。

应用依赖反转原则同样被认为是应用了适配器模式，例如：高层的类定义了它自己的适配器接口（高层类所依赖的抽象接口）。
被适配的对象同样依赖于适配器接口的抽象（这是当然的，因为它实现了这个接口），同时它的实现则可以使用它自身所在低层模块的代码。
通过这种方式，高层组件则不依赖于低层组件，因为它（高层组件）仅间接的通过调用适配器接口多态方法使用了低层组件，
而这些多态方法则是由被适配对象以及它的低层模块所实现的。

## 什么是静态类型

编译器可以静态地（在编译时）验证程序是合理的。也就是说，如果值（在运行时）不符合程序规定的约束，编译将失败

## Variance(变性)

为什么 java 没有协变逆变的说法, 这是因为在 java 中函数不作为变量(逆变), 且 java 类型往往是 mutable 的(协变往往是 immutable的)

当一个类型 T' 是 T 的子类型时, Container[T'] 是不是 Container[T] 的子类型?

在 scala 中, 根据 Container 的不同声明, 协变, 逆变和不变给出不同的答案
 
首先是 **协变**, 它认为 Container[T'] 是 Container[T] 的子类型

```scala
class Container[+T]


val cv: Container[AnyRef] = new Container[String]
```


**immutable should be covariance, mutable should invariance**

```scala
// 不可变
val dogs: List[Dogs] = List(new Dog)
val animals: List[Animal] = dogs

val moreAnimals = (new Cat) :: animals 

// 可变, java
List<Dog> dogs = new List(new Dog)
List<Animal> animals = dogs
animals.add(new Cat)

dogs.bark() // cat cannot bark
```
从上面的例子看出, 可变不能是 covariance 的。因为 immutable 每次都返回一个新的东西, 不会改变原始的东西, 所以没有问题

要注意variance并不会被继承，父类声明为variance，子类如果想要保持，仍需要声明

covariance 对类型定义的要求:

```scala
class MyList[+T] {
  def  addMore[B >: T](b: B): MyList[B] = {
    this
  }
}
```
这段代码不能让 B 是 T 的子类型, 原因忘记了(好像和 function 有关系) @todo

B 的类型是 T 的父类, 原因还是里氏替换原则, 里氏替换是说父类可以存在的地方, 都可以替换成子类

假如, MyList[Animal] addMore 参数是 Animal 的子类, 那么 

```scala
MyList[Animal].addMore(dog) -> MyList[Animal] // Ok

MyList[Cat].add(dog) -> MyList[Cat] // 替换为子类 MyList[Cat], 就不妥了
```

addMore 可以看成是 `Function[B, Container[]]` B 必须是 -B 的才可以


其次是 **逆变**

```scala
class Container[-T]
val cv: Container[String] = new Container[AnyRef]
```

用到逆变的一个典型例子就是函数特质

```scala
trait Function1[-T1, +R] extends AnyRef

// 比如
def tweet: Cat => List = x=> x.legs

def tweet2: Animal => ArrayList
```

tweet 就是 tweet2 的子类, 因为根据里氏替换原则, 凡是 tweet 能存在的地方, tweet2 都能存在。

java8 的函数似乎是没有 invariance 的

### 逆变点与协变点

[参考](http://hongjiang.info/scala-pitfalls-10/)

```scala
class In[+A]{ def fun(x:A){} } // wrong

error: covariant type A occurs in contravariant position in type A of value x
class In[+A]{def fun(x:A){}}

class In[-A]{ def fun(x:A){} } // correct
```

要解释清楚这个问题，需要理解协变点(covariant position) 和 逆变点(contravariant position)

首先，我们假设 class In[+A]{ def fun(x:A){} } 可以通过编译，那么对于 In[AnyRef] 和 In[String] 这两个父子类型来说，fun方法分别对应：

```scala
父类型 In[AnyRef] 中的方法 fun(x: AnyRef){}
子类型 In[String] 中的方法 fun(x: String){}
```

根据里氏替换原则，所有使用父类型对象的地方都可以换成子类型对象。现在问题就来了，假设这样使用了父类：

```scala
father.fun(notString)
child.fun(notString) // 失败，非String类型，不接受
```

之前父类型的 fun 可以接收 AnyRef类型的参数，是一个更广的范围。而子类型的 fun 却只能接收String这样更窄的范围。
显然这不符合里氏替换原则了，因为父类做的事情，子类不能完全胜任，只能部分满足，是无法代替父类型的。

所以要想符合里氏替换，子类型中的fun函数参数类型必须是父类型中函数参数的超类（至少跟父类型中的参数类型一致），这样才能满足父类中fun方法可以做的事情，子类中fun方法也都可以做。

正是因为需要符合里氏替换法则，方法中的参数类型声明时必须符合逆变（或不变），
以让子类方法可以接收更大的范围的参数(处理能力增强)；而不能声明为协变，子类方法可接收的范围是父类中参数类型的子集(处理能力减弱)。

上面说的是参数, 下面说的是返回值, 方法返回值的位置称为协变点(covariant position)

```scala
// 方法返回值类型可以是协变的
scala> class In[+A]{ def fun(): A = null.asInstanceOf[A] }
defined class In

// 方法返回值类型不能是逆变的
scala> class In[-A]{ def fun(): A = null.asInstanceOf[A] }
<console>:8: error: contravariant type A occurs in covariant position in type ()A of method fun
```

利用 Function 的思路来考虑

```scala
class In[+A]{ def fun(): A = null.asInstanceOf[A] }
Unit => A Function1[Unit, +A] 符合 Function 定义

class In[-A] { def fun(x: A) {} } 
我们完全可以把它看做一个函数类型，即 A => Unit 与 Function1[-A, Unit]等价，而

class In[+A]{ def fun(): A = null.asInstanceOf[A] }
则与 Function0[+A] 等价。
```

协变和逆变对函数签名的影响就是他们的函数参数都是 -R 的, 返回值都是 +R 的

## 边界

```scala
class ObjectPool[T <: Connection] // scala
class ObjectPool[T >: Nothing] // scala
class ObjectPool[Nothing <: T <: Connection] // scala

class ObjectPool<T extends Connection> // java
```

List 的 :: 定义

```scala
def ::[B >: A] (x: B): List[B] = new scala.collection.immutable.::(x, this)

val options: List[String] = List("")
1 :: options // 返回的是 List[Any], 我想这可能是因为发生了类型提示, 不然 int 不会是 String 的父类型

上面就无法恢复类型了

// java8 list 也支持 nil, cons 了

class List<E> {
   static <Z> List<Z> nil() { ... };
   static <Z> List<Z> cons(Z head, List<Z> tail) { ... };
   E head() { ... }
}
```




## 量化(存在类型)

```scala
def count(l: List[_]) = l.size

def count(l: List[T forSome { type T }]) = l.size // 这样写就麻烦了

def hashcodes(l: Seq[_ <: AnyRef]) = l map (_.hashCode)
```

## 视界 (view bound)

一个视界指定一个类型可以被“看作是”另一个类型, 往往通过隐式类型转换完成

```scala
implicit def strToInt(x: String) = x.toInt
val y: Int = "123"
math.max("123", 111)

// 存在 A 到 Int 的隐式类型转换, 假如本来就是 Int 也可以
class Container[A <% Int] { def addIt(x: A) = 123 + x }
```

方法可以通过隐含参数执行更复杂的类型限制。例如，List支持对数字内容执行sum，但对其他内容却不行。可是Scala的数字类型并不都共享一个超类，所以我们不能使用T <: Number
相反，要使之能工作，Scala的math库对适当的类型T 定义了一个隐含的Numeric[T]。 然后在List定义中使用它：

```scala
sum[B >: A](implicit num: Numeric[B]): B // context bound

```

如果你调用List(1,2).sum()，你并不需要传入一个 num 参数, 它是隐式设置的。但如果你调用List("whoop").sum()，它会抱怨无法设置num。


```
// 用于修饰 implicit 类型
A =:= B	A 必须和 B相等
A <:< B	A 必须是 B的子类
A <%< B	A 必须可以被看做是 B


class Container[A](value: A) { def addIt(implicit evidence: A =:= Int) = 123 + value }

(new Container(123)).addIt // correct
(new Container("123")).addIt // error

class Container[A](value: A) { def addIt(implicit evidence: A <%< Int) = 123 + value }
(new Container("123")).addIt // correct
```

```scala
def min[B >: A](implicit cmp: Ordering[B]): A = {
  if (isEmpty) throw new UnsupportedOperationException("empty.min")
  reduceLeft((x, y) => if (cmp.lteq(x, y)) x else y)
}
```

1. 集合中的元素并不是必须实现 Ordered 特质，但 Ordered 的使用仍然可以执行静态类型检查。
2. 无需任何额外的库支持，你也可以定义自己的排序：

## F 界多态性

解决泛型继承的问题

```scala
trait Container extends Ordered[Container]
    def compare(that: Container): Int

class MyContainer extends Container {
  def compare(that: MyContainer): Int
}
```

编译失败，因为我们对 Container 指定了Ordered特质，而不是对特定子类型指定的。

为了调和这一点，我们改用F-界的多态性。


```scala
trait Container[A <: Container[A]] extends Ordered[A]

class MyContainer extends Container[MyContainer] { 
  def compare(that: MyContainer) = 0 
}

(new MyContainer, new MyContainer, new MyContainer, new YourContainer).min // 还是会报错...
```

这个问题怎么解决呢? @todo

## implicit

```scala
// implicit method
implicit def int2double(n: Int): Double

// implict class
implicit class RichInt(n: Int)

// implicit object
implicit object NumberLikeInt extends NumberLike[Int]

// implicit parameter
def maxElementInList(elements: List[T])(implicit orderer: (T, T) -> Boolean): T
def maxElementInList(elements: List[T])(implicit order: Ordered[T]): T

// view bounds
def maxElementInList[T <% Ordered[T]](elements: List[T]): T

// context bounds
def foo[A](implicit x: Ordered[A])
def foo[A : Ordered[A]]
```

```
def maxElementList[T <% Ordered[T]](elements: List[T]): T
def maxElementList(elements: List[T])(implicit order: Ordered[T]): T
```

用一句话说 implicit parameter 的作用，`为泛型提供类型约束`。继续拿 Java 来作比较，java 对泛型的约束只有上界和下界，
scala 的类型系统则复杂的多，泛型除了提供对上下界的支持外还有对隐式类型的支持，隐式类型的约束要比上下界要松一些，
它不要求泛型 T 非得是某个 Number 的子类，只要 T 能隐式转化为 Number 类即可，上面已经说过，
无法通过直接修改 Library 中类扩充方法，其实还有一件事做不到，那就是无法为已有的类添加公共的父类，
这一点在 type class 具体讲。现在只要知道 implicit parameter 是说，存在 T 到 Ordered[T] 类的隐式转换，就可以调用 maxElementList。
因为这只是一种类型约束，在函数的实现中并不需要这个参数，所以可以用 <% 符号简写


### type class

```
def top10wordSpark(inputFilePath: String, sc: SparkContext)= {
    val lines = sc.textFile(inputFilePath).cache
    val wordCount =
      lines
        .flatMap(line => line.split(" "))
        .filter(validWord)
        .map(word => (word, 1))
        .reduceByKey((a, b) => a + b)
    wordCount.collect().sortWith(_._2 > _._2).take(10).foreach(println)
  }
```

scala 自带除 reduceByKey 以为的所有函数，scala 也有 aggregate 实现类似的功能，但是用起来不如 reduceByKey 舒服，
所以我们这里自己实现以下 reduceByKey 方法。仿照 spark reduceByKey 的定义

```scala
// spark reduceByKey
def reduceByKey(partitioner: Partitioner, func: (V, V) => V): RDD[(K, V)] =
  self.withScope {
    combineByKey[V]((v: V) => v, func, func, partitioner)
}

//our version
def reduceByKey[K, V](collection: Traversable[Tuple2[K, V]], func: (V, V) => V) = {
    collection
      .groupBy(_._1)
      .map { case (group: K, traversable) 
          => traversable.reduce { (a, b) => (a._1, func(a._2, b._2)) } }
}

val col = List((("some"),100), (("key"),100), (("some"),50))  
ReduceByKey1.reduceByKey(col, (a, b) => a + b) // Error
```

最后一段代码报错，因为编译器不认为泛型 T 拥有 + 函数。这种质疑是合理的，在使用时我们能保证泛型 V 总是 int, double, long 
等有 + 运算的类型，但是我们要把这个信息告诉编译器。

```
// 添加上界约束
def reduceByKey[K, V <: Addable](collection: Traversable[Tuple2[K, V]], func: (V, V) => V) = {
    collection
      .groupBy(_._1)
      .map { case (group: K, traversable) => traversable.reduce { (a, b) => (a._1, func(a._2, b._2)) } }
  }
trait Addable {
    def +
}  
```
long, int, double 的父类都是确定好的了，在他们的继承结构中，没有什么是像 NumbericLike 这种类的声明。我们要创造 NumbericLike 这种东西，使用 type class.

```scala
// 创建 type class 并提供常用类型的实现
trait Addable[T] {
  def add(x: T, y: T): T
}
object Addable {
  implicit object IntAddable extends Addable[Int] {
    def add(x: Int, y: Int) = x + y
  }
implicit object StringAddable extends Addable[String] {
    def add(x: String, y: String) = x + y
  }
}


// 重新定义 reduceByKey 方法，声明隐式类型转换的约束
def reduceByKey2[K, V](collection: Traversable[Tuple2[K, V]])(implicit numLike: Addable[V]) = {
    collection
        .groupBy(_._1)
        .map { case (group: K, traversable) => traversable.reduce { (a, b) => (a._1, numLike.add(a._2, b._2))}}
  }

```

这样，只要上下文存在 Addable[T] 的类型，类型 T 即可使用 reduceByKey 函数。在 object Addable 中声明了常用类型的 Addable 声明，
而伴生对象是 implicit 检索的最后一个，当没有用户自定义的 implicit 后，系统会用伴生对象里的声明。当然，如果 reduceByKey 还
想支持 -, *, / 运算的话，可以把 Addable 替换为 NumbericLike 声明。

### CanBuildFrom 

**list to map**

```scala
import scala.collection.breakOut
val map : Map[Int,String] = List("London", "Paris").map(x => (x.length, x))(breakOut)

def breakOut[From, T, To](implicit b : CanBuildFrom[Nothing, T, To]) =
  new CanBuildFrom[From, T, To] {
    def apply(from: From) = b.apply() 
    def apply() = b.apply()
  }
```


map 的实现原理

```scala
def map[B, That](f: A => B)(implicit bf: CanBuildFrom[Repr, B, That]): That = {
    def builder = { // extracted to keep method size under 35 bytes, so that it can be JIT-inlined
      val b = bf(repr)
      b.sizeHint(this)
      b
    }
    val b = builder
    for (x <- this) b += f(x)
    b.result
}
  
class Builder[-B, +Repr] { // Builder 看样是 mutable 的
	def +=(B): Builder[B, Repr]
	def result: Repr
	def mapResult(f: Repr => NewRepr): Builder[B, NewRepr] //what about this one
	def clear()
}  

trait CanBuildFrom[-From, -Elem, +To] {
    def apply(This): Builder[B, That]
}
  
```

Repr 是当前集合的类型, 比如 List, Map 什么的, That 是新集合的类型, 比如 Repr 是 List[Int] 时, That 可以是 Map

结合两段代码分析, 上面那段代码要求 b: CanBuildFrom[Nothing, T, To] 其实是放宽了要求, 把三个类型的约束转成了两个。
任何 CanBuildFrom[_, (String, String), To] 都是它的子类,相当于这里放宽了条件。
breakout 生成的 CanBuildFrom[List, (String, String), Map] 调用 apply 方法就会返回 Builder[(String, String), Map]

```scala
  def newBuilder[A, B]: Builder[(A, B), CC[A, B]] = new MapBuilder[A, B, CC[A, B]](empty[A, B])

  /** The standard `CanBuildFrom` class for maps.
   */
  class MapCanBuildFrom[A, B] extends CanBuildFrom[Coll, (A, B), CC[A, B]] {
    def apply(from: Coll) = newBuilder[A, B]
    def apply() = newBuilder
  }
```

breakout 返回的 Builder 很可能就是上面这个 MapBuilder, 这样就有了从 (, ) -> Map 之间的转化了

[what's break out](http://docs.scala-lang.org/tutorials/FAQ/breakout)


### 线性化 Linearization Order

```scala
class A {
  def foo() = "A"
}

trait B extends A {
  override def foo() = "B" + super.foo()
}

trait C extends B {
  override def foo() = "C" + super.foo()
}

trait D extends A {
  override def foo() = "D" + super.foo()
}

object LinearizationPlayground {
    def main(args: Array[String]) {

      var d = new A with D with C with B;
      println(d.foo) // CBDA????
  }    
}
```

[link](http://stackoverflow.com/questions/34242536/linearization-order-in-scala)

An intuitive way to reason about linearization is to refer to the construction order and to visualise the linear hierarchy.

You could think this way: the base class is constructed first; traits are constructed left-to-right because a trait on 
the right is added "later" and thus "overrides" the previous traits. However, to construct a trait, its base traits 
must be constructed first (obvious); and, of course, if a trait is constructed, is not reconstructed again. Now, 
the construction order is the reverse of the linearisation. Think of "base" traits/classes as higher in the linear 
hierarchy, and traits lower in the hierarchy as closer to the class/object which is the subject of the linearisation. 
The linearisation affects how `super´ is resolved in a trait: it will resolve to the closest base trait (but higher in 
the hierarchy!).

Linearisation of A with D with C with B is

```
(top of hierarchy) A (constructed first as base class)
linearisation of D
    A (not considered as A occurs before) // 因为 A 已经出现过了, 所以不再考虑
    D (D extends A)
linearisation of C
    A (not considered as A occurs before) 
    B (B extends A)
    C (C extends B)
linearisation of B
    A (not considered as A occurs before)
    B (not considered as B occurs before)
```

right most, deep most 的意义, 应该是这样, 底层的依赖是 B, deep most 所以要把 A 放到 B 的前面

然后看 C, C 应该放到继承链中的最上层, 但它与实际的继承关系反过来了, 所以应该下放到最底层

D, 它本该在最上层, 但是 A 是它的子类, 所以下放, 放到 A 的下面, 最终的结果还是 ADBC, 与原始的结论一样
