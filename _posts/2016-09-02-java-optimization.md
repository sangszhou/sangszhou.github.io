---
layout: post
title:  "JVM 编译器和运行期优化"
date:   "2016-09-03 00:00:00"
categories: java
keywords: java, optimization
---

## 编译器优化

编译过程大致可以分成3个过程:

> 解析与填充符号表过程

> 插入式注释处理器的注解过程处理

> 分析与字节码生成过程

### 解析与填充符号表

解析步骤包括词法分析和语法分析两个过程

**词法、语法分析**

**词法分析**是将源代码的字节流变成标记（Token）集合，单个字符是程序编码过程的最小元素，而标记则是编译过程的最小元素。

在Javac的源代码中，词法分析过程由 `com.sun.tools.javac.parser.Scanner` 类来实现

**词法分析**是根据Token序列构造抽象语法树的过程，抽象语法是一种用来描述程序代码语法结构的树形表示方式，
语法树的每一个节点都代表着程序代码中的一个语法结构

语法分析过程由 `com.sun.tools.javac.parser.Parse` 类来实现，这个阶段产生出抽象语法树有
`com.sun.tools.javac.tree.JTree` 类表示，经过这个步骤之后，编译器就基本不会再对源代码文件进行操作了, 后续的操作都建立在抽象语法树上

**填充符号表**

符号表（Symbol Table）是由一组符号地址和符号信息构成的表格

在语法分析中，符号表所登记的内容将用于语法分析检查和产生中间代码

在目标代码生成阶段，当对符号名进行地址分配时，符号表是地址分配的依据

在Javac源代码中，填充符号表的过程由 `com.sun.tools.javac.compiler.Enter` 类来实现，
此过程的出口是一个待处理列表（ToDoList），包含每一个编译单元的抽象语法树的顶级节点以及 package-info-java 的顶级节点

### 注解处理器

在Javac源码中，插入式注解处理器的初始化过程是在 initProcessAnnotations（）方法中完成的，而它的执行过程则
是在ProcessAnnotations（）方法中完成的。这个方法判断是否有新的注解处理器需要执行，如果有的话，
通过com.sun.tools.javac.Processing.JavacProvcessingEnviroment类的doProcessing（）方法生成一
个新的JavaCompiler对象对编译的后续步骤进行处理。

### 语义分析与字节码生成

**（1）标注检查:**

内容包括诸如变量使用前后是否已被声明，变量与赋值之间的数据类型是否能够匹配等。在标注检查步骤中，还有一个重要的动作，称为常量折叠。

标注检查步骤在javac源代码中实现类是com.sun.tools.javacComp.Attr类和com.sun.tools.javac.comp.Check类。

**（2）数据及控制分析**

对程序上下文逻辑更进一步的验证，它可以检查出诸如程序局部变量在使用前后是否有赋值，方法的每一条路径是否都有返回值，是否所有的受检查异常都被正确出来了等问题。

在Javac的源代码中，数据及控制流分析的入口是flow（）方法，具体操作是由com.sun.tools.javac.comp.Flow类来完成的。

**（3）解语法糖**

语法糖（Syntatic Sugar），也称糖衣语法，指在计算机语言中添加的某种语法，这种语法对语言的功能并没有影响，但是更方便使用。

在Javac的源代码中，解语法糖的过程由的desugar（）方法触发，在com.dun.tools.javac.comp.TransTypes类和com.sun.tools.javac.comp.Lower类中完成。

**（4）字节码生成**

字节码生成是Javac编译过程的最后一个阶段，在Javac源代码里面有 `com.sun.tolls.javac.jvm.Gen` 类来完成。

完成了语法树的遍历和调整之后，就会把填充了所有需要信息的符号表交给 `com.sun.tolls.javac.jvm.ClassWrite` 类，
由这个类的WiteClass（）方法输出字节码，生成最终的class文件。到此为止整个编译过程就结束了 


### Java 语法糖

**泛型与类型擦除**
C#里面泛型无论在程序源码中，编译后的IL中，货值运行期的CLR中，都是切实存在的，List<int>与List<string>就两个不同稍微类型，
它们在系统运行期生成，有自己的虚方法表和，类型数据，这种实现机制称为类型膨胀，基于这种方法实现的泛型称为真是泛型。

Java语言中的泛型不一样，**它只在程序源代码中存在**，在编译后的字节码文件中，就已经替换为原来的原生类型了，
并且在相应的地方**插入了强制转型代码**，因此，对于运行期的Java语言来说，ArrayList<String>与ArrayList<int>就是同一个类，
所以泛型技术实际上是Java语言的一颗语法糖，java语言中的泛型实现方法称为类型擦除，基于这种方法的泛型称为伪泛型。

虚拟机规范中引入了诸如Signature，LocalVariableType等新的属性用于解决伴随泛型而来的参数类型识别问题。

**自动装箱、拆箱与遍历循环**

自动装箱、拆箱在编译之后转化成了对应的包装盒还原方法，而遍历循环则把代码还原成迭代器的实现

包装类的“==”运算在不遇到算术符运算的情况下不会自动拆箱，以及它们 equals() 方法不处理数据转型的关系

**条件编译**

Java语言可以进行条件编译，方法就是使用条件为常量的if语句

Java语言中条件编译的实现是，Java语言的一颗语法糖，根据布尔常量的真假，编译器将会把分支中
不成立的代码清除掉。这一项工作将在编译器解除语法糖阶段（com.sun.tools.javac.comp.Lower类中）实现


## 运行期优化

**JIT**
为了提高热点代码的执行效率，在运行时，虚拟机将会把这些代码编译成与本地平台相关的机器码，
并进行各种层次的优化，完成这个任务的编译器称为即时编译器 (Just In Time,JIT编译器)

### Hotspot虚拟机内的即时编译器

**解释器与编译器**
主流的商用虚拟机，如Hotspot，J9等，都同时包含解释器和编译器

解释器与编译器两者各有优势：当程序需要快速启动和执行的时候，解释器可以首先发挥作用，省去编译的时间，
立即执行。在程序运行后，随着时间得推移，编译器组件发挥作用，把越来越多的代码编译成本地代码，可以获取更高的执行效率。

Hotspot虚拟机中内置了两个编译器，分别称为Client Compiler 和 Server Compiler，简称C1编译器和C2编译器，也叫Opto编译器。

目前主流的Hotspot虚拟机中，默认采用解释器与其中一个编译器直接配合的方式工作。用户可以只用“-client”或“-server”参
数去强制指定虚拟机运行在Client模式或Server模式。

可以使用参数“-Xint”强制虚拟机运行于“解释模式”
可以使用参数“-Xcomp”强制虚拟机运行于“编译模式”

**分层编译：**
a) 第0层：程序解释执行，解释器不启动性能监控功能，可以触发第1层 编译
b) 第1层：也称为C1编译，将字节码编译为本地代码，进行简单，可靠的优化，如有必要将加入性能监控的逻辑
c) 第2层（或2层以上）：也称为C2编译，也是将字节码编译成本地代码，但是会启用一些编译耗时较低的优化，甚至会根据性能监控信息进行一些不可靠的激进优化

**编译对象与触发条件**

热点探测判定方式：基于**采样**的热点探测和**计数**的热点探测

Hotspot虚拟机采用基于计数器的热点探测方法，它为每个方法准备了两类计数器：方法调用计数器和回收计数器


在字节码中遇到控制流跳转指令就称为回边, 一般来讲, 它是循环体代码执行的次数。


### 编译过程

Client Compiler是一个简单快速地三段编译器，主要的关注点在于**局部性的优化**，而放弃了许多耗时较长的局部优化手段。

第一阶段，一个平台独立的前端将字节构造成一种高级中间代码表示（HIR）

第二阶段，一个平台相关的后端从HIR中产生低级中间代码表示（LIR）

最后阶段，在平台相关的后端使用线性扫描算法在LIR上分配寄存器，并在LIR上做窥孔优化，然后产生机器代码

Server Compiler是专门面向服务端的典型用用并为服务端的性能配置特别调整过的编译器，也是一个充分优化过的高级编译器，它会执行所有经典的优化动作。 

### 优化技术
编译器策略：延迟编译，分层编译，**栈上替换**，延迟优化，程序依赖图表示，静态单赋值表示

基于性能监控的优化技术：乐观空值断言，乐观类型断言，乐观类型增强，乐观数组增强，裁剪未被选择的分支，乐观的多态内联。分支频率预测，调用频率预测

基于证据的优化技术：精确性推断，内存值推断，内存值跟踪，**常量折叠**，重组，操作符退化，空值检查消除。类型检测退化，类型检测消除，代数化简，公共子表达式消除

数据流敏感重写：条件常量传播，基于六承载的类型缩减转换，无用代码消除

语言相关的优化技术：类型继承关系分析，去虚拟机化，符号常量传播，**自动装箱**，**消除逃逸分析**，**锁消除**，锁膨胀，消除反射

内存及代码位置交换：表达式提升，表达式下沉，冗余存储消除，相邻存储合并，交汇点分离

循环变换：**循环展开**，循环剥离，安全点消除，迭代分离，范围检查消除

局部代码调整：**内联**，全局代码提升，基于热度的代码分离，Switch调整

控制流图变换：本地代码编排，本独代码封包，延迟槽填充，着色图寄存器分配，线性扫描寄存器分配，复写聚合，常量分裂，复写移除，地址模式匹配。指令窥孔优化，基于确定有限状态机的代码生成

**内联**
一个是去除方法调用的成本(如建立栈帧), 而是为了其他优化建立良好的基础。方法内膨胀以后便于在更大范围内进行后续的优化手段

### hotspot 逃逸技术

逃逸分析并不是直接的优化手段，而是一个代码分析，通过动态分析对象的作用域，为其它优化手段如栈上分配、标量替换和同步消除等提供依据，
发生逃逸行为的情况有两种：方法逃逸和线程逃逸

> 方法逃逸：当一个对象在方法中定义之后，作为参数传递到其它方法中

> 线程逃逸：如类变量或实例变量，可能被其它线程访问到

如果不存在逃逸行为，则可以对该对象进行如下优化：同步消除、标量替换和栈上分配

### 同步消除

线程同步本身比较耗，如果确定一个对象不会逃逸出线程，无法被其它线程访问到，那该对象的读写就不会存在竞争，则可以消除对该对象的同步锁，
通过-XX:+EliminateLocks可以开启同步消除

### 标量替换

1. 标量是指不可分割的量，如 java 中基本数据类型和 reference 类型，相对的一个数据可以继续分解，称为聚合量

2. 如果把一个对象拆散，将其成员变量恢复到基本类型来访问就叫做标量替换；

3. 如果逃逸分析发现一个对象不会被外部访问，并且该对象可以被拆散，那么经过优化之后，并不直接生成该对象，而是在栈上创建若干个成员变量；

通过-XX:+EliminateAllocations可以开启标量替换， -XX:+PrintEliminateAllocations查看标量替换情况

### 栈上分配

故名思议就是在栈上分配对象，其实目前 Hotspot 并没有实现真正意义上的栈上分配，实际上是标量替换

```java
private static int fn(int age) {
    User user = new User(age);
    int i = user.getAge();
    return i;
}
```
User对象的作用域局限在方法fn中，可以使用标量替换的优化手段在栈上分配对象的成员变量，这样就不会生成User对象，
大大减轻GC的压力，下面通过例子看看逃逸分析的影响

```java
public class JVM {
    public static void main(String[] args) throws Exception {
        int sum = 0;
        int count = 1000000;
        //warm up
        for (int i = 0; i < count ; i++)
            sum += fn(i);
        Thread.sleep(500);
        for (int i = 0; i < count ; i++)
            sum += fn(i);

        System.out.println(sum);
        System.in.read();

    private static int fn(int age)
        User user = new User(age);
        int i = user.getAge();
        return i;
}

class User {
    private final int age;
    public User(int age)
        this.age = age;

    public int getAge()
        return age;
}
```

### 自动装箱, 拆箱和遍历

java 语言中使用最多的语法糖

```java
List<Integer> list = Arrays.asList(1, 2, 3, 4)
int sum = 0
for(int i: list)
    sum += i
```

```
List list = Arrays.asList(new Integer[] { Integer.valueOf(1), ...})
int sum = 0
for(Iterator localIterator = list.iterator; localIterator.hasNext();)
    int i = localIterator.next.intValue
    sum =+ i
```

## 参考

[With Example](http://blog.csdn.net/pacosonswjtu/article/details/51045611)



