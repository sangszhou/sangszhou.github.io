---
layout: post
title: spring aop
categories: [spring]
keywords: spring
---

### 什么是 AOP 更新

AOP 的核心思想就是“将应用程序中的核心逻辑同对其提供支持的通用服务进行分离。” 这些通用服务可能是日志，权限认证等等。

AOP( Aspect-OrientedProgramming, 面向方面编程), 可以说是OOP（Object-Oriented Programing，面向对象编程） 的补充和完善。OOP引入封装、继承和多态性等概念来建立一种对象层次结构用以模拟公共行为的一个集合。 它定义的是一个从上到下，从一般到特殊的层次结构。
当我们需要为分散的对象引入公共行为的时候，OOP则显得无能为力。
比如，日志代码往往水平地散布在所有对象层次中，而与它所散布到的对象的核心功能毫无关系。这种散布在各处的无关核心业务员的代码被称为横切（cross-cutting）代码，在OOP设计中，它导致了大量代码的重复，而不利于各个模块的重用。

所谓“方面”，简单地说，就是将那些与业务无关，却为业务模块所共同调用的逻辑或责任封装起来，便于减少系统的重复代码，
降低模块间的耦合度，并有利于未来的可操作性和可维护性。

实现AOP的技术，主要分为两大类：一是采用动态代理技术，利用截取消息的方式，对该消息进行装饰，以取代原有对象行为的 执行；二是采用静态织入的方式，引入特定的语法创建“方面”，从而使得编译器可以在编译期间织入有关“方面”的代码。

1. join point（连接点）：是程序执行中的一个精确执行点，例如类中的一个方法。它是一个抽象的概念，在实现AOP时，并不需要去定义一个join point。
2. point cut（切入点）：本质上是一个捕获连接点的结构。在AOP中，可以定义一个point cut，来捕获相关方法的调用。
3. advice（通知）：是point cut的执行代码，是执行“切面”的具体逻辑。
4. aspect（切面）：point cut 和 advice 结合起来就是aspect，它类似于OOP中定义的一个类，但它代表的更多是对象间横向的关系。
5. introduce（引入）：为对象引入附加的方法或属性，从而达到修改对象结构的目的。有的AOP工具又将其称为mixin。



## @todo 多态?

### 什么是多态

1. 编译时多态：编译时动态重载

2. 运行时多态：指一个对象可以具有多个类型，方法的覆盖

```
Car c = new Bus();
```

Car 编译时类型编译时检查变量类型是否存在，是否有调用的方法
Bus运行时类型 实际运行是访问 heap 中的对象，调用实际的方法。

运行时多态是由运行时类型决定的, 编译时多态是由编译时类型决定的



### 工作中用到的多态

### 怎么实现的

### 静态语言与动态语言

### 强类型, 弱类型

## 设计原则
[link](http://blog.jobbole.com/85529/)

**开闭原则**

Software entities should be open for extension,but closed for modification.

应该满足将来在不可修改源代码的情况下对模块的职能扩展，或者改变模块的行为



**接口隔离原则**

客户端不应该依赖它不需要的接口；一个类对另一个类的依赖应该建立在最小的接口上

接口隔离原则的含义是：建立单一接口，不要建立庞大臃肿的接口，尽量细化接口，接口中的方法尽量少。
也就是说，我们要为各个类建立专用的接口，而不要试图去建立一个很庞大的接口供所有依赖它的类去调用。
本文例子中，将一个庞大的接口变更为3个专用的接口所采用的就是接口隔离原则。在程序设计中，依赖几个专用的接口要比依赖一个综合的接口更灵活。
接口是设计时对外部设定的“契约”，通过分散定义多个接口，可以预防外来变更的扩散，提高系统的灵活性和可维护性。

感觉和 **单一职责原则**比较类似

**依赖倒置原则**

定义：高层模块不应该依赖低层模块，二者都应该依赖其抽象；抽象不应该依赖细节；细节应该依赖抽象

依赖倒置原则的核心思想是面向接口编程，我们依旧用一个例子来说明面向接口编程比相对于面向实现编程好在什么地方。场景是这样的，母亲给孩子讲故事，只要给她一本书，她就可以照着书给孩子讲故事了。代码如下：



## DI 和 IOC

在java中，软件代码的逻辑组件和功能服务的划分，通常被定义为Java类或者对象的实例。而每一个对象都必须使用其他的对象，
或者与其他对象合作以便完成它的任务。比如，对于对象A，可以说凡是对象 A 涉及的其他对象都是它的依赖。

在旧有的Java中，习惯于自己 new 到一个实例，由类或者对象自身来管理自己的依赖实例。
而IoC就是为了避免这一问题。这样子，我们就可以通过简单的配置，来改变获得的实例，而不用去修改大量的代码。

### 依赖的注入方式

**Setter 注入**

**构造器注入**

就是通过类的构造器了，带参数的构造器来设置依赖的类实例或属性

**方法注入** 就是普通方法的注入, 使用频率较低

## 面向接口编程

## Spring IOC 架构

![](/images/posts/spring/spring-ioc-structure.x-png)

其中BeanFactory作为最顶层的一个接口类，它定义了IOC容器的基本功能规范，BeanFactory 有三个子类：
`ListableBeanFactory`、`HierarchicalBeanFactory` 和 `AutowireCapableBeanFactory`。
但是从上图中我们可以发现最终的默认实现类是 DefaultListableBeanFactory，他实现了所有的接口。
那为何要定义这么多层次的接口呢？查阅这些接口的源码和说明发现，每个接口都有他使用的场合，
它主要是为了区分在 Spring 内部在操作过程中对象的传递和转化过程中，对对象的数据访问所做的限制。
例如 ListableBeanFactory 接口表示这些 Bean 是可列表的，而 HierarchicalBeanFactory 表示的是
这些 Bean 是有继承关系的，也就是每个Bean 有可能有父 Bean。AutowireCapableBeanFactory 接口
定义 Bean 的自动装配规则。这四个接口共同定义了 Bean 的集合、Bean 之间的关系、以及 Bean 行为. (例子?)

```java
public interface BeanFactory {
    String FACTORY_BEAN_PREFIX = "&";

    Object getBean(String name) throws BeansException;

    Object getBean(String name, class requiredType) throws BeansException;

    boolean containsBean(String name);

    boolen isSingleton(String name) throws NoSuchBeanDefinitionException;

    Class getType(String name) throws NoSuchBeanDefinitionException;

    String[] getAlias(String name)
}
```

spring 提供了很多 IOC 的实现, 比如 XmlBeanFactory, ClasspathXmlApplicationContext, AnnotationApplicationContext 等,
其中 XmlBeanFactory 就是针对最基本的 IOC 容器, 这个 IOC 容器可以读取 XML 文件定义的 BeanDefinition, AnnotationApplicationContext 提供的 ApplicationContext 是 spring 提供的一个高级 IOC 容器, 他除了能够提供 IOC 容器的基本功能以外, 还为用户
提供了以下的附加服务。

1. 支持信息源, 可以实现国际化 (MessageSource) 接口
2. 访问资源 (实现了 ResourcePatternResolver 接口)
3. 支持应用事件 (实现 ApplicationEventPublisher 接口)

Bean 的解析过程非常复杂, 功能被分的很细, 因为需要扩展的地方很多, 必须保证足够的灵活性, 以应对可能的变化。
Bean 的解析就是对 spring 配置文件的解析, 这个解析过程主要通过以下图中的类完成:

![](/images/posts/spring/BeanReaderClasses.x-png)

## IOC 容器的创建过程

### XmlBeanFactory 的整个流程

```java
public class XmlBeanFactory extends DefaultListenableFactory {
    private final XmlBeanDefinitionReader reader;

    public XmlBeanFactory(Resource resource) throws BeanException {
        this(resource, null)
    }

    public XmlBeanFactory(Resource resource, BeanFactory parent) {
        super(parent)
        this.reader = new XmlBeanDefinitionReader(this)
        this.reader.loadBeanDefinitions(reader)
    }
}

// build resource object by xml which contans BeanDefinition info
ClassPathResource resource = new ClassPathResource("context.xml")

DefaultListableBeanFactory factory = new DefaultListableBeanFactory()

// 创建 reader, 用于载入 BeanDefinition, 之所以需要 BeanFactory 作为参数
// 是因为会将读取的信息回调配置给 factory
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(reader)

// 完成的 bean 会被放到 IOC 容器中
reader.loadBeanDefinitions(resource)
```


## 反射和动态代理

```java
 Class.forName()

 Field field = Unsafe.class.getDeclaredField("theUnsafe")
 field.setAccessible(true)
 unsafe = (Unsafe) field.get(Unsafe.class);

 Field[] fields = clazz.getDeclaredFields();
 Method[] methods = clazz.getDeclaredMethods();
 Class<?>[] clazzes = method.getParameterTypes();
 Student student = (Student)clazz.newInstance();

// 另一种获得 unsafe 的解法是通过改造函数

```

提高反射性能

反射慢的原因有两个, 一个是找 class 文件可能花时间比较久, 另一个是无法使用 JIT 编译器优化反射操作

1. 使用缓存, 能得到的 class 文件就重用
2. 不要遍历 field, method
3. setAccessible(true) 可以省掉安全检查的时间

### Java 动态代理

Interface：对于JDK proxy，业务类是需要一个Interface的，这也是一个缺陷

### cglib 动态代理, 为什么它不需要实现接口

JDK的动态代理机制只能代理实现了接口的类，而不能实现接口的类就不能实现JDK的动态代理，cglib是针
对类来实现代理的，他的原理是对指定的目标类生成一个子类，并覆盖其中方法实现增强，但因为采用的是继
承，所以不能对final修饰的类进行代理。

JDK实现动态代理需要实现类通过接口定义业务方法，对于没有接口的类，如何实现动态
代理呢，这就需要CGLib了。CGLib采用了非常底层的字节码技术，其原理是通过字节码技术为一个
类创建子类，并在子类中采用方法拦截的技术拦截所有父类方法的调用，顺势织入横切逻辑。JDK动态
代理与CGLib动态代理均是实现Spring AOP的基础

CGLib创建的动态代理对象性能比JDK创建的动态代理对象的性能高不少，但是CGLib
在创建代理对象时所花费的时间却比JDK多得多，所以对于单例的对象，因为无需频繁创建
对象，用CGLib合适，反之，使用JDK方式要更为合适一些。同时，由于CGLib由于是采用动态创
建子类的方法，对于final方法，无法进行代理

```java
public class BookFacadeCglib implements MethodInterceptor {  
    private Object target;  

    public Object getInstance(Object target) {  
        this.target = target;  
        Enhancer enhancer = new Enhancer();  
        enhancer.setSuperclass(this.target.getClass());  
        // 回调方法  
        enhancer.setCallback(this);  
        // 创建代理对象  
        return enhancer.create();  

    // intercept 所有的函数
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) {  
        System.out.println("事物开始");  
        proxy.invokeSuper(obj, args);  
        System.out.println("事物结束");  
        return null;   
}
```

### 动态代理的原理

```java
public interface Subject
    void request()

public class RealSubject implements Subject
    @override
    public void request
        println("real subject")

public class ProxyHandler implements Invocationhandler
    private Subject subject

    public ProxyHandler(Subject subject)
        this.subject = subject

    public Object invoke(Object proxy, Method method, Object[] args)
        println("Before method execution")
        Object result = method.invoke(subject, args)
        println("after execution)
        return result

public class DynamicProxyTest
    main(String args[])
        RealSubject realSubject = new RealSubject
        ProxyHandler proxyHandler = new ProxyHandler(realSubject)

        Subject proxied = (Subject) Proxy.newProxyInstance(realSubject.class.getClassLoader, realSubject.getClass.getInterface
            proxyHandler)

        proxied.request

public static Object newProxyInstance(ClassLoader loader,
                                      Class<?>[] interfaces,
                                      InvocationHandler h)
```

```java
final class $Proxy1 extends Proxy implements Subject
    private InvocationHandler h
    private $Proxy1
    public $Proxy1(InvocationHandler h)
        this.h = h

    public int request(int i)
        Method method = Subject.class.getMethod("request", new Class[]{int.class})
        return (Integer)h.invoke(this, method, new Object[]{new Integer(i)})

// new Proxy 方法的实现
    public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                              InvocationHandler h)
        // lookup cache or create
        Class<?> cl = getProxyClass0(loader, intfs);
        final Constructor<?> cons = cl.getConstructor(constructorParams);
        final InvocationHandler ih = h;
        return cons.newInstance(new Object[]{h});
```

实现 ProxyHandler 还不需要知道 Subject, 技巧就是把 Subject 搞成类似于  Object 的东西, 把所有变化的东西封装成一个类型, 而变化的部分仅有两个
第一个是 interface, 另一个是我们拦截方法后要做的事情 (preAction, postAction)

```java
public class JdkDynamicAopProxy implements AopProxy, InvocationHandler
    AdvisedSupport advised
    public JdkDynamicAopProxy(AdvisedSupport advised)
        this.advised = advised

    public Object getProxy()
        return Proxy.newProxyInstance(getClass.getClassLoader, new Class[]{advised.getTargetSource.getTargetClass}, this)

    public Object invoke(Object proxy, final Method method, Object[] args)
        MethodInterceptor methodInterceptor = advised.getMethodInterceptor
        return methodInterceptor.invoke(new ReflectiveMethodInvocation(advised.getTargetSource.getTarget, method, args)
```

在这样一个 JdkDynamicAopProxy 中, 既没有 interface 又没有抽象逻辑, 这两个变化的逻辑都抽象出来放在 AdvisedSupport 中了

一般, before, after, around 的代码写在 TimeInterceptor 中

```java
class AdvisedSupport  // bean
    TargetSource targetSource
    MethodInterceptor methodInterceptor

interface MethodInterceptor extends Interceptor
    Object invoke(MethodInvocation invocation)

public interface TargetSource extends TargetClassAware
    Class<?> getTargetClass();
```

举个例子

```java
class TimeInterceptor implements MethodInterceptor
    Object invoke(MethodInvocation invocation)
        long time = System.nanoTime
        println("start time")
        Object proceed = invocation.proceed
        println("end time")
        return proceed

TimeInterceptor timerInterceptor = new TimerInterceptor
advisedSupport.setMethodInterceptor(timeInterceptor)
```

This is exactly how spring do it. Decouple method interceptor from business logic and let Proxy be static one.

dynamic proxy 创建新类的代码

```java
创建代码如下

    long num;
    // 获得代理类数字标识

   synchronized (nextUniqueNumberLock) {
     num = nextUniqueNumber ++;
   }

    //获得创建新类的类名$Proxy，包名为接口包名，但需要注意的是，如果有两个接口而且不在同一个包下，也会报错

    String proxyName = proxyPkg + proxyClassNamePrefix + num;
    //调用class处理文件生成类的字节码，根据接口列表创建一个新类，这个类为代理类，
    byte[] proxyClassFile = ProxyGenerator.generateProxyClass(proxyName, interfaces);
    //通过JNI接口，将Class字节码文件定义一个新类

    proxyClass = defineClass0(loader, proxyName, proxyClassFile, 0, proxyClassFile.length);

    Constructor cons = cl.getConstructor(constructorParams);

    public class $Proxy1 extends Proxy implements 传入的接口{

    }
```

所以 JDK 的动态代理, 让新类继承了 Proxy 就失去继承另外一个 class 的能力, 我想如果有 java8 的话, 说不定
就成了

### CGLIB works

Spring 和 cglib 动态代理中都有 interface MethodInterceptor, 但是他们的抽象方法签名并不一致, 一个是 intercept, 另一个是 invoke

cglib采用的是用创建一个继承实现类的子类，用asm库动态修改子类的代码来实现的，所以可以用传入的类引用执行代理类

```java
public class AroundAdvice implements MethodInterceptor
    public Object intercept(Object target, Method method, Object[] args, MethodProxy proxy)
        println("before")
        Object result = proxy.invokeSuper(target, new String[]())
        println("after..")
        return result

public class ChineseProxyFactory
    static chinese getAuthInstance
        Enhancer en = new Enhancer
        en.setSuperClass(Chinese.class)
        en.setCallback(new AroundAdvice)
        return (Chinse)en.create
```

上面粗体字代码就是使用 CGLIB 的 Enhancer 生成代理对象的关键代码，此时的 Enhancer 将以 Chinese 类作为目标类，
以 AroundAdvice 对象作为“Advice”，程序将会生成一个 Chinese 的子类，这个子类就是 CGLIB 生成代理类，它可作为 Chinese 对象使用，但
它增强了 Chinese 类的方法

cglib 的 method interceptor 唯一函数是 intercept

```java
public interface MethodInterceptor extends Callback {
    Object intercept(Object var1, Method var2, Object[] var3, MethodProxy var4) throws Throwable;
}

// General purpose AOP callback. Used when the target is dynamic or when the proxy is not frozen
static class DynamicAdvisedInterceptor implements MethodInterceptor
    private final AdvisedSupport advised;

        Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy)
            List<object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, target)
            if(chain.isEmpty && modified.isPublic(method.getModifier)
                retVal = methodProxy.invoke(target, argsToUse)
            else
                retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain).proceed
        return retVal
```

```java
interface AopProxy
    Object getProxy
    Object getProxy(ClassLoader classLoader)

class CglibAopProxy implements AopProxy
	/** The configuration used to configure this proxy */
	protected final AdvisedSupport advised;

    public Object getProxy(ClassLoader classLoader) {
        Class<?> rootClass = this.advised.getTargetClass();
        Enhancer enhancer = createEnhancer();
        enhancer.setClassLoader(classLoader);
        enhancer.setSuperclass(proxySuperClass);
        enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
        enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
        enhancer.setStrategy(new ClassLoaderAwareUndeclaredThrowableStrategy(classLoader));
        Callback[] callbacks = getCallbacks(rootClass); // retrieve callback from advisedSupport
        enhancer.setCallbackFilter(new ProxyCallbackFilter(
            this.advised.getConfigurationOnlyCopy(), this.fixedInterceptorMap, this.fixedInterceptorOffset));
        enhancer.setCallbackTypes(types);
        return createProxyClassAndInstance(enhancer, callbacks); // 在此函数内添加 callback

```


### what is Bean Factory, have you used XMLBeanFactory?

Ans: BeanFactory is factory Pattern which is based on IOC design principles.it is used to make a clear separation between application configuration and dependency from actual code. The XmlBeanFactory is one of the implementations of bean Factory which we have used in our project. The org.springframework.beans.factory.xml.XmlBeanFactory is used to create bean instance defined in our XML file.

```java
BeanFactory factory = new XmlBeanFactory(new FileInputStream("beans.xml"));
Or

ClassPathResource resorce = new ClassPathResource("beans.xml");
XmlBeanFactory factory = new XmlBeanFactory(resorce);
```

### Question 4: What are the difference between BeanFactory and ApplicationContext in spring? (answer) @todo

ApplicationContext is the preferred way of using spring because of functionality provided by it and interviewer
wanted to check whether you are familiar with it or not.

### Explain the Spring Bean-LifeCycle

Ans: Spring framework is based on IOC so we call it as IOC container also So Spring beans reside inside the IOC container.
Spring beans are nothing but Plain old java object (POJO).

Following steps explain their life cycle inside the container.

1. The container will look the bean definition inside configuration file (e.g. bean.xml).
2. using reflection container will create the object and if any property is defined inside the bean definition then it will also be set.
3. If the bean implements the `BeanNameAware` interface, the factory calls setBeanName() passing the bean’s ID.
4. If the bean implements the `BeanFactoryAware` interface, the factory calls setBeanFactory(), passing an instance of itself.
5. If there are any BeanPostProcessors associated with the bean, their post- ProcessBeforeInitialization() methods will be called before the properties for the Bean are set.
6. If an init() method is specified for the bean, it will be called.
7. If the Bean class implements the DisposableBean interface, then the method destroy() will be called when the Application no longer needs the bean reference.
8. If the Bean definition in the Configuration file contains a 'destroy-method' attribute, then the corresponding method definition in the Bean class will be called.

1)  Spring对Bean 进行实例化．
2)  Spring 将值和Bean的引用注入进Bean对应的属性中。
3)  如果Bean 实现了BeanNameAware 接口， Spring 将bean 的ID 传递给setBeanName() 接口方法．
4)  如果Bean实现了BeanFactoryAware 接口， Spring 将调用setBeanFactory()接口方法，将BeanFactory容器实例传入．
5)  如果Bean 实现了ApplicationcontextAware 接口 Spring 将调用setApplicationContext() 接口方法，将应用上下文的引用传入。
6)  如果Bean实现了BeanPostProcessor 接口Spring 将调用它们的postProcessBeforeInitialization 接口方法。
7)  如果Bean 实现了InitializingBean 接口，Spring 将调用它们的afterPropertiesSet()接口方法． 类似地，如果Bean 使用init-method 声明了初始化方法，该方法也会被调用。
8)  如果Bean 实现了BeanPostProcessor 接口， Spring 将调用它们的postPoressAfterInitilization方法．
9)  此时此刻Bean 已经准备就绪．可以被应用程序使用了． 它们将一直驻留在应用上下文中．直到该应用上下文补销毁。
10) 如果Bean实现了DisposableBean 接口，Spring 将调用它的destroy()接口方法。同样，如果Bean 使用destroy-method 声明了销毁方法，方法也会被调用。

### What is IOC or inversion of control? (answer)

As the name implies Inversion of control means now we have inverted the control of creating
the object from our own using new operator to container or framework. Now it’s the responsibility of container to create
an object as required. We maintain one XML file where we configure our components, services, all the classes and their
property. We just need to mention which service is needed by which component and container will create the object for us.
This concept is known as dependency injection because all object dependency (resources) is injected into it by the framework.

```xml
<bean id="createNewStock" class="springexample.stockMarket.CreateNewStockAccont">
        <property name="newBid"/>
</bean>
```

In this example, CreateNewStockAccont class contain getter and setter for newBid and container will
instantiate newBid and set the value automatically when it is used. This whole process is also called
wiring in Spring and by using annotation it can be done automatically by Spring, refereed as auto-wiring of bean in Spring.

### Question 5: What are different modules in spring?

1.      The Core container module
2.      Application context module
3.      AOP module (Aspect Oriented Programming)
4.      JDBC abstraction and DAO module
5.      O/R mapping integration module (Object/Relational)
6.      Web module
7.      MVC framework module

### Question 6: What is the difference between singleton and prototype bean? @todo

Ans: This is another popular spring interview questions and an important concept to understand.
Basically, a bean has scopes which define their existence on the application

**Singleton:** means single bean definition to a single object instance per Spring IOC container.

**Prototype:** means a single bean definition to any number of object instances.
Whatever beans we defined in spring framework are singleton beans. There is an attribute in bean tag named ‘singleton’
if specified true then bean becomes singleton and if set to false then the bean becomes a prototype bean.
By default, it is set to true. So, all the beans in spring framework are by default singleton beans.


```xml
<bean id="createNewStock"     class="springexample.stockMarket.CreateNewStockAccont" singleton=”false”>
        <property name="newBid"/>
</bean>
```

### Question 7: What type of transaction Management Spring support?

Ans: This spring interview questions is little difficult as compared to previous questions just because
transaction management is a complex concept and not every developer familiar with it.
Transaction management is critical in any applications that will interact with the database.
The application has to ensure that the data is consistent and the integrity of the data is maintained.  
Following two type of transaction management is supported by spring:

1. Programmatic transaction management
2. Declarative transaction management.

### bean 的作用域

在Spring中创建一个bean的时候，我们可以声明它的作用域。只需要在bean定义的时候通过’scope’属性定义即可。
例如，当Spring需要产生每次一个新的bean实例时，应该声明bean的scope属性为prototype。
如果每次你希望Spring返回一个实例，应该声明bean的scope属性为singleton。

Spring框架支持如下五种不同的作用域：

1. singleton：在Spring IOC容器中仅存在一个Bean实例，Bean以单实例的方式存在。
2. prototype：一个bean可以定义多个实例。
3. request：每次HTTP请求都会创建一个新的Bean。该作用域仅适用于WebApplicationContext环境。
4. session：一个HTTP Session定义一个Bean。该作用域仅适用于WebApplicationContext环境.
5. globalSession：同一个全局HTTP Session定义一个Bean。该作用域同样仅适用于WebApplicationContext环境.

bean默认的scope属性是 ’singleton‘。

Spring框架中的单例beans不是线程安全的

bean标签有两个重要的属性(init-method 和 destroy-method)，你可以通过这两个属性定义自己的初始化方法和析构方法。

Update: when preDestroy is implemented

three ways to add pre destroy method

```java
@PreDestroy tag

destroy-method in xml

DisposableBean interface as stated above
```


BeanFactory 关闭的时候似乎就会调用 preDestroy 方法

```java
public static void main(String[] args) {
    ClassPathXmlApplicationContext ctx = new ClassPathXmlApplicationContext("spring.xml");
	System.out.println("Spring Context initialized");
		
	//MyEmployeeService service = ctx.getBean("myEmployeeService", MyEmployeeService.class);
	EmployeeService service = ctx.getBean("employeeService", EmployeeService.class);
	System.out.println("Bean retrieved from Spring Context");	
	System.out.println("Employee Name="+service.getEmployee().getName());
	ctx.close();
	System.out.println("Spring Context Closed");
}
```

Spring也有相应的注解：@PostConstruct 和 @PreDestroy

### 解释自动装配的各种模式

自动装配提供五种不同的模式供Spring容器用来自动装配beans之间的依赖注入:

1. no：默认的方式是不进行自动装配，通过手工设置ref 属性来进行装配bean。
2. byName：通过参数名自动装配，Spring容器查找beans的属性，这些beans在XML配置文件中被设置为byName。之后容器试图匹配、装配和该bean的属性具有相同名字的bean。
3. byType：通过参数的数据类型自动自动装配，Spring容器查找beans的属性，这些beans在XML配置文件中被设置为byType。之后容器试图匹配和装配和该bean的属性类型一样的bean。如果有多个bean符合条件，则抛出错误。
4. constructor：这个同byType类似，不过是应用于构造函数的参数。如果在BeanFactory中不是恰好有一个bean与构造函数参数相同类型，则抛出一个严重的错误。
5. autodetect：如果有默认的构造方法，通过 construct的方式自动装配，否则使用 byType的方式自动装配。

自动装配有如下局限性：

1. 重写：你仍然需要使用 和< property>设置指明依赖，这意味着总要重写自动装配。
2. 原生数据类型:你不能自动装配简单的属性，如原生类型、字符串和类。
3. 模糊特性：自动装配总是没有自定义装配精确，因此，如果可能尽量使用自定义装配。

### 常用注解

**@Required 注解**

@Required表明bean的属性必须在配置时设置，可以在bean的定义中明确指定也可通过自动装配设置。如果bean的属性未设置，则抛出BeanInitializationException异常。

**@Autowired 注解**

@Autowired 注解提供更加精细的控制，包括自动装配在何处完成以及如何完成。它可以像@Required一样自动装配setter方法、构造器、属性或者具有任意名称和/或多个参数的PN方法。

**@Qualifier 注解**

当有多个相同类型的bean而只有其中的一个需要自动装配时，将@Qualifier 注解和@Autowire 注解结合使用消除这种混淆，指明需要装配的bean。

### AOP 概念

54.连接点(Join point)

连接点代表应用程序中插入AOP切面的地点。它实际上是Spring AOP框架在应用程序中执行动作的地点。

55.通知(Advice)

通知表示在方法执行前后需要执行的动作。实际上它是Spring AOP框架在程序执行过程中触发的一些代码。

Spring切面可以执行一下五种类型的通知:

before(前置通知)：在一个方法之前执行的通知。

after(最终通知)：当某连接点退出的时候执行的通知（不论是正常返回还是异常退出）。

after-returning(后置通知)：在某连接点正常完成后执行的通知。

after-throwing(异常通知)：在方法抛出异常退出时执行的通知。

around(环绕通知)：在方法调用前后触发的通知。

56.切入点(Pointcut)

切入点是一个或一组连接点，通知将在这些位置执行。可以通过表达式或匹配的方式指明切入点。

### 为了降低Java开发的复杂性， Spring采取了以下4 种关键策略

1. 基于POJO的轻量级和最小侵入性编程，
2. 通过依赖注入和面向接口实现松耦合，
3. 基于切面和惯例进行声明式编程，
4. 通过切面和模板减少样板式代码。

### 依赖注入有哪些方式

1. 接口注入（interface injection） 接口注入指的就是在接口中定义要注入的信息，并通过接口完成注入。
2. Set注入（setterinjection）Set注入指的就是在接受注入的类中定义一个Set方法，并在参数中定义需要注入的元素。
3. 构造注入（constructor injection）构造注入指的就是在接受注入的类中定义一个构造方法，并在参数中定义需要注入的元素

### Spring应用上下文的有哪些

1. ClassPathXmlApplicationContext——从类路径下的XML配置文件中加载上下文定义，把应用上下文定义文件当作类资源。
2. FileSystemXmlApplicationcontext该取文件系统统下XML配置文件并加载上下文定义。
3. XmlWebApplicationContext——读取Web应用下的XML 配置文件并装载上下文定义。

使用FileSystemXmlapplicationcontext和使用ClassPathXmlApplicationContext的区别在于:
FileSystemXmlapplicationcontext在指定的文件系统路径下查找文件，


## 参考资料

[Spring Interview Questions](http://ifeve.com/java-memory-model-0/)
