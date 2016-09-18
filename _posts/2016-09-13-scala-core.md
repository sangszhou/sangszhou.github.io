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

```
class ClassFIle {

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

```flatMap
def flatMap[B, That](f: A => GenTraversableOnce[B])(implicit bf: CanBuildFrom[Repr, B, That]): That = {
    def builder = bf(repr) // extracted to keep method size under 35 bytes, so that it can be JIT-inlined
    val b = builder
    for (x <- this) b ++= f(x).seq
    b.result
  }

def filter(p: A => Boolean): Repr = {
    val b = newBuilder
    for (x <- this)
      if (p(x)) b += x
    b.result
  }
  
def collect[B, That](pf: PartialFunction[A, B])(implicit bf: CanBuildFrom[Repr, B, That]): That = {
    val b = bf(repr)
    for (x <- this) if (pf.isDefinedAt(x)) b += pf(x)
    b.result
  }

def reduceLeft[B >: A](op: (B, A) => B): B = {
    if (isEmpty)
      throw new UnsupportedOperationException("empty.reduceLeft")

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
    onComplete {
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

object 方法

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