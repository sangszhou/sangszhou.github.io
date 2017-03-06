---
layout: post
title:  "Java logging system Part 2"
date:   "2016-02-14 17:50:00"
categories: java
keywords: java
---

### 问题与解决问题

需求: 基于 Storm 的 Data Injection System 需要为不同的 Worker 打印的 log 分门别类，而默认情况下，storm 会把这些内容全部打到一起，
所以，需要手动做些工作，把不同 worker 打印的 log 分开

### 背景知识

写的很好，复制了 [这篇文章](http://www.jianshu.com/p/d1a565f192b9)

在后端logger系统中，有三个最基础的概念需要先熟悉: **Logger**, **Appender**, **Layout(Encoder)**

1. Logger 日志记录器 - 日志记录器就是一个普通的Java类而已
2. Appender 输出源 - 输出源是日志最终输出的地方，比如控制台或者文件
3. Layout(Encoder) 布局 - 布局决定了日志的打印格式，比如使用 %r [%t] %-5p %c %x - %m%n可以打印出 467 [main] INFO org.apache.log4j.examples.Sort - Exiting main method.这样的日志。

默认情况下 logback 会在 classpath 下寻找名为 logback.xml 的配置文件（还有另外4种查找配置文件的方式，见文档）如果没有找到任何配置文件，会使用 **BasicConfigurator** 初始化一个默认配置，它会把结果打印到控制台，这就是上面 “HelloWorld” 示例的情况。为了让 logback 可以加载到配置文件，需要在 src/main/resources/ 目录下创建文件 logback.xml（Gradle会自动把 resources 目录下的文件放到classpath目录）。

```java
public class BasicConfigurator extends ContextAwareBase implements Configurator {

    public BasicConfigurator() {
    }

    public void configure(LoggerContext lc) {
        addInfo("Setting up default configuration.");
        
        ConsoleAppender<ILoggingEvent> ca = new ConsoleAppender<ILoggingEvent>();
        ca.setContext(lc);
        ca.setName("console");
        LayoutWrappingEncoder<ILoggingEvent> encoder = new LayoutWrappingEncoder<ILoggingEvent>();
        encoder.setContext(lc);
        
 
        // same as 
        // PatternLayout layout = new PatternLayout();
        // layout.setPattern("%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n");
        TTLLLayout layout = new TTLLLayout();
 
        layout.setContext(lc);
        layout.start();
        encoder.setLayout(layout);
        
        ca.setEncoder(encoder);
        ca.start();
        
        Logger rootLogger = lc.getLogger(Logger.ROOT_LOGGER_NAME);
        rootLogger.addAppender(ca);
    }
}
```
所以在没有配置任何 log 的时候，默认的 log 会打印到 console

```xml
<configuration>
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <!-- encoders are assigned the type
         ch.qos.logback.classic.encoder.PatternLayoutEncoder by default -->
    <encoder>
      <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
    </encoder>
  </appender>

  <root level="debug">
    <appender-ref ref="STDOUT" />
  </root>
</configuration>
```

这是和默认 log 等效的配置

### 日志记录器 - logger

在上面的 "HelloWorld" 示例中，我们通过 Logger logger = LoggerFactory.getLogger(App.class); 工厂方法获得了一个日志记录器。并通过 logger.debug("Hello world."); 方法打印了一条信息，但是如果我们通过 logger.info 方法是打印不出来的，原因是什么呢？

在logback中日志记录器是继承的，继承的规则是 com.hello.foo 会继承 com.hello 的日志配置，父子关系通过 . 来分割，所以 com 是 com.hello 的父节点。在logback中默认会有一个 root-logger（根 - 日志记录器）的存在，**所有的其他日志记录器都会默认继承它的配置**。在配置文件中看到的:

```xml
<root level="debug">
  <appender-ref ref="STDOUT" />
</root>
```

就是“根”。所以当我们调用 Logger logger = LoggerFactory.getLogger(App.class); 的时候，默认是从 root-logger 那里继承了日志输出配置，而 root-logger 默认的log打印级别是 debug，所以用 logger.info 打印不出日志。

### 日志输出源 - Appenders

```java
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
    </encoder>
  </appender>
```

这个输出源配置把日志打印到控制台，格式为 %d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n，输出源的 name 和 class 属性是必须配置的选项。比如还可以配置把日志打印到文件：

```xml
<appender name="FILE" class="ch.qos.logback.core.FileAppender">
    <file>myApp.log</file>
    <encoder>
      <pattern>%date %level [%thread] %logger{10} [%file:%line] %msg%n</pattern>
    </encoder>
</appender>
```

有了输出源之后，就可以给logger配置输出源，**一个logger可以配置多个输出源**：

在logback中，level的配置会被继承，但是appender的配置会被子logger保留。这么说有点抽象，看下面的例子：

```xml
<configuration>

  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
    </encoder>
  </appender>

  <logger name="com.hello">
    <appender-ref ref="STDOUT" />
  </logger>

  <root level="debug">
    <appender-ref ref="STDOUT" />
  </root>
</configuration>
```

这个配置会导致控制台打印两次：

```
11:38:06.068 [main] INFO  com.hello.App - Hello world.Info
11:38:06.068 [main] INFO  com.hello.App - Hello world.Info
```

当调用 LoggerFactory.getLogger("com.hello.App") 的时候，它继承了来自 <logger name="com.hello"> 和 root-logger 的输出源，而他们的输出源都是控制台，所以导致在控制台输出了两次。解决办法是，要么在有继承关系的logger中配置不同的输出源（从业务上避免），要么在子logger中覆盖父节点中的配置。可以通过 additivity="false" 配置实现：

```xml
<logger name="com.hello" additivity="false">
    <appender-ref ref="STDOUT" />
</logger>
```

additivity 属性告诉logback不要继承来自父类的设置。

### 布局 - Layout 和 编码 - Encoder

Layouts are logback components responsible for transforming an incoming event into a String.

Layout主要用来把log事件转换成String。一般布局都是配置在 Appender 里面的：

```xml
<appender name="CONSOLE" class="org.apache.log4j.ConsoleAppender">
    <layout class="org.apache.log4j.PatternLayout">
        <param name="ConversionPattern" value="%d{yyyy-MM-dd HH:mm:ss} [%c]-[%p] %m%n" />
    </layout>
</appender>
```

注意，上面的示例使用的Appender是org.apache.log4j.ConsoleAppender ，这是log4j的配置而不是这里讲的logback，这是因为在过去日志系统确实都是使用Layout来做日志转换的，但是由于一些 固有的问题 ，logback在Layout上面又封装了一层 - Encoder，表现在配置上就是这样的（这才是logback的配置）：

```xml
<appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
    </encoder>
  </appender>
```

但是Encoder的出现并不影响我们的配置，只是在形式上多了一个<encoder/>标签。一般使用最多的是 PatternLayout 或 PatternLayouEncoder ，他们的特点是提供了很多约定的标记，这些标记都以 % 开头，比如logger名称、日志等级日期、线程名等。比如：%level [%thread]: %message%n可以打印出日志等级，线程名字和日志本身。

Layouts, as discussed in detail in the next chapter, are only able to transform an event into a String. Moreover, given that a layout has no control over when events get written out, layouts cannot aggregate events into batches. Contrast this with encoders which not only have total control over the format of the bytes written out, but also control when (and if) those bytes get written out.

### 常用配置

**1. 变量定义和替换**

```java
<configuration>
  <property name="USER_HOME" value="/home/sebastien" />
  <appender name="FILE" class="ch.qos.logback.core.FileAppender">
    <file>${USER_HOME}/myApp.log</file>
    <encoder>
      <pattern>%msg%n</pattern>
    </encoder>
  </appender>
  <root level="debug">
    <appender-ref ref="FILE" />
  </root>
</configuration>
```

这里定义了一个变量 <property name="USER_HOME" value="/home/sebastien" /> 并通过 ${USER_HOME} 引用。变量定义的意义在于可以对一些全局性的属性/变量做统一的配置。如果有很多变量的时候，可以把变量定义在单独的文件中，通过 <property file="src/main/java/chapters/configuration/variables1.properties" /> 来引用。

```
// variables1.properties
USER_HOME=/home/sebastien
fileName=myApp.log
destination=${USER_HOME}/${fileName}
```