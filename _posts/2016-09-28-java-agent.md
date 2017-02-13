---
layout: post
title: java agent and aspectj
categories: [java]
keywords: agent, java
---

## aspectj

下面介绍了如何使用 BTrace 动态替换 .class 文件, 其实我们也可以手动替换 .class 文件，而 aspectj 就是这么一个工具，可以用来在 compile 或者 runtime 替换文件

先说 runtime, runtime 是指一个正在运行的进程或者将要进行的进程，这个进程是一个 java 程序，但是它并不在我们的管辖范围内，比如一个 hadoop server 程序不受我们管控，或者 `hadoop fs -l` 命令对应的 hadoop client, 它是一个 jar 但是并不是我们程序所依赖的东西。这个时候，只能使用 runtime swap. 实现方法是先写一个 interceptor.jar, jar 中有一个 aspect 文件指明要替换的方法，类型和织入的代码，把这个 jar 包和 aspectj-weaver.jar 放到 Classpath 下，当 hadoop-client.jar 执行的时候就会调用 aspectj-weaver 这个执行引擎并调用 inteceptor.jar 中织入的代码。这个过程可能说的不对或者有遗漏的地方

创建 jar, 指定搜寻 class 文件的范围: aop-ajc.xml 并发送到远端服务器

```xml
<aspectj>
	<aspects>
		<aspect name="hadoop.FileSystemInterceptor"/>
	</aspects>

	<weaver options="-warn:none -Xlint:default Xlint:ignore -Xset:weaveJavaxPackages=true -Xset:weaveJavaPackages=true">
		<!-- <exclude within="org.apache.hadoop.conf..*">
		<exclude within="org.apache.hadoop.crypto..*">
		<exclude within="org.apache.hadoop.ha..*">
		<exclude within="org.apache.hadoop.http..*">
		<exclude within="org.apache.hadoop.io..*">	
		<exclude within="org.apache.hadoop.ipc..*">
		<exclude within="org.apache.hadoop.jmx..*">
		<exclude within="org.apache.hadoop.log..*">
		<exclude within="org.apache.hadoop.metrics..*">
		<exclude within="org.apache.hadoop.metrics2..*">
		<exclude within="org.apache.hadoop.net..*">	
		<exclude within="org.apache.hadoop.record..*">
		<exclude within="org.apache.hadoop.security..*">
		<exclude within="org.apache.hadoop.service..*">			
		<exclude within="org.apache.hadoop.tools..*">
		<exclude within="org.apache.hadoop.tracing..*">
		<exclude within="org.apache.hadoop.util.*."> -->

		<!-- <include within="org.apache.hadoop.fs.*"/>
		<include within="org.apache.hadoop.fs.shell.*"/> -->
		<include within="com.mapr.fs..*"/>
	</weaver>	
</aspectj>
```

```
使用 aspectj 编译 jar

ajc -1.7 -d bin -cp $CLASSPATH:/ws/aspectj/src/lib/jackson-annotations-2.1.2.jar:/ws/aspectj/src/lib/com.fasterxml.jackson.core.jar:/ws/aspectj/src/lib/com.fasterxml.jackson.databind.jar:/ws/aspectj/src/lib/hadoop-common-2.7.0.jar:/ws/aspectj/src/lib/ezmorph-1.0.6.jar:/ws/aspectj/src/lib/json-lib-2.4-jdk15.jar:/ws/aspectj/src/lib/httpclient-4.5.2.jar:/ws/aspectj/src/lib/httpcore-4.4.4.jar:/ws/aspectj/src/lib/jackson-core-asl-1.8.5.jar:/ws/aspectj/src/lib/jackson-mapper-asl-1.8.5.jar:/ws/aspectj/src/lib/commons-beanutils-1.9.2.jar:/ws/aspectj/src/lib/commons-collections4-4.1.jar:/ws/aspectj/src/lib/commons-lang3-3.4.jar:/ws/aspectj/src/lib/commons-logging-1.2.jar src/utils/HadoopEvent.java src/hadoop/FileSystemInterceptor.java -outxml -outjar HadoopMonitorProxyClient.jar


jar uvf HadoopMonitorProxyClient.jar META-INF/aop-ajc.xml

scp HadoopMonitorProxyClient.jar vdeadmin@host1:/users/vdeadmin/hdfs_open_hook/HadoopMonitorProxy

```

**compile-time weaver**

当我们的程序本身一个单独 run 的东西时，就使用 compile time 织入，它的实现方式是替换 class 文件，可以在 target 文件夹下查看 class 文件被修改后的样子

在我的 gitlab 上有例子

## Overview

**JVM instrumentation feature** was introduced in JDK 1.5 and is based on byte code instrumentation (BCI). Actually, when a class is loaded, 
you can alter the corresponding byte code to introduce features such as **methods execution profiling** or **event tracing**. 
Most of Java **Application Performance Management** (APM) solutions use this mechanism to monitor JVM.

![](/images/posts/java/java-agent-overview-min.png)

## Instrumentation Agent

To enable JVM instrumentation,  you have to provide an agent (or more) that is deployed as a JAR file. An attribute in the JAR file manifest specifies the agent class which will be loaded to start the agent.

**with a command-line interface**: by adding this option to the command-line: -javaagent:jarpath[=options] where jarpath is the path to 
the agent JAR file. options is the agent options. This switch may be used multiple times on the same command-line, thus creating multiple agents. 
More than one agent may use the same jarpath.

**by dynamic loading:** the JVM must implement a mechanism to start agents sometime after the the VM has started. That way, a tool 
can "attach" an agent to a running JVM (for instance profilers or ByteMan)

After the JVM has initialized, the agent class will be loaded by the **system class loader**. If the class loader fails to load the agent, the JVM will abort.

Next, the JVM instantiates an **Instrumentation** interface implementation and given the context, tries to invoke one of the two 
methods that an agent must implement: premain or agentmain.

The signatures of the premain and agentmain method are:

```java
public static void premain(String agentArgs, Instrumentation inst);
public static void premain(String agentArgs);

public static void agentmain(String agentArgs, Instrumentation inst);
public static void agentmain(String agentArgs);
```

The agent needs to implement only one signature per method.  The JVM first attempts to invoke the first signature, and if the agent class does not implement it then the JVM will attempt to invoke the second signature.

## Byte Code Instrumentation

With the premain and agentmain methods, the agent can register a **ClassFileTransformer** instance by providing an implementation of 
this interface in order to transform class files. To register the transformer, the agent can use the **addTransformer** method of the given Instrumentation instance.

Now, all future class definitions will be seen by the transformer, except definitions of classes upon which any registered transformer is dependent. 
The transformer is called when classes are loaded, when they are redefined. and optionally, when they are retransformed (if the transformer was added to 
the instrumentation instance with the boolean canRetransform set to true).

The following method of the ClassFileTransformer interface is responsible of any class file transformation:

```java
byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, 
    ProtectionDomain protectionDomain, byte[] classfileBuffer);
```

## Basic Profiling

To illustrate how the class file transformer can be used, we are going to set up some basic method profiling with **Javassist**:

1. Write a trivial class that will be instrumented
2. Write a ClassFileTransformer to inject some code to print method execution time
3. Write an agent that registers the previous transformer
4. Write the corresponding JUnit test

```java
public class Sleeping {
     public void randomSleep() throws InterruptedException
        // randomly sleeps between 500ms and 1200s
        long randomSleepDuration = (long) (500 + Math.random() * 700);
        System.out.printf("Sleeping for %d ms ..\n", randomSleepDuration);
        Thread.sleep(randomSleepDuration);
```

The transformer class: 这里使用的是 javaassist 而不是 asm

```java
import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.security.ProtectionDomain;
import javassist.ClassPool;
import javassist.CtClass;
import javassist.CtMethod;

// 添加一个函数到 Sleeping 类
public class SleepingClassFileTransformer implements ClassFileTransformer {
 
    public byte[] transform(ClassLoader loader, String className, Class classBeingRedefined,
        ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
 
        byte[] byteCode = classfileBuffer;
 
        if (className.equals("org/javabenchmark/instrumentation/Sleeping")) {
 
            try {
                ClassPool cp = ClassPool.getDefault();
                CtClass cc = cp.get("org.javabenchmark.instrumentation.Sleeping");
                CtMethod m = cc.getDeclaredMethod("randomSleep");
                m.addLocalVariable("elapsedTime", CtClass.longType);
                m.insertBefore("elapsedTime = System.currentTimeMillis();");
                m.insertAfter("{elapsedTime = System.currentTimeMillis() - elapsedTime;"
                        + "System.out.println(\"Method Executed in ms: \" + elapsedTime);}");
                byteCode = cc.toBytecode();
                cc.detach();
            } catch (Exception ex) {
                ex.printStackTrace();
            }
        }
 
        return byteCode;
    }
}
```

The transformer checks if the class to transform is the Sleeping one, then it injects the code that prints the time elapsed by the execution of the randomSleep() method.

```java
public class Agent {
public static void premain(String agentArgs, Instrumentation inst) {
    // registers the transformer
    inst.addTransformer(new SleepingClassFileTransformer());
```

Testing, JUnit code

```java
package org.javabenchmark.instrumentation;
 
import org.junit.Test;
 
public class AgentTest {
    @Test
    public void shouldInstantiateSleepingInstance() throws InterruptedException {
        Sleeping sleeping = new Sleeping();
        sleeping.randomSleep();
```

But, before executing the test you need to setup Maven to enable the JVM agent:

* build the JAR that contains the agent's code before running test
* add the JVM option -javagent to the JVM that runs the test
* add the Javassist dependency

To achieve this, add the following XML code to your pom.xml file, inside the <build><plugins> ... </plugins></build> section: 

```java
<!-- Prepares Agent JAR before test execution -->
<plugin>
    <groupid>org.apache.maven.plugins</groupid>
    <artifactid>maven-jar-plugin</artifactid>
    <version>2.4</version>
    <executions>
        <execution>
            <phase>process-classes</phase>
            <goals>
                <goal>jar</goal>
            </goals>
            <configuration>
                <archive>
                    <manifestentries>
                        <premain-class>org.javabenchmark.instrumentation.Agent</premain-class>
                    </manifestentries>
                </archive>
            </configuration>
        </execution>
    </executions>
</plugin>
 
<!-- executes test with -javaagent option -->
<plugin>
    <groupid>org.apache.maven.plugins</groupid>
    <artifactid>maven-surefire-plugin</artifactid>
    <version>2.14</version>
    <configuration>
        <argline>-javaagent:target/${project.build.finalName}.jar</argline>
    </configuration>
</plugin>

//dep
<dependency>
    <groupid>org.javassist</groupid>
    <artifactid>javassist</artifactid>
    <version>3.14.0-GA</version>
    <type>jar</type>
</dependency>
```

Then, running the test should produce something like this: 

```java
-------------------------------------------------------
 T E S T S
-------------------------------------------------------
Running org.javabenchmark.instrumentation.AgentTest
Sleeping for 818 ms ..
Method Executed in ms: 820
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 1.001 sec
```

You can see that there is a new trace: Method Executed in ms: 820 proving that instrumentation works. Without instrumentation, you would only have the Sleeping for 818 ms .. trace.

It is easy to profile your code with the instrumentation API and the Javassist API: write transformers with Javassist, write an agent to register them, then use the -javaagent option and you're done !

## BTrace 的用法

使用代码的方式, attach agent

```java
public void attach(String pid, String agentPath, String sysCp, String bootCp) throws IOException {
    VirtualMachine vm = VirtualMachine.attach(pid);
    vm.loadAgent(agentPath, agentArgs);
```

agent 的写法

```java
public static void premain(String args, Instrumentation inst) {
        main(args, inst);
}

public static void agentmain(String args, Instrumentation inst) {
        main(args, inst);
}

private static synchronized void main(final String args, final Instrumentation inst) {
        loadArgs(args);
        parseArgs();

        // private static final BTraceTransformer transformer = new BTraceTransformer(debug);
        inst.addTransformer(transformer, true);
        startScripts();

        String tmp = argMap.get("noServer");
        boolean noServer = tmp != null && !"false".equals(tmp);
        if (noServer) {
            if (isDebug()) {
                debugPrint("noServer is true, server not started");
            }
            return;
        }
        Thread agentThread = new Thread(new Runnable() {
            @Override
            public void run() {
                BTraceRuntime.enter();
                try {
                    startServer();
                } finally {
                    BTraceRuntime.leave();
                }
            }
        });
        BTraceRuntime.initUnsafe();
        BTraceRuntime.enter();
        try {
            agentThread.setDaemon(true);
            if (isDebug()) {
                debugPrint("starting agent thread");
            }
            agentThread.start();
        } finally {
            BTraceRuntime.leave();
        }
    }
```

### Client

首先是client.compile方法，使用的是Java compile api，将我们传递的java源文件编译为.class文件，当然你如果使用 btrace 提前编译了源代码，那么这里就不会有这一步

```java
import com.sun.btrace.annotations.*;
import static com.sun.btrace.BTraceUtils.*;
@BTrace
public class HelloWorld {
    @OnMethod(
        clazz="java.lang.Thread",
        method="start"
    )
    public static void func() {
        println("about to start a thread!");
    }
}
```

@OnMethod告诉Btrace解析引擎需要代理的类和方法。 这个例子的作用是当java.lang.Thread类的任意一个对象调用 start 方法后，会调用func方法。

client端在编译完脚本之后，进行了一次字节码修改，但是仅仅是做了一些兼容性，例如域访问控制器、简写等。

接着client.attach中使用java的attach api将agent动态attach到目标jvm进程中(ava agent，通常有两种方式添加到jvm进程中：动态attach；在目标jvm启动之前添加agent参数)。

最后client的submit方法，会向agent发送监控命令以及传递对应code的字节码。

### Agent

jdk5虽然已经有了instrument api，但是其仅仅支持premain方法，也就是仅仅支持在main方法运行之前执行一些动作，而jdk6后加入了agentmain方法和VirtualMachine，
是可以在main方法运行后执行的(如果是通过命令行启动的，那么agentmain方法不会被调用)。此外，在jdk6之前，程序启动之后是无法再设置boot class加载路径和system class加载路径的。
而jdk6之后，instrument新增的appendToBootstrapClassLoaderSearch和appendToSystemClassLoaderSearch是可以动态添加classpath的。

agent被提交到目标jvm进程后，首先会添加boot classpath.

```java
...
inst.appendToBootstrapClassLoaderSearch(jf);
...
inst.appendToSystemClassLoaderSearch(jf);
```

接着开启一个serversocket等待client的连接。之后client和agent之间的数据通讯，比如生成.class发送到agent，agent将线上程序打印的数据回传给 client都是通过socket来进行的。当agent接收到监控命令后，主要有以下两部分工作：

```java
重写类：遍历当前所有的class,根据正则找到匹配的类，用asm重写
替换类：替换掉原来的class
```

agent接受到client发来的监控指令以及对应的参数后，会load所有的class,根据正则去匹配指定的类和方法，
并使用脚本解析引擎去处理发送过来的字节码然后使用ASM将脚本里标注的类java.lang.Thread的字节码重写，
植入跟踪代码或新的逻辑。在上面那个例子中，Java.lang.Thread这个类的字节码被重写并在start方法体尾部植入了func方法的调用。

```java
new ClassFileTransformer() {
    public byte[] transform(ClassLoader l, String className, Class c， ProtectionDomain pd, byte[] b) throws IllegalClassFormatException {
        // BTrace解析脚本，利用asm重写bytecode，然后classLoader加载
    }
}, true);
```

其中，在agent的agentmain中通过handleNewClient方法启动一个异步线程进行class transformer，而在这个异步线程中最终是通过调用
https://github.com/btraceio/btrace/blob/master/src/share/classes/com/sun/btrace/agent/Client.java
中的retransformLoaded()来进行的。

### 总结

其实BTrace就是使用了java attach api附加agent.jar，然后使用脚本解析引擎+asm来重写指定类的字节码，再使用instrument实现对原有类的替换。借鉴这些，我们也完全可以实现自己的动态追踪工具。

