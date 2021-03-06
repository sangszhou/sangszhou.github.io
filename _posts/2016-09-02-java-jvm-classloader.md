---
layout: post
title:  "虚拟机执行子系统"
date:   "2016-09-02 00:00:00"
categories: java
keywords: java, jvm
---

## 类文件结构

Class文件是一组以8位字节为基础单位的二进制流，各数据项严格按顺序排列其中，中间没有添加任何分隔符。
根据Java虚拟机规范的规定，CLASS文件格式采用一种类似C语言结构体的伪结构来存储，
这种伪结构中只有两种数据类型：无符号数和表。

1. 无符号数属于基本的数据类型，以u1,u2,u4,u8来分别表示1字节，2字节，
4字节和8个字节的无符号数，无符号数用来描述数字，索引引用，数量值或按照 UTF8 编码构成字符串数

2. 表是由多个无符号数或其他表作为数据项构成的复合数据类型，所有表都习惯性的以"_info"结尾，
表用于描述有层次关系的复合结构的数据。整个CLASS文件本质上也是一张表


![](/images/posts/javavm/classinfo_pic.png)

魔数 (magic)(4个字节) 0xCAFEBABE, 次版本号(2个字节), 主版本号(2个字节)

常量计数器 (2个字节) 从 1 开始算, 0x0016，十进制为22，代表有21个常量, 索引为 1~21
没有使用0索引是因为在后面某些指向常量池的索引可以通过0索引表示不引用任何一个常量池项目的意思。

### 常量池
    
常量池中主要存放两大类常量：字面量（Literal）和符号引用（Symbolic References）

字面量的例子有文本字符串，被声明为final的常量值等。

符号引用包含三类常量：

* 类和接口的全限定名
* 字段的名称和描述符
* 方法的名称和描述符

JAVA代码在进行JAVAC编译时，并不像C和C++那样有连接这一步骤，而是在虚拟机加载CLASS文件的时候进行动态连接。
也就是说，在CLASS文件中不会保存各个方法和字段的最终内存布局信息，因此这些字段和方法的符号
引用不经过转换的话是无法直接被虚拟机使用的。当虚拟机运行时，需要从常量池获得对应的符号引用，
在类创建时或运行时解析时解析并翻译到具体的内存地址中

常量池中每一项常量都是一个表，共有11种，表开始的第一位都是一个字节的标志位，表明这个常量属于哪种类型

JAVA程序中不能定义超过64KB英文字符的变量和方法名，否则无法编译. 使用JAVAP工具可以分析class文件字节码

```javap -verbose TestClass```

**访问标志**
用于识别一些类或接口的访问信息，包括这个CLASS是类还是接口，是否为public,是否为abstract等

**类索引、父类索引与接口索引集合**
用来确定继承关系

**字段表集合**
用于描述接口或类中声明的变量。字段field包括了类级变量或实例级变量，但不包括在方法内部声明的变量。
不包括方法内部声明的变量.描述了该字段是否是Public，private, protected, static,final,volatile,transient等

描述符用来描述字段的数据类型,方法的参数列表和返回值.用表示标识字符表示,对象类型用L+对象的全限定名表示

**方法表集合**
包括访问标志，名称索引，描述符索引，属性表索引。方法中的代码储存在方法属性表集合中一个名为Code的属性中。

在JAVA虚拟机规范中，要重载一个方法，除了要与原方法具有相同的简单名称外，还要求必须拥有一个与原方法不同的特征签名，特征签名在JAVA代码层面和字节码层面有不同的定义。代码层面的签名只包括方法名称，参数顺序及参数类型，字节码层面还包括方法返回值和异常表。因此在CLASS文件中，如果两个方法有相同的名称和特征签名，但返回值不同，是合法的。

**属性表集合**
属性表集合的限制较少，不要求各个属性表有严格的顺序,并且只要不与已有的属性名重复，
任何人实现的编译器都可以向属性表中写入自己定义的属性信息。JAVA虚拟机在运行时会忽略掉它不认识的属性。
JAVA虚拟机规范中预定义了9项虚拟机实现应当能识别的属性：

Code, 方法名, JAVA代码编译成的字节码指令



### code 属性

JAVA程序方法体中的代码经过Javac编译器处理之后，最终变为字节码指令存储在Code属性中。

max_stack代表操作栈深度的最大值，在方法执行的任何时刻，操作数栈都不会超过这个深度，
虚拟机运行的时候需要根据这个值来分配栈帧中的操作栈深度。

max_locals代表局部变量表所需的存储空间，单位为slot，对于byte,char,float,int,short,
boolean,reference和returnAddress每个局部变量占用一个slot,而double和long需要两个slot.

并不是方法中用到了多少个局部变量，就把这些局部变量所占的Slot之和作为max_locals的值，
原因是局部变量表中的slot可以重用，当代码执行超过一个局部变量的作用域时，这个局部变量所占用
的slot就可以被其他局部变量所使用。这个值编译器会自动计算得出

code_length和code用来存储java源程序编译后生成的字节码指令，code_length代表字节码长度，
code用于存储字节码指令的一系列字节流。字节码的每个指令就是一个字节。这样可以推出，
一个字节最多可以代表256条指令，目前已经使用了约200条。而code_length有4个字节，
所以一个方法做多允许有65535条字节码指令，如果超过这个限制，javac就会拒绝编译，一般JSP可能会这个原因导致失败。

在任何实例方法中，都可以通过this关键字访问到此方法所属的对象，它的底层实现就是通过javac编译器在
编译的时候把this关键字的访问转变为对一个普通方法参数的访问。因此，任何实例方法的参数Args_size最少是1，
而且locals最少也是1.而静态方法就可以为0了。异常表实际是Java代码的一部分，从start_pc行到end_pc行出现了类
型为catch_type的异常，就转到第handler_pc行处理。这四个参数就组成了异常表。对于finally的实现，实际上就是
对catch字段和前面对于任意情况都运行的异常表记录

### java 代码到字节码的例子

```java
public abstract class Animal {

    private int legs = 0;

    public abstract boolean hasLeg();
}
```

javap - c Animal.class

```java
public abstract class play.Animal {
  public play.Animal(); // 这个是构造函数
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: aload_0
       5: iconst_0
       6: putfield      #2                  // Field legs:I  // 初始化
       9: return

  public abstract boolean hasLeg();
}
```

javap -v Animal.class

```java
Constant pool:
   #1 = Methodref          #4.#18         // java/lang/Object."<init>":()V
   #2 = Fieldref           #3.#19         // play/Animal.legs:I
   #3 = Class              #20            // play/Animal
   #4 = Class              #21            // java/lang/Object
   #5 = Utf8               legs
   #6 = Utf8               I
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               LocalVariableTable
  #12 = Utf8               this
  #13 = Utf8               Lplay/Animal;
  #14 = Utf8               hasLeg
  #15 = Utf8               ()Z
  #16 = Utf8               SourceFile
  #17 = Utf8               Animal.java
  #18 = NameAndType        #7:#8          // "<init>":()V
  #19 = NameAndType        #5:#6          // legs:I
  #20 = Utf8               play/Animal
  #21 = Utf8               java/lang/Object
{
  public play.Animal();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: aload_0
         5: iconst_0
         6: putfield      #2                  // Field legs:I
         9: return
      LineNumberTable:
        line 6: 0
        line 8: 4
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      10     0  this   Lplay/Animal;

  public abstract boolean hasLeg();
    descriptor: ()Z
    flags: ACC_PUBLIC, ACC_ABSTRACT
}
```

```java
public class Cat extends Animal {

    static int id;

    @Override
    public boolean hasLeg() {
        return true;
    }

    public void walk() {
        try {
            System.out.println("walk with leg");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

```java
public class play.Cat extends play.Animal {
  static int id;

  public play.Cat();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method play/Animal."<init>":()V
       4: return

  public boolean hasLeg();
    Code:
       0: iconst_1
       1: ireturn

  public void walk();
    Code:
       0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
       3: ldc           #3                  // String walk with leg
       5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
       8: goto          24
      11: astore_1
      12: aload_1
      13: invokevirtual #6                  // Method java/lang/NullPointerException.printStackTrace:()V
      16: goto          24
      19: astore_1
      20: aload_1
      21: invokevirtual #8                  // Method java/lang/Exception.printStackTrace:()V
      24: return
    Exception table:
       from    to  target type
           0     8    11   Class java/lang/NullPointerException
           0     8    19   Class java/lang/Exception
}
```


```java

static int id = 0; // 对应的字节码 

static <clinit>()V
   L0
    LINENUMBER 8 L0
    ICONST_0
    PUTSTATIC overflow/Cat.id : I
    RETURN
    MAXSTACK = 1
    MAXLOCALS = 0

```


## 类加载

Java源代码被编译成class字节码，最终需要加载到虚拟机中才能运行。整个生命周期包括：
加载、验证、准备、解析、初始化、使用和卸载7个阶段。

![](/images/posts/javavm/loadclassprocess.png)

### 加载

> 1. 通过一个类的全限定名获取描述此类的二进制字节流
> 2. 将这个字节流所代表的静态存储结构保存为方法区的运行时数据结构
> 3. 在java堆中生成一个代表这个类的java.lang.Class对象，作为访问方法区的入口

虚拟机设计团队把加载动作放到JVM外部实现，以便让应用程序决定如何获取所需的类，实现这个动作的代码称为“类加载器”，JVM提供了3种类加载器：

**1. 启动类加载器（Bootstrap ClassLoader）** 
负责加载 JAVA_HOME\lib 目录中的，或通过-Xbootclasspath参数指定路径中的，且被虚拟机认可
(按文件名识别，如rt.jar)的类

**2. 扩展类加载器（Extension ClassLoader）**
负责加载 JAVA_HOME\lib\ext 目录中的，或通过java.ext.dirs系统变量指定路径中的类库。

**3. 应用程序类加载器（Application ClassLoader）**
负责加载用户路径（classpath）上的类库。

JVM基于上述类加载器，通过双亲委派模型进行类的加载，当然我们也可以通过继承java.lang.ClassLoader实现自定义的类加载器。

![](/images/posts/javavm/classloaders.png)

**双亲委派模型有什么好处?**
比如位于rt.jar包中的类java.lang.Object，无论哪个加载器加载这个类，最终都是委托给顶层的启动类加载器进行加载，确保了Object类在各种加载器环境中都是同一个类。


### 为什么有三个 class loader

[stackoverfow](http://stackoverflow.com/questions/28011224/what-is-the-reason-for-having-3-class-loaders-in-java)

The reason for having the three basic class loaders (Bootstrap, extension, system) is mostly security.

Prior to version 1.2 of the JVM, there was just one default class loader, which is what is currently called 
the "Bootstrap" class loader.

The way classes are loaded by class loaders is that each class loader first calls its parent, and if that parent 
doesn't find the requested class, the current one is looking for it itself.

A key concept is the fact that the JVM will not grant package access (the access that methods and fields have 
if you didn't specifically mention private, public or protected) unless the class that asks for this access comes 
from the same class loader that loaded the class it wishes to access.

So, suppose a user calls his class java.lang.MyClass. Theoretically, it could get package access to all the fields 
and methods in the java.lang package and change the way they work. The language itself doesn't prevent this. But 
the JVM will block this, because all the real java.lang classes were loaded by bootstrap class loader. Not the same 
loader = no access.

There are other security features built into the class loaders that make it hard to do certain types of hacking.

So why three class loaders? Because they represent three levels of trust. The classes that are most trusted are 
the core API classes. Next are installed extensions, and then classes that appear in the classpath, which means they 
are local to your machine.

2. Then you want to be able todeploy another application in Tomcat. Maybe this second application uses a 
library that the first one also uses, but in a different version. So you want each app to have its own isolated 
class loader, otherwise the classes of app 2 might interfere badly with the classes from app 1.


### 尝试绕过双亲委派

```
双亲委派模型在 jdk1.2 引入，但它不是一个强制性的约束模型，而是 Java 设计者推荐给开发者的一种类加载方式
```
换句话说，也就是可以不用这个模型的，自己实现类加载就可以，毕竟类加载器的原始作用就是：“通过类的全限定名得到类的二进制码流”

[资料](http://blog.csdn.net/scythe666/article/details/51956047) 写的很好


**由不同的类加载器加载的指定类型还是相同的类型吗**

在Java中，一个类用其完全匹配类名(fully qualified class name)作为标识，这里指的完全匹配类名包括包名和类名。
但在JVM中一个类用其全名和一个加载类ClassLoader的实例作为唯一标识，不同类加载器加载的类将被置于不同
的命名空间.我们可以用两个自定义类加载器去加载某自定义类型（注意，不要将自定义类型的字节码放
置到系统路径或者扩展路径中，否则会被系统类加载器或扩展类加载器抢先加载），然后用获取到的两个Class
实例进行java.lang.Object.equals（…）判断，将会得到不相等的结果。这个大家可以写两个自定义的类加载
器去加载相同的自定义类型，然后做个判断；同时，可以测试加载java.*类型，然后再对比测试一下测试结果。

**在代码中直接调用Class.forName（String name）方法，到底会触发那个类加载器进行类加载行为**

Class.forName(String name)默认会使用调用类的类加载器来进行类加载。我们直接来分析一下对应的jdk的代码：

```java
//java.lang.Class.java  
public static Class<?>forName(String className)throws ClassNotFoundException {  
    return forName0(className,true, ClassLoader.getCallerClassLoader());  
}  

//java.lang.ClassLoader.java  
// Returns the invoker's class loader, or null if none.  
static ClassLoader getCallerClassLoader() {  
       // 获取调用类（caller）的类型  
       Class caller = Reflection.getCallerClass(3);  
       // This can be null if the VM is requesting it  
       if (caller ==null) {  
           return null;  
        }  
       //调用java.lang.Class中本地方法获取加载该调用类（caller）的ClassLoader  
       return caller.getClassLoader0();  
}  

//java.lang.Class.java  
//虚拟机本地实现，获取当前类的类加载器，前面介绍的Class的getClassLoader()也使用此方法  
native ClassLoader getClassLoader0();  

//java.lang.Class.java  
public static Class<?>forName(String className) throws ClassNotFoundException {  
    return forName0(className,true, ClassLoader.getCallerClassLoader());  
}  
//java.lang.ClassLoader.java  
// Returns the invoker's class loader, or null if none.  
static ClassLoader getCallerClassLoader() {  
       // 获取调用类（caller）的类型  
       Class caller = Reflection.getCallerClass(3);  
       // This can be null if the VM is requesting it  
       if (caller ==null) {  
           return null;  
       }  
       //调用java.lang.Class中本地方法获取加载该调用类（caller）的ClassLoader  
       return caller.getClassLoader0();  
}  

//java.lang.Class.java  
//虚拟机本地实现，获取当前类的类加载器，前面介绍的Class的getClassLoader()也使用此方法  
native ClassLoader getClassLoader0();  
```

**在编写自定义类加载器时，如果没有设定父加载器，那么父加载器是？**

前面讲过，在不指定父类加载器的情况下，默认采用系统类加载器。可能有人觉得不明白，现在我们来看一下JDK对应的代码实现。众所周知，我们编写自定义的类加载器直接或者间接继承自java.lang.ClassLoader抽象类，对应的无参默认构造函数实现如下：

```java
//摘自java.lang.ClassLoader.java  
protected ClassLoader() {  
          SecurityManager security = System.getSecurityManager();  
          if (security !=null) {  
               security.checkCreateClassLoader();  
          }  
          this.parent = getSystemClassLoader();  
          initialized =true;  
}  
```

我们再来看一下对应的getSystemClassLoader()方法的实现:

```java
privatestaticsynchronizedvoid initSystemClassLoader() {  
           //...  
           sun.misc.Launcher l = sun.misc.Launcher.getLauncher();  
           scl = l.getClassLoader();  
           //...  
}
```

**Java虚拟机的第一个类加载器是Bootstrap，这个加载器很特殊，它不是Java类，因此它不需要被别人加载，它嵌套在Java虚拟机内核里面，也就是JVM启动的时候Bootstrap就已经启动，它是用C++写的二进制代码（不是字节码），它可以去加载别的类。**

这也是我们在测试时为什么发现System.class.getClassLoader()结果为null的原因，这并不表示System这个类没有类加载器，而是它的加载器比较特殊，是BootstrapClassLoader，由于它不是Java类，因此获得它的引用肯定返回null。

**委托机制的意义 — 防止内存中出现多份同样的字节码**

比如两个类A和类B都要加载System类：

如果不用委托而是自己加载自己的，那么类A就会加载一份System字节码，然后类B又会加载一份System字节码，这样内存中就出现了两份System字节码。
如果使用委托机制，会递归的向父类查找，也就是首选用Bootstrap尝试加载，如果找不到再向下。这里的System就能在Bootstrap中找到然后加载，如果此时类B也要加载System，也从Bootstrap开始，此时Bootstrap发现已经加载过了System那么直接返回内存中的System即可而不需要重新加载，这样内存中就只有一份System的字节码了。




### 能不能自己写个类叫 java.lang.System？
    
答案：通常不可以，但可以采取另类方法达到这个需求。 

解释：为了不让我们写System类，类加载采用委托机制，这样可以保证爸爸们优先，爸爸们能找到的类，儿子就没有机会加载。
而System类是Bootstrap加载器加载的，就算自己重写，也总是使用Java系统提供的System，自己写的System类根本没有机会得到加载。

此外, 即便写出来了, JVM 也会报 SecurityException, 因为不让用 java.lang 包名。

### 自定义类加载器

```java
public class LoaderTest {

    public static void main(String[] args) {
        try {
            System.out.println(ClassLoader.getSystemClassLoader());
            System.out.println(ClassLoader.getSystemClassLoader().getParent());
            System.out.println(ClassLoader.getSystemClassLoader().getParent().getParent());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
// result
sun.misc.Launcher$AppClassLoader@6d06d69c  
sun.misc.Launcher$ExtClassLoader@70dea4e  
null  
```


```java
// 文件系统类加载器
public class FileSystemClassLoader extends ClassLoader {
    private String rootDir;
    public FileSystemClassLoader(String rootDir) {
        this.rootDir = rootDir;
    }

    // 获取类的字节码
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] classData = getClassData(name);  // 获取类的字节数组
        if (classData == null) {
            throw new ClassNotFoundException();
        } else {
            return defineClass(name, classData, 0, classData.length); // 系统函数
        }
    }

    private byte[] getClassData(String className) {
        // 读取类文件的字节
        String path = classNameToPath(className);
        try {
            InputStream ins = new FileInputStream(path);
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            int bufferSize = 4096;
            byte[] buffer = new byte[bufferSize];
            int bytesNumRead = 0;
            // 读取类文件的字节码
            while ((bytesNumRead = ins.read(buffer)) != -1) {
                baos.write(buffer, 0, bytesNumRead);
            }
            return baos.toByteArray();
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }

    private String classNameToPath(String className) {
        // 得到类文件的完全路径
        return rootDir + File.separatorChar + className.replace('.', File.separatorChar) + ".class";
    }
}
```

如上所示，类 FileSystemClassLoader继承自类java.lang.ClassLoader。在java.lang.ClassLoader类的常用方法中，
一般来说，自己开发的类加载器只需要覆写 findClass(String name)方法即可。java.lang.ClassLoader类的方法loadClass()封装了
前面提到的代理模式的实现。该方法会首先调用findLoadedClass()方法来检查该类是否已经被加载过；如果没有加载过的话，会调用父类加
载器的loadClass()方法来尝试加载该类；如果父类加载器无法加载该类的话，就调用findClass()方法来查找该类。因此，为了保证类加载器都正
确实现代理模式，在开发自己的类加载器时，最好不要覆写 loadClass()方法，而是覆写 findClass()方法。

类 FileSystemClassLoader的 findClass()方法首先根据类的全名在硬盘上查找类的字节代码文件（.class 文件），然后读取该文
件内容，最后通过defineClass()方法来把这些字节代码转换成 java.lang.Class类的实例。

```java
public class NetworkClassLoader extends ClassLoader {  
    private String rootUrl;  
    public NetworkClassLoader(String rootUrl) {  
        // 指定URL  
        this.rootUrl = rootUrl;  
    }  
    // 获取类的字节码  
    @Override  
    protected Class<?> findClass(String name) throws ClassNotFoundException {  
        byte[] classData = getClassData(name);  
        if (classData == null) {  
            throw new ClassNotFoundException();  
        } else {  
            return defineClass(name, classData, 0, classData.length);  
        }  
    }  

    private byte[] getClassData(String className) {  
        // 从网络上读取的类的字节  
        String path = classNameToPath(className);  
        try {  
            URL url = new URL(path);  
            InputStream ins = url.openStream();  
            ByteArrayOutputStream baos = new ByteArrayOutputStream();  
            int bufferSize = 4096;  
            byte[] buffer = new byte[bufferSize];  
            int bytesNumRead = 0;  
            // 读取类文件的字节  
            while ((bytesNumRead = ins.read(buffer)) != -1) {  
                baos.write(buffer, 0, bytesNumRead);  
            }  
            return baos.toByteArray();  
        } catch (Exception e) {  
            e.printStackTrace();  
        }  
        return null;  
    }  
  
    private String classNameToPath(String className) {  
        // 得到类文件的URL  
        return rootUrl + "/" + className.replace('.', '/') + ".class";  
    }  
}  
```

在通过NetworkClassLoader加载了某个版本的类之后，一般有两种做法来使用它。第一种做法是使用Java反射API。另外一种做法是使用接口。需要注意的是，并不能直接在客户端代码中引用从服务器上下载的类，因为客户端代码的类加载器找不到这些类。使用Java反射API可以直接调用Java类的方法。而使用接口的做法则是把接口的类放在客户端中，从服务器上加载实现此接口的不同版本的类。在客户端通过相同的接口来使用这些实现类。我们使用接口的方式。示例如下：


### 如何在运行时判断系统类加载器能加载哪些路径下的类

1.  可以直接调用ClassLoader.getSystemClassLoader()或者其他方式获取到系统类加载器（系统类加载器和扩展类加载器本身都派生自URLClassLoader），调用URLClassLoader中的getURLs()方法可以获取到。
2.  可以直接通过获取系统属性java.class.path来查看当前类路径上的条目信息 ：System.getProperty("java.class.path")。

### 编写自定义类加载器时，一般有哪些注意点

**一般尽量不要覆写已有的loadClass(...)方法中的委派逻辑**

一般在JDK 1.2之前的版本才这样做，而且事实证明，这样做极有可能引起系统默认的类加载器不能正常工作。在JVM规范和JDK文档中（1.2或者以后版本中），都没有建议用户覆写loadClass(…)方法，相比而言，明确提示开发者在开发自定义的类加载器时覆写findClass(…)逻辑。举一个例子来验证该问题：

```java
//用户自定义类加载器WrongClassLoader.Java（覆写loadClass逻辑）  
public class WrongClassLoader extends ClassLoader {  
  
    public Class<?> loadClass(String name) throws ClassNotFoundException {  
        return this.findClass(name);  
    }  
  
    protected Class<?> findClass(String name) throws ClassNotFoundException {  
        // 假设此处只是到工程以外的特定目录D:\library下去加载类  
        // 具体实现代码省略  
    }  
}  

WrongClassLoader loader = new WrongClassLoader();  
Class classLoaded = loader.loadClass("beans.Account");  

// 虽然找得到 Acount, 但是无法加载 Account 的父类 Object
java.io.FileNotFoundException: D:"classes"java"lang"Object.class (系统找不到指定的路径。)  

```

### 在编写自定义类加载器时，如果没有设定父加载器，那么父加载器是谁

前面讲过，在不指定父类加载器的情况下，默认采用系统类加载器。可能有人觉得不明白，现在我们来看一下JDK对应的代码实现。众所周知，我们编写自定义的类加载器直接或者间接继承自java.lang.ClassLoader抽象类，对应的无参默认构造函数实现如下：

```java
System.out.println(sun.misc.Launcher.getLauncher().getClassLoader());
// result
sun.misc.Launcher$AppClassLoader@73d16e93  
```

所以，我们现在可以相信当自定义类加载器没有指定父类加载器的情况下，默认的父类加载器即为系统类加载器。同时，我们可以得出如下结论：即使用户自定义类加载器不指定父类加载器，那么，同样可以加载如下三个地方的类：

1. <Java_Runtime_Home>/lib下的类；

2. < Java_Runtime_Home >/lib/ext下或者由系统变量java.ext.dir指定位置中的类；

3. 当前工程类路径下或者由系统变量java.class.path指定位置中的类。

[详细介绍类加载器](http://blog.csdn.net/zhoudaxia/article/details/35824249)
资料 [写的不错](http://blog.csdn.net/zshake/article/details/49491619)

### 验证

为了确保Class文件符合当前虚拟机要求，需要对其字节流数据进行验证，主要包括格式验证、元数据验证、字节码验证和符号引用验证。

**1. 格式验证:**

>验证字节流是否符合class文件格式的规范，并且能被当前虚拟机处理，
>如是否以魔数0xCAFEBABE开头、主次版本号是否在当前虚拟机处理范围内、常量池是否有不支持的常量类型等。
>只有经过格式验证的字节流，才会存储到方法区的数据结构，剩余3个验证都基于方法区的数据进行。

**2. 元数据验证:**

>  对字节码描述的数据进行语义分析，以保证符合Java语言规范，如是否继承了final修饰的类、
>是否实现了父类的抽象方法、是否覆盖了父类的final方法或final字段等。

**3. 字节码验证:** 

> 对类的方法体进行分析，确保在方法运行时不会有危害虚拟机的事件发生，
>如保证操作数栈的数据类型和指令代码序列的匹配、保证跳转指令的正确性、保证类型转换的有效性等。

**4. 符号引用验证:**

>  为了确保后续的解析动作能够正常执行，对符号引用进行验证，
>如通过字符串描述的全限定名是都能找到对应的类、在指定类中是否存在符合方法的字段描述符等。

### 准备

在准备阶段，为类变量（static修饰）在方法区中分配内存并设置初始值。

```java
private static int var = 100;
```

准备阶段完成后，var 值为0，而不是100。在初始化阶段，才会把100赋值给val，但是有个特殊情况：

```java
private static final int VAL= 100;
```

在编译阶段会为VAL生成ConstantValue属性，在准备阶段虚拟机会根据ConstantValue属性将VAL赋值为100。

### 解析

解析阶段是将常量池中的符号引用替换为直接引用的过程，符号引用和直接引用有什么不同？

1. 符号引用使用一组符号来描述所引用的目标，可以是任何形式的字面常量，定义在Class文件格式中
2. 直接引用可以是直接指向目标的指针、相对偏移量或则能间接定位到目标的句柄


### 初始化

初始化阶段是执行类构造器<clinit>方法的过程，<clinit>方法由类变量的赋值动作和静态语句块按照在源文件出现的顺序合并而成，
该合并操作由编译器完成

```java
    private static int value = 100;
    static int a = 100;
    static int b = 100;
    static int c;

    static {
        c = a + b;
        System.out.println("it only run once");
    }
```

1. <clinit>方法对于类或接口不是必须的，如果一个类中没有静态代码块，也没有静态变量的赋值操作，那么编译器不会生成<clinit>
2. <clinit>方法与实例构造器不同，不需要显式的调用父类的<clinit>方法，虚拟机会保证父类的<clinit>优先执行
3. 为了防止多次执行<clinit>，虚拟机会确保<clinit>方法在多线程环境下被正确的加锁同步执行，如果有多个线程同时初始化一个类，那么只有一个线程能够执行<clinit>方法，其它线程进行阻塞等待，直到<clinit>执行完成
4. 注意：执行接口的<clinit>方法不需要先执行父接口的<clinit>，只有使用父接口中定义的变量时，才会执行

### 类初始化场景
虚拟机中严格规定了有且只有5种情况必须对类进行初始化。

* 执行new、getstatic、putstatic和invokestatic指令
* 使用reflect对类进行反射调用
* 初始化一个类的时候，父类还没有初始化，会事先初始化父类
* 启动虚拟机时，需要初始化包含main方法的类
* 在JDK1.7中，如果java.lang.invoke.MethodHandler实例最后的解析结果REF_getStatic、REF_putStatic、REF_invokeStatic的方法句柄，并且这个方法句柄对应的类没有进行初始化

**以下几种情况，不会触发类初始化**

1. 通过子类引用父类的静态字段，只会触发父类的初始化，而不会触发子类的初始化。

```java
class Parent {
    static int a = 100;
    static {
        System.out.println("parent init！");
    }
}

class Child extends Parent {
    static {
        System.out.println("child init！");
    }
}

public class Init{  
    public static void main(String[] args){  
        System.out.println(Child.a); // parent init  
    }  
}
```

2. 定义对象数组，不会触发该类的初始化

```java
public class Init{  
    public static void main(String[] args){  
        Parent[] parents = new Parent[10];
    }  
}
```

3. 常量在编译期间会存入调用类的常量池中，本质上并没有直接引用定义常量的类，不会触发定义常量所在的类

```java
class Const {
    static final int A = 100;
    static {
        System.out.println("Const init");
    }
}

public class Init{  
    public static void main(String[] args){  
        System.out.println(Const.A);  
    }  
}
```
无输出，说明没有触发类Const的初始化，在编译阶段，Const类中常量A的值100存储到Init类的常量池中，这两个类在编译成class文件之后就没有联系了           

4. 通过类名获取Class对象，不会触发类的初始化

```java
public class test {
   public static void main(String[] args) throws ClassNotFoundException {
        Class c_dog = Dog.class;
        Class clazz = Class.forName("zzzzzz.Cat");
    }
}

class Cat {
    private String name;
    private int age;
    static {
        System.out.println("Cat is load");
    }
}

class Dog {
    private String name;
    private int age;
    static {
        System.out.println("Dog is load");
    }
}
```



## 虚拟机字节码执行引擎

执行引擎在执行JAVA代码的时候可以选择解释执行（通过解释器执行）和编译执行（通过即使编译器产生本地代码执行）两种选择。

### 运行时栈帧结构
    
栈帧(StackFrame)是用于支持虚拟机进行方法调用和方法执行的数据结构，它是虚拟机运行时数据区中的虚拟机栈(VirtualMachine Stack)的栈元素。栈帧存储了方法的局部变量表，操作数栈，动态连接和方法返回地址等信息。每一个方法调用的过程，就对应着一个栈帧在虚拟机栈中入栈到出栈的过程。

一个线程中的方法调用链可能很长，很多方法都同时处于执行状态，对于执行引擎来说，活动线程中，只有栈顶的栈帧是有效的，成为Curent Stack Frame。 这个栈帧所关联的方法称为当前方法(Current Method)。执行引擎所运行的所有字节码指令都只针对当前栈帧进行操作。

![](/images/posts/javavm/stackframe.png)

**局部变量表**

用于存放方法参数和方法内部定义的局部变量，在编译成CLASS文件时，就在方法的CODE属性的max_locals数据项中确定了该方法所需要分配的最大局部变量表的容量。一个Slot可以存放一个32位以内的数据类型，这些类型有boolean,byte,char,short,

int,float,reference和returnAddress。returnAddress是为字节码指令jsr,jsr_w和ret服务的，指向下一条字节码的地址。对于64位数据，JVM会以高位在前的方式分配两个连续的Slot空间。

JVM通过索引定位的方式使用局部变量表，索引值的范围从0开始到局部变量表最大的SLOT数量。在方法执行时，JVM使用局部变量表完成参数值到参数变量列表的传递过程。如果是实例方法，那么局部变量表的第0位索引的SLOT默认是用于传递方法所属对象实例的引用，在方法中可以通过“this"访问到这个隐含的参数。其余参数则按照参数表的顺序排列，参数表分配完毕后，再根据方法体内部定义的变量顺序和作用于分配其余SLOT。

局部变量表中的SLOT是可以重用的，如果当前字节码PC计数器的值已经超出了某个变量的作用域，那么这个变量对应的SLOT就可以交给其他变量使用。

**操作数栈**

LIFO。操作数栈的最大深度在编译时写入到Code属性的max_stacks数据项中。

操作数栈的每一个元素可以是任意JAVA数据类型，32位数据占栈容量为1，64位占栈容量为2.

当一个方法开始执行时，这个方法的操作数栈是空的，在方法执行过程中，会有各种字节码指令向操作数栈中写入和提取内容。比如，加法的字节码指令iadd在运行时会将栈顶两个元素相加并出战，再将结果入栈。在编译器和校验阶段的保证下，操作数栈中元素的数据类型必须与字节码指令的序列严格匹配。

**动态连接**

每个栈帧都包含一个指向运行时常量池中该栈帧所属方法的引用，持有这个引用是为了支持方法调用过程中的动态连接。
字节码中的方法调用指令就以常量池中指向方法的符号引用为参数。这些符号引用一部分会在类加载阶段或第一次使用
的时候转换为直接引用，成为静态解析，另一部分会在每一次运行时转换为直接引用，成为动态连接。

**方法返回地址**

有两种方式退出当前执行的方法，一是执行引擎遇到任意一个方法返回的字节码指令，这种方法称为正常完成出口。二是在方法执行过程中遇到无法处理的异常，这种方法称为异常完成出口。

无论哪种方法，方法退出后，都需要返回到调用者的位置，正常退出时，调用者的PC计数器值可以作为返回地址，栈帧中很可能会保存这个计数器值，而异常退出时，返回地址要通过异常处理器表来确定。

方法退出的过程实际上是将当前栈帧出战，并恢复上层方法的局部变量表和操作数栈，把返回值压入调用者的操作数栈中。

**方法调用**
方法调用不等于方法执行，其唯一的任务就是确定调用哪一个具体方法。

**解析**

所有方法调用中的目标方法在Class文件里面都是一个常量池中的符号引用，而不是方法在实际运行时内存布局中的入口地址。

在类加载的解析阶段，一部分符号引用会被转化为直接引用，这种解析成立的前提是：方法在程序真正运行
之前就有一个可确定的调用版本，且这个方法的调用版本在运行时是不可改变的（“编译期可知，运行期不可变”）。
符合这个条件的有静态方法和私有方法两大类。JVM提供了4条方法调用的字节码指令：

> 1.   invokestatic:调用静态方法
> 2.   invokespecial:调用实例构造器<init>方法，私有方法和父类方法
> 3.   invokevirtual:调用所有的虚方法
> 4.   invokeinterface:调用接口方法，会在运行时再确定一个实现此接口的对象。

只要能被 invokestatic和invokespecial调用的方法，都可以在解析阶段进行转化。

除此以外（除静态方法，实例构造器，私有方法，父类方法以外）其他方法称为虚方法。

JAVA非虚方法除了invokestatic和invokespecial以外，还有一种就是final修饰的方法，因为该方法无法被覆盖，
这种被final修饰的方法是用invokevirtual指令调用的

### 分派


**静态分派**

```java
Parent father =new Son();
```

Parent被称为静态类型，Son称为实际类型。

虚拟机（编译器）重载时通过参数的静态类型作为判断依据。所有依赖静态类型来定位**方法执行版本**的分派动作，都称为静态分派。
静态分派的最典型应用就是**方法重载**。

在重载的情况下，很多时候重载版本并不是唯一的，而是寻找一个最合适的版本。比如存在多个重载方法的情况中，调用的目标顺序是一定的。

**动态分派**

它与重写（override）有着密切的关系。

相对于前面的重载中，引用类型对于具体调用哪个方法起决定性作用，在重写中，引用指向的对象的具体类型决定了调用的具体目标方法。

invokevirtual 指令的多态查找

1. 找到操作数栈顶的第一个元素指向的实际类型, 记做 C
2. 如果类型 C 中找到与常量中描述符和简单名称都相符的方法, 则进行访问权限校验
3. 否则, 按照继承关系从下往上一次对 C 的各个父类进行第二步的搜索和验证过程
4. 如果还找不到合适的方法, 则抛出 AbstractMethodError

invokevirtual 指令把常量池中的类方法符号引用解析成直接引用, 不同的实际类型解析成不同的直接引用, 这就是重写的本质

这种在运行时期通过实际类型确认方法执行版本的分派过程称为动态分派

由于动态分派是非常频繁的操作，因此在JVM具体实现中基于性能考虑，常常 做一些优化，最初那个用
的“稳定优化”手段就是为类在方法去中建立一个虚方法表（vtable），于此对应，invokeinterface 执行时也会用到接口方发表，itable。

虚方法表中存放着各个方法的实际入口地址，如果某个方法在子类中**没有被重写，那么子类的虚方法表里的地址入口和父类相同方法的地址入口是一致的**。

如果子类重写了这个方法，子类方法表中的地址就会被替换为指向子类实现版本的入口地址。
为了程序实现上的方便，具有相同签名的方法，在父类、子类的虚方发表中都应当具有一样的索引序号，这样当类型变换时，
仅需要变更要查找的方法表即可。方法表一般在类加载的**链接**阶段进行初始化，准备了类的变量初始值后，虚拟机会把该类的方法表也初始化完毕。


### **单分派和多分派**

**方法的接收者**与**方法的参数**统称为方法的宗量。根据分派基于多少种宗量，可以将分派划分为单分派和多分派两种：

单分派是根据一个宗量对目标方法进行选择，多分派是根据多于一个宗量对目标方法进行选择。

首先进行静态分派，生成相应的字节码，在常量池中生成对应的方法符号引用，这个过程根据了两个宗量进行选择（接收者和参数），
因此静态分派是多分派类型。再进行动态分派，将符号引用变成直接引用时，**只对方法的接收者进行选择**，因此只有一个宗量，
动态分派是单分派。


### **基于栈的字节码解释执行引擎**

Java编译器输出的指令流，基本上是一种基于栈的指令集架构，与之对应的是寄存器指令集架构。基于栈的指令集主
要的优点是可移植性。而寄存器由硬件直接提供，程序直接依赖这些硬件寄存器则不可避免地受到硬件的约束。

```java
public int cal()
    int a = 100
    int b = 200
    int c = 300
    return (a+b)+c
```

对应的字节码表示

locals 表示局部变量表的长度, a, b, c 加上 this, 应该是 4

stack 表示栈的深度, a+b 这个运算需要两个栈深度, 所以最多是 2

参数只有 this, 所以参数个数为 1

**字节码是**

```
// 开始时刻, 局部变量表的第 0 位是 this 指针

bipush 100 // 100 -> 操作数栈顶

istore_1 // 把操作数栈顶的元素放到局部变量表的第一个 slot 中

bipush 200
istore_2
bipush 300
istore_3

iload1 // 把局部变量表的第一个元素放到操作数栈顶
iload2 // 第二个元素放到操作数栈顶
iadd // 把栈顶的元素出栈, 做整型加法, 然后再入栈
iload3 // 局部变量表第三个元素入栈
imul // 
ireturn // 操作数栈顶元素返回给此方法的调用者
```

















