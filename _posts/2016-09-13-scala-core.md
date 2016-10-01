---
layout: post
title: scala 高阶函数与字节码
categories: [scala]
keywords: scala
---

## Trait 的多重继承

但声明一个只包含方法声明而不包含方法体的 trait 类, 它会编译成一个 java 接口

```scala
trait Emptry {
    def e: Int
}
```

生成的 .class 文件

```java
public interface Emptry {
    public abstract int e()
}
```

如果 trait 既有 abstract method 又有 concrete method, 那么将会生成两个 .class 文件,
一个接口类和一个包含代码的新类。但一个类继承这个 trait 时, trait 声明的变量将会被复制到这个类
文件中, 而定义在 trait 中的方法作为这个继承类的外观模式的方法, 这个类调用这个方法时, 将调用类的新类
中的对应方法。

javap -c Empty.class

```java
public abstract class Empty$class {
    public static int concrete(Empty)
        Code:
            0: iconst_2
            1: ireturn
    public static void $init$(Empty) { // 注意方法是 static 的
        Code:
            0: return
    }        
}
```

注意 Empty$class 的函数, 都是 Empty 类型的, 而不是自己的类型。这也说明了为什么是 $init$ 而不是构造函数, 因为构造函数的
参数是 this

```java
public interface Empty {
    public abstract int e();
    public abstract int concrete();
}
```

**继承**

```java
public class ConcreteEmpty extends Empty {
    public int concrete();
        Code:
            0: aload_0
            1: invokestatic #17 // Method demo/Empty$class.concrete:(Ldemo/Empty;)I
            4. ireturn
    
    public int e();
        Code:
            0: iconst_2
            1: ireturn
    
    public ConcreteEmpty(); // 构造函数
        Code:
            0: aload_0
            1: invokespecial #24  // Method java/lang/Object."<init>": ()V
            4: aload_0
            5: invokestatic  #28  // Method Empty$class.$init$:(Empty)V
            8: return 
}
```

从字节码可以看出, ConcreteEmpty 的 concrete 方法, 也就是 trait 中已经实现的方法, 它会调用另一个类型的实现


## 高阶函数

```scala
class ClassFile {

  def highOrder(f: Int => Int) = {
    f(1)
  }
}


public class ClassFIle {
  public int highOrder(scala.Function1<java.lang.Object, java.lang.Object>);
    Code:
       0: aload_1
       1: iconst_1
       2: invokeinterface #16,  2           // InterfaceMethod scala/Function1.apply$mcII$sp:(I)I
       7: ireturn

  public ClassFIle();
    Code:
       0: aload_0
       1: invokespecial #24                 // Method java/lang/Object."<init>":()V
       4: return
}
```


## List

@todo missing foldLeft 的实现

```scala
def foldLeft[B](z: B)(f: (B, A) => B): B = {
    var acc = z
    var these = this
    while (!these.isEmpty) {
      acc = f(acc, these.head)
      these = these.tail
    }
    acc
}
```

```scala
def flatMap[B, That](f: A => GenTraversableOnce[B])(implicit bf: CanBuildFrom[Repr, B, That]): That = {
    def builder = bf(repr) // extracted to keep method size under 35 bytes, so that it can be JIT-inlined
    val b = builder
    for (x <- this) b ++= f(x).seq
    b.result

def filter(p: A => Boolean): Repr = {
    val b = newBuilder
    for (x <- this)
      if (p(x)) b += x
    b.result
  
def collect[B, That](pf: PartialFunction[A, B])(implicit bf: CanBuildFrom[Repr, B, That]): That = {
    val b = bf(repr)
    for (x <- this) if (pf.isDefinedAt(x)) b += pf(x)
    b.result

def reduceLeft[B >: A](op: (B, A) => B): B = {
    if (isEmpty) throw new UnsupportedOperationException("empty.reduceLeft")

    var first = true
    var acc: B = 0.asInstanceOf[B]

    for (x <- self) {
      if (first) {
        acc = x
        first = false
      }
      else acc = op(acc, x)
    }
    acc
  }
```

## Future

```scala
def flatMap[S](f: T => Future[S])(implicit executor: ExecutionContext): Future[S] = {
    import impl.Promise.DefaultPromise
    val p = new DefaultPromise[S]() 
    onComplete { // this.onComplete
      case f: Failure[_] => p complete f.asInstanceOf[Failure[S]]
      case Success(v) => try f(v) match {
        // If possible, link DefaultPromises to avoid space leaks
        case dp: DefaultPromise[_] => dp.asInstanceOf[DefaultPromise[S]].linkRootOf(p)
        case fut => fut.onComplete(p.complete)(internalExecutor)
      } catch { case NonFatal(t) => p failure t }
    }
    p.future
  }

def map[S](f: T => S)(implicit executor: ExecutionContext): Future[S] = { // transform(f, identity)
    val p = Promise[S]()
    onComplete { v => p complete (v map f) }
    p.future
  }

def recover[U >: T](pf: PartialFunction[Throwable, U])(implicit executor: ExecutionContext): Future[U] = {
    val p = Promise[U]()
    onComplete { v => p complete (v recover pf) }
    p.future
  }

def recoverWith[U >: T](pf: PartialFunction[Throwable, Future[U]])(implicit executor: ExecutionContext): Future[U] = {
    val p = Promise[U]()
    onComplete {
      case Failure(t) => try pf.applyOrElse(t, (_: Throwable) => this).onComplete(p.complete)(internalExecutor) catch { case NonFatal(t) => p failure t }
      case other => p complete other
    }
    p.future
  }
```

**object Future 方法**

```scala
def sequence[A, M[_] <: TraversableOnce[_]](in: M[Future[A]])(implicit cbf: CanBuildFrom[M[Future[A]], A, M[A]], executor: ExecutionContext): Future[M[A]] = {
    in.foldLeft(Promise.successful(cbf(in)).future) {
      (fr, fa) => for (r <- fr; a <- fa.asInstanceOf[Future[A]]) yield (r += a)
    } map (_.result)
  }

def firstCompletedOf[T](futures: TraversableOnce[Future[T]])(implicit executor: ExecutionContext): Future[T] = {
    val p = Promise[T]()
    val completeFirst: Try[T] => Unit = p tryComplete _
    futures foreach { _ onComplete completeFirst }
    p.future
}

// 通过计数器知道是否全部遍历结束了
def find[T](futurestravonce: TraversableOnce[Future[T]])(predicate: T => Boolean)(implicit executor: ExecutionContext): Future[Option[T]] = {
    val futures = futurestravonce.toBuffer
    if (futures.isEmpty) Promise.successful[Option[T]](None).future
    else {
      val result = Promise[Option[T]]()
      val ref = new AtomicInteger(futures.size)
      val search: Try[T] => Unit = v => try {
        v match {
          case Success(r) => if (predicate(r)) result tryComplete Success(Some(r))
          case _ =>
        }
      } finally {
        if (ref.decrementAndGet == 0) {
          result tryComplete Success(None)
        }
      }

      futures.foreach(_ onComplete search)

      result.future
    }
  }
```


```scala
def apply[T](body: =>T)(implicit execctx: ExecutionContext): Future[T] = impl.Future(body)

def apply[T](body: =>T)(implicit executor: ExecutionContext): scala.concurrent.Future[T] = {
    val runnable = new PromiseCompletingRunnable(body)
    executor.prepare.execute(runnable)
    runnable.promise.future
}

class PromiseCompletingRunnable[T](body: => T) extends Runnable {
    val promise = new Promise.DefaultPromise[T]()
    override def run() = {
      promise complete {
        try Success(body) catch { case NonFatal(e) => Failure(e) }
      }
    }
  }
```

## 无参函数

无参方法的惯例是:

1. 方法没有参数
2. 方法不会改变可变状态(无副作用)

这个惯例支持统一访问原则(uniform access principal), 客户代码不应由属性是字段实现还是方法实现而受到影响

```scala
abstract class A { def a: Int }
class B extends A { val a = 1 } // 里面的 val 可以写成 var
```

当然 val 成员也可以在子类中被 override

```scala
class A { val a = 2}
clas B extends A { override val a = 3 }
```

但父类中成员声明为var则子类用val重写是不行的，因为var提供了getter/setter，而val只有getter:

如果一个类中，出现了同名的 成员变量和无参函数，则编译时会报错（有参则没有问题），这点与java不同。

java中有4个命名空间：

> 包, 类型, 方法, 字段

方法与字段是不同的命名空间，所以字段与方法同名是不会冲突的。


而scala中仅有2个命名空间：

> 值（字段/方法/包/单例）

> 类型（类/特质）

所以在scala可以实现用val重写无参方法这种事情。

不过把字段、方法、单例 放在同一个命名空间还好理解，但“包”也与它们在同一个命名空间是怎么回事？

scala里包与字段和方法共享相同命名空间是为了让你引入包，而不仅仅是引入类型名以及单例对象的字段和方法。这点也与java不同，java只能import一个包下的类。

```scala
import java.{util => u} 

class A {
  val a = new u.ArrayList[String](); 
  def u = 2 //命名冲突
}
```

原则上，scala中的函数都可以省略空括号，然而在调用的方法超出其调用者对象的属性时，推荐仍保持空括号。

比如，方法执行了I/O, 或有改变可变状态，或读取了不是调用者字段的var，总之无论是直接还是非直接的使用可变对象

### def fun = 1 和 def fun() = 1 的区别

```scala
def foo() = println("h")
foo // 等同于 foo()

def bar = println("hi")
bar // hi
bar() // error Unit does not take parameters
```

bar() 实际被翻译为 bar.apply(), 但是 bar 的返回值为 Unit, 他没有 apply 方法, 所以就报错了

```scala
class MyClass { def apply() = println("haha") }

def m = new MyClass
m // MyClass@xxx
m() // haha
```

### 字节码

```scala
class Foo {
  def bar() = 123;
}


class Foo {
  def bar = 123;
}
```

Foo, Bar 的字节码是一样的, 额外的信息放到了常量池里面
$ javap -verbose Foo.class

```
const #195 = Asciz      ScalaSig;
const #196 = Asciz      Lscala/reflect/ScalaSignature;;
const #197 = Asciz      bytes;
const #198 = Asciz      ^F^A!3A!^A^B^A^S\t\tb)^[7f^Y^Vtw\r^^5DQ^V^\7.^Z:^K^E\r!^
Q^A^B4jY^VT!!^B^D^B^UM^\^W\r\1tifdWMC^A^H^....
```

### Function0 和无参构造器

```scala
def withFunction(f: => Int): Unit = {
    f
}

def withFunction2(f: () => Int) = {
    f()
}

def withFunction3(f: () => Int): ()=> Int = {
    f
}

public void withFunction(scala.Function0<java.lang.Object>);
    Code:
       0: aload_1
       1: invokeinterface #16,  1           // InterfaceMethod scala/Function0.apply$mcI$sp:()I
       6: pop
       7: return

public int withFunction2(scala.Function0<java.lang.Object>);
    Code:
       0: aload_1
       1: invokeinterface #16,  1           // InterfaceMethod scala/Function0.apply$mcI$sp:()I
       6: ireturn
```

with Function 一般是用来接收 ```{xxx}``` 这种语法块的

对于 withFunction 函数, 即可接收 withFunction 又可接收 def m() 又可以接收 def m, 但是 withFunction2 仅接受 def m()

看字节码又都一样, 实在 confusing

### loan pattern

loan pattern 往往包括一个函数作为变量, 在函数体内创建资源, 资源作为函数的参数, 使用资源, 然后释放资源

```scala
def writeFile(fileName: File)(operation: PrintWriter => Unit) { 
    val writer = new PrintWriter(fileName)  // 贷出资源writer 
  
    try{ 
        operation(pw)   // 客户使用资源 
    }finally { 
        writer.close()  // 用完则释放被回收 
    } 
} 
```

### cake pattern

cake pattern 是依赖注入, scala 的答案, 其实它本身也很简单

名字的由来:

蛋糕有很多风味， 你可以根据你的需要增加相应的风味。依赖注入就是增加调料。

蛋糕有很多层 (layer)。如果你想要一个更大的蛋糕你就可以增加更多的层。


```java
abstract class BarUsingFooAble {
  def bar() = "bar calls foo: " + foo.foo()
  def foo:FooAble //abstract 
}

object Main {
  def main(args: Array[String]) {
    val fooable = new FooAble {}
    val barWithFoo = new BarUsingFooAble{
      def foo: FooAble = fooable 
    }
    println(barWithFoo.bar())
  }
}
```

```java
class BarUsingFooAble {
  this: FooAble => //see note #1 self-annotation
  def bar() = "bar calls foo: " + foo()
}

object Main {
  def main(args: Array[String]) {
    val barWithFoo = new BarUsingFooAble with FooAble
    println(barWithFoo.bar())
  }
}
```

```scala
trait UserRepositoryComponent {
  val userRepository:UserRepository
}

trait UserServiceComponent {
  this: UserRepositoryComponent =>
  val userService: UserService
}
```

这里使用 self-type annotation 声明 UserServiceComponent 需要 UserRepositoryComponent ( this: UserRepositoryComponent => )。 如果需要多个依赖，可以使用下面的格式:

```scala
object ComponentRegistry extends UserServiceComponent with UserRepositoryComponent {
  override val userRepository: UserRepository = new MockUserRepository
  override val userService: UserService = new UserService(userRepository)
}
```

这还有一个好处就是所有的对象都是 val 类型的。

