---
layout: post
title: spring code 
categories: [spring]
keywords: spring
---


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

spring 提供了很多 IOC 的实现, 比如 XmlBeanFactory, ClasspathXmlApplicationContext 等,
其中 XmlBeanFactory 就是针对最基本的 IOC 容器, 这个 IOC 容器可以读取 XML 文件定义的 BeanDefinition,

ApplicationContext 是 spring 提供的一个高级 IOC 容器, 他除了能够提供 IOC 容器的基本功能以外, 还为用户
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

## 参考资料

[Spring Interview Questions](http://ifeve.com/java-memory-model-0/)

