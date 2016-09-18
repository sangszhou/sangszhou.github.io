---
layout: post
title: design pattern 
categories: [design pattern]
keywords: java
---


## @todo 多态?

### 什么是多态

1. 编译时多态：编译时动态重载；

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

## 开闭原则

Software entities should be open for extension,but closed for modification.

应该满足将来在不可修改源代码的情况下对模块的职能扩展，或者改变模块的行为



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



## 参考资料

[Spring Interview Questions](http://ifeve.com/java-memory-model-0/)

