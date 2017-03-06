---
layout: post
title:  "Java logging system Part 1"
date:   "2016-02-14 17:50:00"
categories: java
keywords: java
---


### logback impl

**使用**

需要的 jar 包

1. logback-core
2. logback-classic
3. slf4j-api

**使用方式**

```java
private static final Logger logger=LoggerFactory.getLogger(LogbackTest.class);

public static void main(String[] args){
    if(logger.isDebugEnabled()){
        logger.debug("slf4j-logback debug message");
    }
    if(logger.isInfoEnabled()){
        logger.debug("slf4j-logback info message");
    }
    if(logger.isTraceEnabled()){
        logger.debug("slf4j-logback trace message");
    }

    LoggerContext lc = (LoggerContext) LoggerFactory.getILoggerFactory();
    StatusPrinter.print(lc);
}
```

官方使用方式，其实就是和 slf4j 集成起来了, 上述的 Logger, LoggerFactory 都是 slf4j 自己的接口类

没有配置文件的情况下，使用的是默认配置。搜寻配置文件的过程如下：

1. Logback tries to find a file called logback.groovy in the classpath.
2. If no such file is found, logback tries to find a file called logback-test.xml in the classpath
3. If no such file is found, it checks for the file logback.xml in the classpath
4. If no such file is found, and the executing JVM has the ServiceLoader (JDK 6 and above) the ServiceLoader will be used to resolve an implementation of com.qos.logback.classic.spi.Configurator. The first implementation found will be used. See ServiceLoader documentation for more details
5. If none of the above succeeds, logback configures itself automatically using the BasicConfigurator which will cause logging output to be directed to the console

也可以在类路径下加上一个类似如下的logback.xml的配置文件，如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
    </encoder>
  </appender>

  <root level="DEBUG">          
    <appender-ref ref="STDOUT" />
  </root>  

</configuration>
```

logback则会去解析对应的配置文件。

**使用过程简单分析**

slf4j与底层的日志系统进行绑定

在jar包中寻找org/slf4j/impl/StaticLoggerBinder.class 这个类，如在logback-classic中就含有这个

1. StaticLoggerBinder.class
2. StaticMarkerBinder.class
3. StaticMDCBinder.class

如果找到多个StaticLoggerBinder，则表明目前底层有多个实际的日志框架，slf4j会随机选择一个

使用上述找到的StaticLoggerBinder创建一个实例，并返回一个ILoggerFactory实例：

return StaticLoggerBinder.getSingleton().getLoggerFactory()；

以logback-classic中的StaticLoggerBinder为例，在StaticLoggerBinder.getSingleton()过程中：会去加载解析配置文件 源码如下：

```java
public URL findURLOfDefaultConfigurationFile(boolean updateStatus) {

   ClassLoader myClassLoader = Loader.getClassLoaderOfObject(this);
   //寻找logback.configurationFile的系统属性
   URL url = findConfigFileURLFromSystemProperties(myClassLoader, updateStatus);
   if (url != null) {
     return url;
   }
   //寻找logback.groovy
   url = getResource(GROOVY_AUTOCONFIG_FILE, myClassLoader, updateStatus);
   if (url != null) {
     return url;
   }
   //寻找logback-test.xml
   url = getResource(TEST_AUTOCONFIG_FILE, myClassLoader, updateStatus);
   if (url != null) {
     return url;
   }
   //寻找logback.xml
   return getResource(AUTOCONFIG_FILE, myClassLoader, updateStatus);
}
```

目前路径都是定死的，只有logback.configurationFile的系统属性是可以更改的，所以如果我们想更改配置文件的位置（不想放在类路径下），则需要设置这个系统属性：

目前路径都是定死的，只有logback.configurationFile的系统属性是可以更改的，所以如果我们想更改配置文件的位置（不想放在类路径下），则需要设置这个系统属性：

System.setProperty("logback.configurationFile", "/path/to/config.xml");

根据返回的ILoggerFactory实例，来获取Logger

就是根据上述的LoggerContext来创建一个Logger，每个logger与LoggerContext建立了关系，并放到LoggerContext的缓存中，就是LoggerContext的如下属性：

private Map<String, Logger> loggerCache;

其实上述过程就是slf4j与其他日志系统的绑定过程。不同的日志系统与slf4j集成，都会有一个StaticLoggerBinder类，并会拥有一个ILoggerFactory的实现。



### 日志演化历史

最开始出现的是log4j，也是应用最广泛的日志系统，成为了目前java日志系统事实上的标准，一切都是美好的

但java的开发主体sun公司认为自己才是正统，为了干掉log4j在jdk1.4中增加了 jul (因为在java.util.logging包下)日志的实现，造成了目前开发者的混乱，迄今为止仍饱受诟病

各个日志系统互相没有关联，替换和统一变的非常麻烦。A项目用log4j作为日志系统，但同时引了B项目，而B项目用jul作为日志系统，那么你的应用就得使用两个日志系统

为了搞定这个坑爹的问题，开源社区apache提供了一个日志框架作为日志的抽象，叫commons-logging，也被称为jcl(java common logging)，jcl对各种日志接口进行抽象，抽象出一个接口层，对每个日志实现都适配或者转接，这样这些提供给别人的库都直接使用抽象层即可，较好的解决了上述问题

但这时，作为元老级日志log4j的作者觉得jcl不够好，再度出山，搞出了一个更加牛逼的新的日志框架slf4j(这个也是抽象层)，同时针对slf4j的接口实现了一套日志系统，即传说中的 logback

同时这个作者心情一好，又把log4j进行了改造，就是所谓的log4j2，同时支持jcl以及slf4j

### JCL

使用JCL一般(如果是log4j可以不需要)需要一个配置commons-logging.properties在classpath上，这个文件有一行代码：

org.apache.commons.logging.LogFactory= org.apache.commons.logging.impl.LogFactoryImpl

用于通知JCL想要使用哪个日志系统。用户只要修改LogFactory的实现，就能动态的切换日志系统

### SLF4J

slf4j的设计相对较为精巧，作者将接口以及实现分开，其中slf4j-api中定义了接口，开发者需要关心的就是这个接口，无需再关心下层如何实现，同时各个是slf4j接口的实现者只要遵循这个接口，就能够做到日志系统间的无缝兼容

slf4j-api的实现目前比较出名的是接口开发者实现的logback，性能相较log4j来讲更加优秀，也支持占位符等新特性。

同时为了兼容log4j，slf4j-log4j12实现了slf4j-api，作为log4j兼容的适配器，使得用户用起来像在用log4j

为了兼容jul，slf4j-jdk14也实现了slf4j-api，作为jul兼容的适配器，使得用户用起来像在用jul


有了实现，还需要一个桥接器，桥接器将对jcl的调用转接到slf4j上：

1. jul-to-slf4j把对jul的调用桥接到slf4j上
2. jcl-over-slf4j把对jcl的调用桥接到slf4j上
3. log4j-over-slf4j把对log4j的调用桥接到slf4j上

### 日志系统的冲突

多种日志系统并不意味着程序中只能存在一种日志系统，比如log4j与logback是完全可以共存的。

目前日志系统的冲突主要分为两种：
 
1. 同一个日志系统的多个实现
2. 桥接接口与实现类

像slf4j接口实现的冲突，如:

slf4j-log4j、logback、slf4j-jdk14、log4j2之间的冲突

这点很好理解，这几个包都实现了slf4j的接口，同一接口只能有一个实现才能被 jvm 正确识别

但日志系统仍然非常容易冲突，与传统的jar冲突相同，当jvm发现两个一模一样的实现的时候，它就不知道选择哪个或选择了一个错误的，就会提示ClassNotFound.

slf4j为纯日志接口

logback/slf4j-jdk14/slf4j-log4j12/log4j2均可认为是slf4j的实现类

jul/log4j作为最早开始的日志系统，本身就是一种日志的实现，没有任何框架

jul-to-slf4j/log4j-to-slf4j/jcl-over-slf4j/作为桥接接口，可以将原有接口的内容进行接收，然后通过slf4j进行输出

**其余常见问题**

1. slf4j的几个实现jar包一定会冲突，尤其要注意
2. slf4j-api和实现版本最好对应，尤其是1.6.x和1.5.x不兼容，直接升级到最新版本
3. log4j与logback可以同时使用，logback实现slf4j的接口，与log4j没有任何关系，完全可以同时使用
4. 建议选用slf4j + logback，或slf4j + log4j2
5. 日志系统在正常情况下是不会影响应用性能的，但应该注意量，太大量的日志会拖累性能

### 实现


```java
    private static String STATIC_LOGGER_BINDER_PATH = "org/slf4j/impl/StaticLoggerBinder.class";

    static Set<URL> findPossibleStaticLoggerBinderPathSet() {
        // use Set instead of list in order to deal with bug #138
        // LinkedHashSet appropriate here because it preserves insertion order
        // during iteration
        Set<URL> staticLoggerBinderPathSet = new LinkedHashSet<URL>();
        try {
            ClassLoader loggerFactoryClassLoader = LoggerFactory.class.getClassLoader();
            Enumeration<URL> paths;
            if (loggerFactoryClassLoader == null) {
                paths = ClassLoader.getSystemResources(STATIC_LOGGER_BINDER_PATH);
            } else {
                paths = loggerFactoryClassLoader.getResources(STATIC_LOGGER_BINDER_PATH);
            }
            while (paths.hasMoreElements()) {
                URL path = paths.nextElement();
                staticLoggerBinderPathSet.add(path);
            }
        } catch (IOException ioe) {
            Util.report("Error getting resources from path", ioe);
        }
        return staticLoggerBinderPathSet;
    }
    
    private final static void bind() {
        try {
            Set<URL> staticLoggerBinderPathSet = null;
            // skip check under android, see also
            // http://jira.qos.ch/browse/SLF4J-328
            if (!isAndroid()) {
                staticLoggerBinderPathSet = findPossibleStaticLoggerBinderPathSet();
                reportMultipleBindingAmbiguity(staticLoggerBinderPathSet);
            }
            // the next line does the binding
            StaticLoggerBinder.getSingleton();
            INITIALIZATION_STATE = SUCCESSFUL_INITIALIZATION;
            reportActualBinding(staticLoggerBinderPathSet);
            fixSubstituteLoggers();
            replayEvents();
            // release all resources in SUBST_FACTORY
            SUBST_FACTORY.clear();
        } catch (NoClassDefFoundError ncde) {
        
    }
```

