---
layout: post
title: Storm Spring Integration Survey
categories: [Spring]
description: java
keywords: java, thread
---

我想实现 Spring 与 Storm 的集成，目的是让开发者能够在 Storm 程序中使用注解，比如 Service, Configuration, Component, Autowired 等等。
这样做的好处之一类型的定义和实现解耦，这也是 Spring 使用注解的初衷，还有一个是我希望在一个 Topology 的运行上下文放入一些信息，这些信息能够让用户在任何地方都可以拿到，而实现这个目标只有两个方式可选，第一个是 annotation, 第二个是 ThreadLocal. 相比于 annotation, Storm 的线程使用方式更不好把握，所以 ThreadLocal 不太好用。Annotation 似乎是唯一的选择。还有第三点好处，用户如果需要一些第三方的库，比如 kafka, elasticsearch, mongo 时，只需要在配置文件中声明需要的配置，然后在使用的地方 Autowire 一下，就可以使用了，不需要自己再去 new 一个 client。比如

```java
class DummySpout extends Spout {
    @Autowire
    ElasticsearchClient esClient;
    
    @Autowire
    RestTemplate restTemplate;
    
    @Autowire
    TopologyContext context;
}
```


```
elasticsearch {
    addr: ["host1:9200", "host2:9200]
}
```

当 topology 运行时， RestTemplate 是我写好的，织入过 log 代码的实现，es client 就按照用户给出的配置创建，而 Context 也是我预先写好的。



google 了一下，看到若干资源。github 上有一个 [storm-spring](https://github.com/granthenke/storm-spring) 项目，看起来 promising, 我看了下，发现它做的事和我期望的并不相同，storm-spring 只是把用来创建 bean 和组织 topology, 而是在代码内部使用注解。另外一个项目是 [storm-spring-autowire](https://github.com/zzxzz12345/storm-spring-autowire), 它支持最基本的依赖注入：

```java
@Resource
private SentenceCounter counter;

public void prepare(Map stormConf, TopologyContext context, OutputCollector collector) {
  BeanUtil.autoware(this);
}
```

从这段代码中也大概看得出它是怎么实现的，BeanUtil 会把 this 指针传到处理函数中，处理函数拿到 this 指针，遍历带有 Resource 和 Autowire 的 field, 对于每一个 field 到 ApplicationContext 中查找并赋值。下面是 BeanUtil 的实现

```java
public class BeanUtil {
  private static ApplicationContext context;

  static {
    context = new ClassPathXmlApplicationContext("applicationContext.xml");
  }
  public static void autoware(Object obj) {
    Field[] fields = obj.getClass().getDeclaredFields();
    for (Field field : fields) {
      Resource resource = field.getAnnotation(Resource.class);
      Autowired autowired = field.getAnnotation(Autowired.class);
      if (resource != null) {
        Object bean;
        if (resource.name() != null && !resource.name().equals("")) {
          bean = context.getBean(resource.name());
        } else if (context.containsBean(field.getName())) {
          bean = context.getBean(field.getName());
        } else {
          bean = context.getBean(field.getType());
        }
        field.setAccessible(true);
        try {
          field.set(obj, bean);
        } catch (IllegalAccessException e) {
          throw new RuntimeException(e);
        }
      }

      if (autowired != null) {
        Object bean;
        field.setAccessible(true);
        Qualifier qualifier = field.getAnnotation(Qualifier.class);
        if (qualifier != null && qualifier.value() != null && !qualifier.value().equals("")) {
          bean = context.getBean(qualifier.value());
        } else {
          bean = context.getBean(field.getType());
        }
        try {
          field.set(obj, bean);
        } catch (IllegalAccessException e) {
          throw new RuntimeException(e);
        }
      }
    }
  }
}
```

所以这个只是看起来像 spring 而已，annotation 还需要自己扫描，自己注入，创建 bean 的方式还是通过 xml file. 依然不太理想。

只能自己解决了。从 google 检索到的结果来看，这件事似乎做不成，大概有这么几个问题：

1. Storm worker JVM 之间传递 ApplicationContext 的问题，ApplicationContext 是不可序列化的
2. ApplicationContext 在纯 Annotation 的场景下如何创建的，如果是 ClassPathXMLApplicationContext 下就是我们手动 new 的，在纯 annotation 下似乎是靠 @ContextConfiguration 驱动创建的，但是这个东西应该写在哪呢，storm 程序又没有 main class
3. Annotation 一般是靠类型来选取 instance, 怎么排除某些 package 下的 annotation, 有 cluster 和 local mode 下一个类型各有是个实例，每次运行只能选取其中一个 package 下的实现

难点主要是 1，2，要弄明白就得看源码

## How spring annotation dependency injections works

在 Spring boot 项目中，只要给出 @SpringBootApplication, Spring 就会自动完成 Annotation 类型的 Bean 创建和注入过程，所以 DI 的工作原理肯定和这个 annotation 相关, SpringBootApplication 又分为若干个子 annotation

**@Configuration**

带有 @Configuration 的注解类表示这个类可以使用 Spring IoC 容器作为 bean 定义的来源。@Bean 注解告诉 Spring，一个带有 @Bean 的注解方法将返回一个对象，该对象应该被注册为在 Spring 应用程序上下文中的 bean。最简单可行的 @Configuration 类如下所示：

```java
package com.tutorialspoint;
import org.springframework.context.annotation.*;
@Configuration
public class HelloWorldConfig {
   @Bean 
   public HelloWorld helloWorld(){
      return new HelloWorld();
   }
}
```

等同于

```xml
<beans>
   <bean id="helloWorld" class="com.tutorialspoint.HelloWorld" />
</beans>
```

在这里，带有 @Bean 注解的方法名称作为 bean 的 ID，它创建并返回实际的 bean。你的配置类可以声明多个 @Bean。一旦定义了配置类，你就可以使用 AnnotationConfigApplicationContext 来加载并把他们提供给 Spring 容器，如下所示：

```java
public static void main(String[] args) {
   ApplicationContext ctx = new AnnotationConfigApplicationContext(HelloWorldConfig.class); 
   HelloWorld helloWorld = ctx.getBean(HelloWorld.class);
   helloWorld.setMessage("Hello World!");
   helloWorld.getMessage();
}
```

More refer to [极客学院](http://wiki.jikexueyuan.com/project/spring/java-based-configuration.html)

看到这，我发现了除了 ClassPathXMLApplicationContext, FilesystemApplicationContext 外还有一个 AnnotationConfigApplicationContext, 这正好是我需要的东西, 有了这个 Context 就能拿到那些声明 @Component 的类了，接下来需要知道 Autowire 是怎么工作的

## How Storm works in detail

先看这篇文章，从比较高层的角度讲了并发的过程
[Understanding-the-parallelism-of-a-Storm-topology](http://storm.apache.org/releases/1.0.2/Understanding-the-parallelism-of-a-Storm-topology.html)

![](/images/posts/bigdata/storm_topology_archi.png)

Storm集群中有两种节点，一种是控制节点(Nimbus节点)，另一种是工作节点(Supervisor节点)。所有Topology任务的 提交必须在Storm客户端节点上进行(需要配置 storm.yaml文件)，由Nimbus节点分配给其他Supervisor节点进行处理。 Nimbus节点首先将提交的Topology进行分片，分成一个个的Task，并将Task和Supervisor相关的信息提交到 zookeeper集群上，Supervisor会去zookeeper集群上认领自己的Task，通知自己的Worker进程进行Task的处理。 

和同样是计算框架的MapReduce相比，MapReduce集群上运行的是Job，而Storm集群上运行的是Topology。但是Job在运行结束之后会自行结束，Topology却只能被手动的kill掉，否则会一直运行下去 

Storm中，Spout和Bolt都是Component。所以，Storm定义了一个名叫IComponent的总接口 
全家普如下：绿色部分是我们最常用、比较简单的部分。红色部分是与事务相关的 

![](/images/posts/bigdata/storm_topology_archi.png)

**Spout**

Spout 是Stream的消息产生源， Spout组件的实现可以通过继承BaseRichSpout类或者其他Spout类来完成，也可以通过实现IRichSpout接口来实现 

```java
public interface ISpout extends Serializable { 
  void open(Map conf, TopologyContext context, SpoutOutputCollector collector);  // 初始化方法 
  
  //在该spout将要关闭时调用。但是不保证其一定被调用，因为在集群中supervisor节点，可以使用kill -9来杀死worker进程。
  //只有当Storm是在本地模式下运行，如果是发送停止命令，可以保证close的执行
  void close();  // 
  
  
  void nextTuple(); 
  // 成功处理tuple时回调的方法，通常情况下，此方法的实现是将消息队列中的消息移除，防止消息重放 
  void ack(Object msgId); 
  
  // 处理tuple失败时回调的方法，通常情况下，此方法的实现是将消息放回消息队列中然后在稍后时间里重放 
  void fail(Object msgId); 
} 
```

nextTuple() -- 这是Spout类中最重要的一个方法。发射一个Tuple到Topology都是通过这个方法来实现的。调用此方法时，storm向spout发出请求，让spout发出元组（tuple）到输出器（ouput collector）。这种方法应该是非阻塞的，所以spout如果没有元组发出，这个方法应该返回。nextTuple、ack 和fail 都在spout任务的同一个线程中被循环调用。 当没有元组的发射时，应该让nextTuple睡眠一个很短的时间（如一毫秒），以免浪费太多的CPU。 

继承了BaseRichSpout后，不用实现close、 activate、 deactivate、 ack、 fail 和 getComponentConfiguration 方法，只关心最基本核心的部分。 

通常情况下（Shell和事务型的除外），实现一个Spout，可以直接实现接口IRichSpout，如果不想写多余的代码，可以直接继承BaseRichSpout 

**Bolt**

Bolt类接收由Spout或者其他上游Bolt类发来的Tuple，对其进行处理。Bolt组件的实现可以通过继承BasicRichBolt类或者IRichBolt接口等来完成 

prepare方法 -- 此方法和Spout中的open方法类似，在集群中一个worker中的task初始化时调用。 它提供了bolt执行的环境 

declareOutputFields方法 -- 用于声明当前Bolt发送的Tuple中包含的字段(field)，和Spout中类似 

cleanup方法 -- 同ISpout的close方法，在关闭前调用。同样不保证其一定执行。 

execute方法 -- 这是Bolt中最关键的一个方法，对于Tuple的处理都可以放到此方法中进行。具体的发送是通过emit方法来完成的。execute接受一个tuple进行处理，并用prepare方法传入的OutputCollector的ack方法（表示成功）或fail（表示失败）来反馈处理结果。 

Storm提供了IBasicBolt接口，其目的就是实现该接口的Bolt不用在代码中提供反馈结果了，Storm内部会自动反馈成功。如果你确实要反馈失败，可以抛出FailedException 

通常情况下，实现一个Bolt，可以实现IRichBolt接口或继承BaseRichBolt，如果不想自己处理结果反馈，可以实现 IBasicBolt 接口或继承 BaseBasicBolt，它实际上相当于自动实现了collector.emit.ack(inputTuple) 

**Topology运行流程**

