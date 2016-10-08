---
layout: post
title: Java new feature
categories: [java]
keywords: java
---

## Lambda and functional interface

```java
Arrays.asList( "a", "b", "d" ).forEach( e -> System.out.println( e ) );
```

the type of e is inferred by compiler. Alternatively, you may explicitly provide the 
type of parameter, wrapping the definition in brackets

```java
Arrays.asList( "a", "b", "d" ).forEach( ( String e ) -> System.out.println( e ) );
```

Lambdas may return a value. The type of the return value will be inferred by compiler. 
The return statement is not required if the lambda body is just a one-liner. The two code snippets below are equivalent:

```java
Arrays.asList( "a", "b", "d" ).sort( ( e1, e2 ) -> e1.compareTo( e2 ) );

Arrays.asList( "a", "b", "d" ).sort( ( e1, e2 ) -> {
    int result = e1.compareTo( e2 );
    return result;
} );
```

1.抽象类不可以多重继承，接口可以（无论是多重类型继承还是多重行为继承）；

2.抽象类和接口所反映出的设计理念不同。其实抽象类表示的是"is-a"关系，接口表示的是"like-a"关系；

3.接口中定义的变量默认是 public static final 型，且必须给其初值，所以实现类中不能改变其值；抽象类中的变量默认是 friendly 型，
  其值可以在子类中重新定义，也可以重新赋值。 

**Java 的 Lambda 表示式是把 e -> println(e) 改成 Function 对象, 而 java 可能是改成 functional interface 加上
Function 对象**

Language designers put a lot of thought on how to make already existing functionality lambda-friendly. 
As a result, the concept of functional interfaces has emerged. The function interface is an interface with 
just one single method. As such, it may be implicitly converted to a lambda expression.
 
The **java.lang.Runnable** and **java.util.concurrent.Callable** are two great examples of functional interfaces.

In practice, the functional interfaces are fragile: if someone adds just one another method to the interface definition, 
it will not be functional anymore and compilation process will fail.

To overcome this fragility and explicitly declare the intent of the interface as being functional, 
Java 8 adds special annotation @FunctionalInterface (all existing interfaces in Java library have 
been annotated with @FunctionalInterface as well). Let us take a look on this simple functional interface definition:

```java
@FunctionalInterface // 再加一个函数就会报错?
public interface Functional {
    void method();
} 
```

One thing to keep in mind: default and static methods do not break the functional interface contract and may be declared:

```java
@FunctionalInterface
public interface FunctionalDefaultMethods {
    void method();
    default void defaultMethod() {           
    }       
}
```

如果多继承产生了二义性, 那么编译器会报错的

## Interface's default and static method

```java
private interface Defaulable {
    // Interfaces now allow default methods, the implementer may or 
    // may not implement (override) them.
    default String notRequired() { 
        return "Default implementation"; 
    }        
}
        
private static class DefaultableImpl implements Defaulable {
}
    
private static class OverridableImpl implements Defaulable {
    @Override
    public String notRequired() {
        return "Overridden implementation";
    }
}
```

Another interesting feature delivered by Java 8 is that interfaces can declare (and provide implementation) of 
static methods. Here is an example.

```java
private interface DefaultableFactory {
    // Interfaces now allow static methods
    static Defaultable create( Supplier< Defaultable > supplier ) {
        return supplier.get();
    }
}
```

**原理**

Lambda表达式利用了类型推断（type inference）技术

内部类来实现Lambda表达式

```java
@FunctionalInterface
interface Print<T> {
  public void print(T x);
}

public class Lambda {   
  public static void PrintString(String s, Print<String> print) {
    print.print(s);
  }

  private static void lambda$0(String x) {
    System.out.println(x);
  }
  
  final class $Lambda$1 implements Print{
    @Override
    public void print(Object x) {
      lambda$0((String)x);
    }
  }
  
  public static void main(String[] args) {
    PrintString("test", new Lambda().new $Lambda$1());
  }
  
  // 原始的用法
  public static void main(String[] args) {
    PrintString("test", (x) -> System.out.println(x));
  }

}
```

## Method reference

The first type of method references is constructor reference with the syntax Class::new or alternatively, 
for generics, Class< T >::new. Please notice that the constructor has **no arguments**.

```java
final Car car = Car.create( Car::new );
final List< Car > cars = Arrays.asList( car );
```

The second type is reference to static method with the syntax Class::static_method. 
Please notice that the method accepts exactly one parameter of type Car.

```java
cars.forEach( Car::collide );
//static collide(car)
```

The third type is reference to instance method of arbitrary object of specific type with the syntax Class::method. Please notice, no arguments are accepted by the method.

```java
cars.forEach( Car::repair );
//car.repair
```

And the last, fourth type is reference to instance method of particular class instance the syntax instance::method. 
Please notice that method accepts exactly one parameter of type Car.

```java
final Car police = Car.create( Car::new );
cars.forEach( police::follow );
//police.follow(car)
```


## Optional

Optional is just a container: it can hold a value of some type T or just be null. It provides a 
lot of useful methods so the explicit null checks have no excuse anymore. Please refer to 
official Java 8 documentation for more details.

```java
Optional< String > fullName = Optional.ofNullable( null );
System.out.println( "Full Name is set? " + fullName.isPresent() );        
System.out.println( "Full Name: " + fullName.orElseGet( () -> "[none]" ) ); 
System.out.println( fullName.map( s -> "Hey " + s + "!" ).orElse( "Hey Stranger!" ) );

Optional< String > firstName = Optional.of( "Tom" );
System.out.println( "First Name is set? " + firstName.isPresent() );       
System.out.println( "First Name: " + firstName.orElseGet( () -> "[none]" ) );
System.out.println( firstName.map( s -> "Hey " + s + "!" ).orElse( "Hey Stranger!" ) );
```

## Streams

The newly added Stream API (java.util.stream) introduces real-world functional-style programming into 
the Java. This is by far the most comprehensive addition to Java library intended to make Java developers 
significantly more productive by allowing them to write effective, clean, and concise code.

```java
// Calculate total points of all active tasks using sum()
final long totalPointsOfOpenTasks = tasks
    .stream()
    .filter( task -> task.getStatus() == Status.OPEN )
    .mapToInt( Task::getPoints ) // static method with arg or member method without arg
    .sum();
        
System.out.println( "Total points: " + totalPointsOfOpenTasks );
```

### parallel

map 也可以完成 mapToInt 的用法, 那么什么时候用 mapToInt 什么时候 map 呢?

```java
// Calculate total points of all tasks
final double totalPoints = tasks
   .stream()
   .parallel()
   .map( task -> task.getPoints() ) // or map( Task::getPoints ) 
   .reduce( 0, Integer::sum );
    
System.out.println( "Total points (all tasks): " + totalPoints );
```

### grouping

```java
// Group tasks by their status
final Map< Status, List< Task > > map = tasks
    .stream()
    .collect( Collectors.groupingBy( Task::getStatus ) );
System.out.println( map );

// result
{CLOSED=[[CLOSED, 8]], OPEN=[[OPEN, 5], [OPEN, 13]]}
```

```java
// Calculate the weight of each tasks (as percent of total points) 
final Collection< String > result = tasks
    .stream()                                        // Stream< String >
    .mapToInt( Task::getPoints )                     // IntStream
    .asLongStream()                                  // LongStream
    .mapToDouble( points -> points / totalPoints )   // DoubleStream
    .boxed()                                         // Stream< Double >
    .mapToLong( weigth -> ( long )( weigth * 100 ) ) // LongStream
    .mapToObj( percentage -> percentage + "%" )      // Stream< String> 
    .collect( Collectors.toList() );                 // List< String > 
        
System.out.println( result );
```

## Date / Time

```java
// Get the system clock as UTC offset 
final Clock clock = Clock.systemUTC();
System.out.println( clock.instant() );
System.out.println( clock.millis() );

// Get the local date and local time
final LocalDate date = LocalDate.now();
final LocalDate dateFromClock = LocalDate.now( clock );
        
System.out.println( date );
System.out.println( dateFromClock );
        
// Get the local date and local time
final LocalTime time = LocalTime.now();
final LocalTime timeFromClock = LocalTime.now( clock );
        
System.out.println( time );
System.out.println( timeFromClock );
```

## JVM

The PermGen space is gone and has been replaced with Metaspace (JEP 122). The 
JVM options -XX:PermSize and –XX:MaxPermSize have been replaced by -XX:MetaSpaceSize and -XX:MaxMetaspaceSize respectively.

[link](https://www.infoq.com/articles/Java-PERMGEN-Removed)

The move to Metaspace was necessary since the PermGen was really hard to tune. There was a possibility that the metadata could move with every full garbage collection. Also, it was difficult to size the PermGen since the size depended on a lot of factors such as the total number of classes, the size of the constant pools, size of methods, etc.

Additionally, each garbage collector in HotSpot needed specialized code for dealing with metadata in the PermGen. 
Detaching metadata from PermGen not only allows the seamless management of Metaspace, but also allows for 
improvements such as simplification of full garbage collections and future concurrent de-allocation of class metadata.


