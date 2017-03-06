---
layout: post
title:  "AOP spring source code analysis"
date:   "2017-02-19 10:50:00"
categories: spring
keywords: spring
---

## 例子 1 在使用 New 的情况下实现 AOP

```java
public class TraceTest {

    public static void main(String args[]) {
        TraceTest test = new TraceTest();
        test.rpcCall();
    }

    // 虽然 intellij 没有给出提示，但是这个 Trace 还是成功的
    @Trace
    public void rpcCall() {
        System.out.println("call rpc");
    }

}

@Aspect
public class TraceAspect {
    @Pointcut("@annotation(Trace)")
    public void tracePointcutDefinition() {}

    @Around("@annotation(Trace) && execution(* *(..))")
    public Object aroundTrace(ProceedingJoinPoint pjp) throws Throwable {
        Object returnObj = null;

        try {
            System.out.println("before traced method started...");
            returnObj = pjp.proceed();

        } catch (Throwable throwable) {
            throw throwable;
        } finally {
            System.out.println("after traced method executed...");
        }

        return returnObj;
    }
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Trace {
}

//before traced method started...
//call rpc
//after traced method executed...
```

按说使用 Spring AOP 是需要容器来注入的，因为容器会自动的把需要注入的代码放到生成的代理类中，但是从上面的例子我们直接 New 了一个实例，但是这个 new 出来的实例就是有被织入的代码，那么它是如何实现的呢？

AOP 有编译时和运行时注入，上面这个例子在编译时已经注入完毕了，生成的 .class 文件已经是修改以后的了，所以不需要再使用容器，来看一下 pom 配置

```xml
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>aspectj-maven-plugin</artifactId>
    <version>1.7</version>
    <configuration>
        <complianceLevel>1.8</complianceLevel>
        <source>1.8</source>
        <target>1.8</target>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>compile</goal>
            </goals>
        </execution>
    </executions>
</plugin>


public void rpcCall();
  Code:
     0: getstatic     #46                 // Field ajc$tjp_0:Lorg/aspectj/lang/JoinPoint$StaticPart;
     3: aload_0
     4: aload_0
     5: invokestatic  #52                 // Method org/aspectj/runtime/reflect/Factory.makeJP:(Lorg/aspectj/lang/JoinPoint$StaticPart;Ljava/lang/Object;Ljava/lang/Object;)Lorg/aspectj/lang/JoinPoint;
     8: astore_1
     9: aload_0
    10: aload_1
    11: invokestatic  #71                 // Method aspectjaop/annotation/TraceAspect.aspectOf:()Laspectjaop/annotation/TraceAspect;
    14: aload_1
    15: checkcast     #59                 // class org/aspectj/lang/ProceedingJoinPoint
    18: invokestatic  #75                 // Method rpcCall_aroundBody1$advice:(Laspectjaop/annotation/TraceTest;Lorg/aspectj/lang/JoinPoint;Laspectjaop/annotation/TraceAspect;Lorg/aspectj/lang/ProceedingJoinPoint;)Ljava/lang/Object;
    21: pop
    22: return
```


所以，maven aspectj 插件在编译的时候会把代码织入到生成的 .class 文件中，这是 AOP 的一个办法

## 例子2 Spring AOP

我们要研究的是 Spring AOP

```java
public class Logging {
    public void beforeAdvice() {
        System.out.println("Going to setup student profile.");
    }

    public void afterAdvice() {
        System.out.println("Student profile has been setup.");
    }

    public void afterReturningAdvice(Object retVal) {
        System.out.println("Returning returning advice ***** :");
    }

    public void AfterThrowingAdvice(IllegalArgumentException ex) {
        System.out.println("There has been an exception: " + ex.toString());
    }
}

public class Student {
    private int age;
    private String name;

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public void ExceptionRaised() {
        System.out.println("this is exception message ");
    }
}


public class MainApp {
    public static void main(String[] args) {
        ApplicationContext context =
                new ClassPathXmlApplicationContext("bean.xml");

        Student student = (Student) context.getBean("student");

        student.getName();
        student.getAge();

        student.ExceptionRaised();
    }
}
```

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-aop</artifactId>
    <version>4.3.4.RELEASE</version>
</dependency>

<dependency>
    <groupId>org.aspectj</groupId>
    <artifactId>aspectjrt</artifactId>
    <version>1.8.10</version>
</dependency>

<dependency>
    <groupId>org.aspectj</groupId>
    <artifactId>aspectjweaver</artifactId>
    <version>1.8.10</version>
</dependency>
```

第一个例子不需要 aspectjweaver 但是第二个例子却少不了。第一个例子需要 aspectj-maven-plugin 但是第二个例子是不需要的，因为 spring context 会托管注入这一流程，
后续的 Spring 章节就是用来分析 Spring AOP 的例子是如何实现的，帮助研究的代码就是例2

下面来分析 AOP 的应用流程。因为例子中被织入代码的类型不是接口，所以动态代理的实现应该是 CGLIB 而不是 JdkDynamicProxy

分析的切入点是函数 getProxy，知道了是 cglib 的 getProxy 函数，就把断点设在这个函数体内，根据函数的返回，能够知道整个调用栈，
也可以根据 intellij idea 的 thread stack 直接找到调用栈。

```java
AbstractAutowireCapableBeanFactory.doCreateBean()
AbstractAutowireCapableBeanFactory.initializeBean()
AbstractAutowireCapableBeanFactory.postProcessBeforeInitialization()
AbstractAutowireCapableBeanFactory.postProcessAfterInitialization()
    AbstractAdvisorAutoProxyCreator.createProxy
        DefaultAopProxyFactory.createAopProxy // 这里选择 JdkDynamicProxy 或者 CGLibProxy
            createProxy// 根据具体选取的 AOP proxy 构造代理实例
```


从上面的代理里，可以明确下面几点问题:

1. 对于每一个要生成的 Bean, 都必须在 PostProcessor 时检查它是否被注入了代码。检查的位置在 AbstractAdvisorAutoProxyCreator.findElibleAdvisor
对于 Bean 返回 List advisor, 很多 advisor 的实现是 PostProcessor 的子类
2. 定义的 Aspect 将被转换成 AspectJPointcutAdvisor，Advisor 分为两部分，第一部分是 Advice 第二部分是 Pointcut，注意 Advice 部分，注意，
在 Advisor 接口中只有 advice 而没有 pointcut, 包括 pointcut 的子类型叫做 AspectJPointcutAdvisor。每个注解生成一个 Advisor，Advisor 的 pointcut
就是注解的内容，它的 advice 部分就是函数的实现，函数的实现引用方法，比如 LoggingService 中的那些方法就是 advice
3. JdkDynamicProxy 是 InvocationHandler 和 AopProxy 的子类，它的成员变量有 AdvisedSupport，所以每一个 JdkDynamicProxy 实例对应一个
Bean
4. advisor holding advice(action to take at joint point, and a filter to determining the capability of the advice)
5. pointcut is compose of a class filter and method matcher
6. ReflectiveInterceptor.proceed. 无论是 cglib 还是 jdkDynamic 都是转化成 MethodInterceptor


看代码的时候还是有不理解的地方

1. 为什么要有 BeanDefinition? 为什么不能省掉这一层? A BeanDefinition describes a bean instance, which 
has property values, constructor argument values, and further information supplied by concrete 
implementations. 


其实 Cglib 构造动态代理的方式还是蛮简单，不需要过多分析。它就是找到各种 advisor 然后各种 set enhancer 就行了，但是反观 JdkDynamicProxy 就复杂些，因为
JdkDynamicProxy 的构造方法有些复杂。首先，JdkDynamicProxy 需要用到 advisedSupport, 在里面填好 advisor 和 targetClass 然后 JdkDynamicProxy 继承
InvocationHandler