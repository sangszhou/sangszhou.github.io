---
layout: post
title: Spring Core And MVC
categories: [spring]
keywords: java, spring
---

## AOP

AOP( Aspect-OrientedProgramming, 面向方面编程), 可以说是OOP（Object-Oriented Programing，面向对象编程）
的补充和完善。OOP引入封装、继承和多态性等概念来建立一种对象层次结构用以模拟公共行为的一个集合。
当我们需要为分散的对象引入公共行为的时候，OOP则显得无能为力。也就是说，OOP允许你定义从上到下的关系，但并不适合定义从左到右的关系。
例如日志功能。日志代码往往水平地散布在所有对象层次中，而与它所散布到的对象的核心功能毫无关系。对于其他类型的代码，如安全性、异常处理和透明的持续性也是如此。这种散布在各处的无关的代码被称为横切（cross-cutting）代码，在OOP设计中，它导致了大量代码的重复，而不利于各个模块的重用。

而AOP技术则恰恰相反，它利用一种称为“横切”的技术，剖解开封装的对象内部，并将那些影响了多个类的公共行为封装到
一个可重用模块，并将其名为“Aspect”，即方面。所谓“方面”，简单地说，就是将那些与业务无关，却为业务模块所共
同调用的逻辑或责任封装起来，便于减少系统的重复代码，降低模块间的耦合度，并有利于未来的可操作性和可维护性。
AOP代表的是一个横向的关系，如果说“对象”是一个空心的圆柱体，其中封装的是对象的属性和行为；那么面向方面编程的方法,
就仿佛一把利刃，将这些空心圆柱体剖开，以获得其内部的消息。而剖开的切面，也就是所谓的“方面”了。然后它又以巧夺天
功的妙手将这些剖开的切面复原，不留痕迹。

使用“横切”技术，AOP把软件系统分为两个部分：核心关注点和横切关注点。业务处理的主要流程是核心关注点，
与之关系不大的部分是横切关注点。横切关注点的一个特点是，他们经常发生在核心关注点的多处，而各处都基本相似。
比如权限认证、日志、事务处理。Aop 的作用在于分离系统中的各种关注点，将核心关注点和横切关注点分离开来。
正如Avanade公司的高级方案构架师Adam Magee所说，AOP的核心思想就是“将应用程序中的商业逻辑同对其提供支持的通
用服务进行分离。”

实现AOP的技术，主要分为两大类：一是采用动态代理技术，利用截取消息的方式，对该消息进行装饰，以取代原有对象行为的
执行；二是采用静态织入的方式，引入特定的语法创建“方面”，从而使得编译器可以在编译期间织入有关“方面”的代码。

![](/images/posts/spring/aop_principal.png)

### Spring AOP代理对象的生成

Spring提供了两种方式来生成代理对象: JDKProxy和Cglib，具体使用哪种方式生成由AopProxyFactory根据
AdvisedSupport 对象的配置来决定。默认的策略是如果目标类是接口，则使用 JDK 动态代理技术，否则使用 Cglib 来生成代理。
下面我们来研究一下Spring如何使用JDK来生成代理对象，具体的生成代码放在 **JdkDynamicAopProxy** 这个类中，
直接上相关代码：

```java
/** 
    * <li>获取代理类要实现的接口,除了Advised对象中配置的,还会加上SpringProxy, Advised(opaque=false) 
    * <li>检查上面得到的接口中有没有定义 equals或者hashcode的接口 
    * <li>调用Proxy.newProxyInstance创建代理对象 
    */  
   public Object getProxy(ClassLoader classLoader) {  
       if (logger.isDebugEnabled()) {  
           logger.debug("Creating JDK dynamic proxy: target source is " +this.advised.getTargetSource());  
       }
         
       Class[] proxiedInterfaces =AopProxyUtils.completeProxiedInterfaces(this.advised);  
       findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);  
       return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);  
}  
```

我们知道 InvocationHandler 是JDK动态代理的核心，生成的代理对象的方法调用都会委托到 InvocationHandler.invoke() 方法。
而通过 JdkDynamicAopProxy 的签名我们可以看到这个类其实也实现了 InvocationHandler，下面我们就通过分析这个类中实现的 invoke() 方法来具体看
下 Spring AOP 是如何织入切面的

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
       MethodInvocation invocation = null;
       Object oldProxy = null;
       boolean setProxyContext = false;
 
       TargetSource targetSource = this.advised.targetSource;
       Class targetClass = null;
       Object target = null;
 
       try {
           //eqauls()方法，具目标对象未实现此方法
           if (!this.equalsDefined && AopUtils.isEqualsMethod(method)){
                return (equals(args[0])? Boolean.TRUE : Boolean.FALSE);
           }
 
           //hashCode()方法，具目标对象未实现此方法
           if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)){
                return newInteger(hashCode());
           }
 
           //Advised接口或者其父接口中定义的方法,直接反射调用,不应用通知
           if (!this.advised.opaque &&method.getDeclaringClass().isInterface()
                    &&method.getDeclaringClass().isAssignableFrom(Advised.class)) {
                // Service invocations onProxyConfig with the proxy config...
                return AopUtils.invokeJoinpointUsingReflection(this.advised,method, args);
           }
 
           Object retVal = null;
 
           if (this.advised.exposeProxy) {
                // Make invocation available ifnecessary.
                oldProxy = AopContext.setCurrentProxy(proxy);
                setProxyContext = true;
           }
 
           //获得目标对象的类
           target = targetSource.getTarget();
           if (target != null) {
                targetClass = target.getClass();
           }
 
           //获取可以应用到此方法上的Interceptor列表
           List chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method,targetClass);
 
           //如果没有可以应用到此方法的通知(Interceptor)，此直接反射调用 method.invoke(target, args)
           if (chain.isEmpty()) {
                retVal = AopUtils.invokeJoinpointUsingReflection(target,method, args);
           } else {
                //创建MethodInvocation
                invocation = newReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
                retVal = invocation.proceed();
           }
 
           // Massage return value if necessary.
           if (retVal != null && retVal == target &&method.getReturnType().isInstance(proxy)
                    &&!RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
                // Special case: it returned"this" and the return type of the method
                // is type-compatible. Notethat we can't help if the target sets
                // a reference to itself inanother returned object.
                retVal = proxy;
           }
           return retVal;
       } finally {
           if (target != null && !targetSource.isStatic()) {
                // Must have come fromTargetSource.
               targetSource.releaseTarget(target);
           }
           if (setProxyContext) {
                // Restore old proxy.
                AopContext.setCurrentProxy(oldProxy);
           }
       }
}
```

主流程可以简述为：获取可以应用到此方法上的通知链（Interceptor Chain）,如果有, 则应用通知,并执行joinpoint; 如果没有,则直接反射执行joinpoint。
而这里的关键是通知链是如何获取的以及它又是如何执行的，下面逐一分析下。

首先，从上面的代码可以看到，通知链是通过cAdvised.getInterceptorsAndDynamicInterceptionAdvice()c这个方法来获取的, 我们来看下这个方法的实现:

```java
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method, Class targetClass) {  
                   MethodCacheKeycacheKey = new MethodCacheKey(method);  
                   List<Object>cached = this.methodCache.get(cacheKey);  
                   if(cached == null) {  
                            cached= this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(  
                                               this,method, targetClass);  
                            this.methodCache.put(cacheKey,cached);  
                   }  
                   returncached;  
}  
```

可以看到实际的获取工作其实是由AdvisorChainFactory. getInterceptorsAndDynamicInterceptionAdvice()这个方法来完成的，获取到的结果会被缓存。
下面来分析下这个方法的实现:

```java
/**
    * 从提供的配置实例config中获取advisor列表,遍历处理这些advisor.如果是IntroductionAdvisor,
    * 则判断此Advisor能否应用到目标类targetClass上.如果是PointcutAdvisor,则判断
    * 此Advisor能否应用到目标方法method上.将满足条件的Advisor通过AdvisorAdaptor转化成Interceptor列表返回.
    */
    publicList getInterceptorsAndDynamicInterceptionAdvice(Advised config, Methodmethod, Class targetClass) {
       // This is somewhat tricky... we have to process introductions first,
       // but we need to preserve order in the ultimate list.
       List interceptorList = new ArrayList(config.getAdvisors().length);
 
       //查看是否包含IntroductionAdvisor
       boolean hasIntroductions = hasMatchingIntroductions(config,targetClass);
 
       //这里实际上注册一系列AdvisorAdapter,用于将Advisor转化成MethodInterceptor
       AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();
 
       Advisor[] advisors = config.getAdvisors();
        for (int i = 0; i <advisors.length; i++) {
           Advisor advisor = advisors[i];
           if (advisor instanceof PointcutAdvisor) {
                // Add it conditionally.
                PointcutAdvisor pointcutAdvisor= (PointcutAdvisor) advisor;
                if(config.isPreFiltered() ||pointcutAdvisor.getPointcut().getClassFilter().matches(targetClass)) {
                    //TODO: 这个地方这两个方法的位置可以互换下
                    //将Advisor转化成Interceptor
                    MethodInterceptor[]interceptors = registry.getInterceptors(advisor);
 
                    //检查当前advisor的pointcut是否可以匹配当前方法
                    MethodMatcher mm =pointcutAdvisor.getPointcut().getMethodMatcher();
 
                    if (MethodMatchers.matches(mm,method, targetClass, hasIntroductions)) {
                        if(mm.isRuntime()) {
                            // Creating a newobject instance in the getInterceptors() method
                            // isn't a problemas we normally cache created chains.
                            for (intj = 0; j < interceptors.length; j++) {
                               interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptors[j],mm));
                            }
                        } else {
                            interceptorList.addAll(Arrays.asList(interceptors));
                        }
                    }
                }
           } else if (advisor instanceof IntroductionAdvisor){
                IntroductionAdvisor ia =(IntroductionAdvisor) advisor;
                if(config.isPreFiltered() || ia.getClassFilter().matches(targetClass)) {
                    Interceptor[] interceptors= registry.getInterceptors(advisor);
                    interceptorList.addAll(Arrays.asList(interceptors));
                }
           } else {
                Interceptor[] interceptors =registry.getInterceptors(advisor);
                interceptorList.addAll(Arrays.asList(interceptors));
           }
       }
       return interceptorList;
}
```

从这段代码可以看出，如果得到的拦截器链为空，则直接反射调用目标方法，否则创建MethodInvocation，调用其proceed方法，触发拦截器链的执行，来看下具体代码


这个方法执行完成后，Advised中配置能够应用到连接点或者目标类的Advisor全部被转化成了MethodInterceptor.

```java
if (chain.isEmpty()) {
                retVal = AopUtils.invokeJoinpointUsingReflection(target,method, args);
            } else {
                //创建MethodInvocation
                invocation = newReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
                retVal = invocation.proceed();
}
```


从这段代码可以看出，如果得到的拦截器链为空，则直接反射调用目标方法，否则创建MethodInvocation，调用其proceed方法，触发拦截器链的执行，来看下具体代码


```java
public Object proceed() throws Throwable {
       //  We start with an index of -1and increment early.
       if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size()- 1) {
           //如果Interceptor执行完了，则执行joinPoint
           return invokeJoinpoint();
       }
 
       Object interceptorOrInterceptionAdvice =
           this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
       
       //如果要动态匹配joinPoint
       if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher){
           // Evaluate dynamic method matcher here: static part will already have
           // been evaluated and found to match.
           InterceptorAndDynamicMethodMatcher dm =
                (InterceptorAndDynamicMethodMatcher)interceptorOrInterceptionAdvice;
           //动态匹配：运行时参数是否满足匹配条件
           if (dm.methodMatcher.matches(this.method, this.targetClass,this.arguments)) {
                //执行当前Intercetpor
                returndm.interceptor.invoke(this);
           }
           else {
                //动态匹配失败时,略过当前Intercetpor,调用下一个Interceptor
                return proceed();
           }
       }
       else {
           // It's an interceptor, so we just invoke it: The pointcutwill have
           // been evaluated statically before this object was constructed.
           //执行当前Intercetpor
           return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
       }
}
```

## Spring MVC

### 指定 Spring mvc 的入口程序 (web.xml)

```xml
<servlet>
    <servlet-name> dispatcher </servlet-name>
    <servlet-class> org.springframework.web.servlet.DispatcherServlet </servlet-class>
    <load-on-startup>1</load-on-startup>
</servlet>

<servlet-mapping>
    <servlet-name> dispatcher </servlet-name>
    <url-pattern> /** </url-pattern>
</servlet-mapping>
```

DispatcherServlet 是核心分派器, 是 Spring MVC 最重要的类之一, 之后我们会对其单独展开分析

### 编写 Spring mvc 的核心配置文件 ([servlet-name]-servlet.xml)

xml 中前面的 beans 就不写了

```xml
    <!-- Enables the Spring MVC @Controller programming model -->  
    <mvc:annotation-driven />  
    
    <context:component-scan base-package="com.abc" />
    
    <bean class = "org.springframework.web.servlet.view.InternalResourceViewResolver">
        <property name = "prefix" value = "/" />
        <property name = "suffix" value = ".jsp" />
    </bean>
```

用于配置 view 的位置以及扫描范围

### 编写 Controller 代码

```java
@Controller
@RequestMapping
public class UserController {
    @RequestMapping("login")
    public ModelAndView login(String name, String password) {
        return new ModelAndView("success");
    }
}
```

### 核心分派器

Servlet 本质上是一个 Java 类, 通过 servlet 接口定义的 HttpServletRequest 对象,
我们可以处理整个请求生命周期的数据, 通过 HttpServletResponse 对象, 我们可以处理 Http 响应行为。

```java
try 
    mappedHandler = getHandler(processRequest, false);
    if(mappedHandler == null || mappedHandler.getHandelr() == null)
        noHandlerFound(processRequest, response)
        return;
    
    HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler())
    
    mv = ha.handle(processedRequest, response, mappedHandler.getHandler())
  catch (ModelAndViewDefinitionException)
    mv = ex.getModelAndView
  catch (Exception)
    Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null)
    mv = processHandlerException(processedRequest, response, handler, ex);
    errorView = (mv != null)
```

HandlerMapping 与其子类:(需要举例子)

**BeanNameUrlHandlerMapping:** 根据 spring 容器中的 bean 的定义来指定请求映射关系

**SimpleUrlHandlerMapping:** 直接指定 URL 与 Controller 的映射关系, 其中 URL 支持 Ant 风格

**DefaultAnnotationHandlerMapping:** 支持通过直接扫描 Controller 类中的 Annotation 来确定
请求映射关系

**RequestMappingHandlerMapping** 通过扫描 Controller 类中的 Annotation 来确定请求映射关系的另一个实现类

## 原理

![](/images/posts/spring/springmvc-prin.jpg)


## 基础

### 注解

**@Autowired**


示例1, autowire 参数, 并根据名字过滤, 这两个注解可以替换为 Resource(name="redBean")

```java

// 产生一个 Bean
@Qualifier("redBean") 
class Red implements Color { // Class code here }

or 
<bean id="redBean" class="com.mycompany.movies.Red"/>

@Autowire
@Qualifier("redBean")
public void setColor(Color color) {
    this.color = color;
}
```

autowire 成员变量

```
@Controller // defines that this class is a spring bean 
@RequestMapping("/users") 
public class SomeController { 

    // tells the application context to inject an instance of UserService here 
    @Autowired private UserService userService;
```

## 实现

### 常用接口

```java
interface HandlerMapping
    HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception;

class HandlerExecutionChain
    private final Object handler;
    private HandlerInterceptor[] interceptors;
    applyPreHandle(HttpServletRequest request, HttpServletResponse response)
    applyPostHandle(HttpServletRequest request, HttpServletResponse response, ModelAndView mv)
    triggerAfterCompletion(HttpServletRequest request, HttpServletResponse response, Exception ex)

interface HandlerInterceptor
    preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
    postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView)
    void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex)

interface HandlerAdapter
    boolean supports(Object handler);
    ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
    
```


## Spring mvc 的设计原则

### Open for extension closed for modification

**使用 final 关键字来限定核心组件的核心方法**

Some methods in the core classes of spring web mvc are marked as final. This has
not been done arbitrarily, but specifically with the principle in mind.

HandlerAdapter 实现类 RequestMappingHandlerAdapter 中, 核心方法 `handleInternal` 就被定义为 final

**大量使用 private 方法**






