---
layout: post
title: Design pattern
categories: [design pattern]
keywords: design pattern
---

## UML 类图

![](/images/posts/designpattern/UML_class.jpg)

1. 车的类图结构为<<abstract>>，表示车是一个抽象类
2. 它有两个继承类：小汽车和自行车；它们之间的关系为**实现关系**，使用带**空心箭头**的**虚线**表示
3. 小汽车为与SUV之间也是继承关系，它们之间的关系为**泛化关系**，使用带空心箭头的**实线**表示
4. 小汽车与发动机之间是**组合关系**，使用带**实心方块**的**实线**表示
5. 学生与班级之间是**聚合关系**，使用带**空心方块的实线**表示
6. 学生与身份证之间为**关联关系**，使用**一根实线**表示
7. 学生上学需要用到自行车，与自行车是一种**依赖关系**，使用**带箭头的虚线**表示

### 类之间的关系

关联, 依赖, 聚合 都看不懂

**泛化关系(generalization)**

类的继承结构表现在UML中为：泛化(generalize)与实现(realize)

泛化关系用一条带空心箭头的直接表示；如下图表示 (A继承自B)

实现关系用一条带空心箭头的虚线表示

**聚合与组合关系**

比如A类中包含B类的一个引用b，当A类的一个对象消亡时，b这个引用所指向的对象也同时消亡（没有任何一个引用指向它，成了垃圾对象），这种情况叫做组合，
反之b所指向的对象还会有另外的引用指向它，这种情况叫聚合。所以聚合的关系比组合的关系更弱一些

**聚合关系(aggregation)**

聚合关系用一条带空心菱形箭头的直线表示，如下图表示A聚合到B上，或者说B由A组成；

聚合关系用于表示实体对象之间的关系，表示整体由部分构成的语义；例如一个部门由多个员工组成；

与组合关系不同的是，整体和部分不是强依赖的，即使整体不存在了，部分仍然存在；例如， 部门撤销了，人员不会消失，他们依然存在；

**组合关系(composition)**

与聚合关系一样，组合关系同样表示整体由部分构成的语义；比如公司由多个部门组成；

但组合关系是一种强依赖的特殊聚合关系，如果整体不存在了，则部分也不存在了；例如， 公司不存在了，部门也将不存在了；

**关联关系(association)**

关联关系是用一条直线表示的；它描述不同类的对象之间的结构关系；它是一种静态关系， 通常与运行状态无关，一般由常识等因素决定的；它一般用来定义对象之间静态的、天然的结构； 所以，关联关系是一种“强关联”的关系；

比如，乘车人和车票之间就是一种关联关系；学生和学校就是一种关联关系；

关联关系默认不强调方向，表示对象间相互知道；如果特别强调方向，如下图，表示A知道B，但 B不知道A；

注：在最终代码中，关联对象通常是以成员变量的形式实现的；

**依赖关系(dependency)**

依赖关系是用一套带箭头的虚线表示的；如下图表示A依赖于B；他描述一个对象在运行期间会用到另一个对象的关系

与关联关系不同的是，它是一种临时性的关系，通常在运行期间产生，并且随着运行时的变化； 依赖关系也可能发生变化

显然，依赖也有方向，双向依赖是一种非常糟糕的结构，我们总是应该保持单向依赖，杜绝双向依赖的产生

注：在最终代码中，依赖关系体现为类构造方法及类方法的传入参数，箭头的指向为调用关系；依赖关系处理临时知道对方外，还是“使用”对方的方法和属性

## 软件设计模式的分类

**创建型**

创建对象时，不再由我们直接实例化对象；而是根据特定场景，由程序来确定创建对象的方式，从而保证更大的性能、更好的架构优势。
创建型模式主要有简单工厂模式（并不是23种设计模式之一）、工厂方法、抽象工厂模式、单例模式、生成器(Builder)模式和原型模式

**结构型**

用于帮助将多个对象组织成更大的结构。结构型模式主要有适配器模式 adapter、桥接模式 bridge、组合器模式 component、
装饰器模式 decorator、门面模式、亨元模式 flyweight 和代理模式 proxy

**行为型**

用于帮助系统间各对象的通信，以及如何控制复杂系统中流程。行为型模式主要有命令模式command、解释器模式、迭代器模式、中介者模式、
备忘录模式、观察者模式、状态模式state、策略模式、模板模式和访问者模式


## 单例模式

有些时候，允许自由创建某个类的实例没有意义，还可能造成系统性能下降。如果一个类始终只能创建一个实例，则这个类被称为单例类，
这种模式就被称为单例模式。

一般建议单例模式的方法命名为：getInstance()，这个方法的返回类型肯定是单例类的类型了。getInstance方法可以有参数，
这些参数可能是创建类实例所需要的参数，当然，大多数情况下是不需要的

```java
public class Singleton {
   
    public staticvoid main(String[] args) {
       //创建Singleton对象不能通过构造器，只能通过getInstance方法
       Singleton s1 = Singleton.getInstance();
       Singleton s2 = Singleton.getInstance();
       //将输出true
       System.out.println(s1 == s2);
    }
   
    //使用一个变量来缓存曾经创建的实例
    private static Singleton instance;
    //将构造器使用private修饰，隐藏该构造器
    private Singleton(){
       System.out.println("Singleton被构造！");
    }
   
    //提供一个静态方法, 用于返回 Singleton 实例
    //该方法可以加入自定义的控制，保证只产生一个Singleton对象
    public static Singleton getInstance() {
       //如果instance为null，表明还不曾创建Singleton对象
       //如果instance不为null，则表明已经创建了Singleton对象，将不会执行该方法
       if (instance == null) {
           //创建一个Singleton对象，并将其缓存起来
           instance = new Singleton();
       }
       return instance;
    }
}
```


## 观察者模式

观察者模式定义了对象之间的一对多依赖，这样一来，当一个对象改变状态时，它的所有依赖者都会收到通知并自动更新。

![](/images/posts/designpattern/ObserverPattern.png)

## 桥梁模式与策略模式

桥接(Bridge)模式是结构型模式的一种，而策略(strategy)模式则属于行为模式。以下是它们的UML结构图。 

![](/images/posts/java/bridgePattern.jpg)

在桥接模式中，Abstraction 通过聚合的方式引用 Implementor

![](/images/posts/java/strategyPattern.jpg)

**策略模式:** 我要画圆，要实心圆，我可以用solidPen来配置，画虚线圆可以用dashedPen来配置。这是strategy模式。 

策略模式用于封装一系列的算法，这些算法通常被封装在一个被称为 Context 的类中，客户端程序可以自由选择其中一种算法，
或让Context为客户端选择一种最佳算法——使用策略模式的优势是为了支持算法的自由切换。


**桥接模式:** 同样是画圆，我是在windows下来画实心圆，就用windowPen+solidPen来配置，在unix下画实心圆就用unixPen+solidPen来配置。
如果要再windows下画虚线圆，就用windowsPen+dashedPen来配置，要在unix下画虚线圆，就用unixPen+dashedPen来配置

画圆方法中，策略只是考虑算法的替换，而桥接考虑的则是不同平台下需要调用不同的工具，接口只是定义一个方法，而具体实现则由具体实现类完成。 

某个类具有两个以上的维度变化，如果只是使用继承将无法实现这种需要，或者使得设计变得相当臃肿。

而桥接模式的做法是把变化部分抽象出来，使变化部分与主类分离开来，从而将多个的变化彻底分离。

最后提供一个管理类来组合不同维度上的变化，通过这种组合来满足业务的需要


**桥接模式:** 不仅 Implementor 具有变化（ConcreteImplementor），而且 Abstraction 也可以发生变化（RefinedAbstraction），
而且两者的变化是完全独立的，RefinedAbstraction与ConcreateImplementor之间松散耦合，它们仅仅
通过Abstraction与Implementor之间的关系联系起来。强调Implementor接口仅提供基本操作，而Abstraction则基于这些基
本操作定义更高层次的操作

**策略模式:** 并不考虑Context的变化，只有算法的可替代性。强调Strategy抽象接口的提供的是一种算法，
一般是无状态、无数据的，Context简单调用这些算法完成其操作

所以相对策略模式，桥接模式要表达的内容要更多，结构也更加复杂

桥接模式表达的主要意义其实是**接口隔离**的原则，即把本质上并不内聚的两种体系区别开来，使得它们可以松散的组合，
而策略在解耦上还仅仅是某一个算法的层次，没有到体系这一层次。 

从结构图中可以看到，策略模式的结构是包容在桥接模式结构中的，Abstraction与Implementor之间就可以认为是策略模式，
但是桥接模式一般Implementor将提供一系列的成体系的操作，而且Implementor是具有状态和数据的静态结构。
而且桥接模式Abstraction也可以独立变化。 

## 工厂方法

### 简单工厂

![](/images/posts/java/SimpleFactory.jpg)

```cpp
#include "Factory.h"
#include "ConcreteProductA.h"
#include "ConcreteProductB.h"
Product* Factory::createProduct(string proname){
	if ( "A" == proname )
	{
		return new ConcreteProductA();
	}
	else if("B" == proname)
	{
		return new ConcreteProductB();
	}
	return  NULL;
}
```

简单工厂模式最大的问题在于工厂类的职责相对过重，增加新的产品需要修改工厂类的判断逻辑，这一点与开闭原则是相违背的。

工厂类负责创建的对象比较少：由于创建的对象较少，不会造成工厂方法中的业务逻辑太过复杂。

**应用**

```java
KeyGenerator keyGen=KeyGenerator.getInstance("DESede");
```

### 工厂方法

![](/images/posts/java/FactoryMethod.jpg)

```cpp
Product* ConcreteFactory::factoryMethod(){

	return  new ConcreteProduct();
}

Factory * fc = new ConcreteFactory();
Product * prod = fc->factoryMethod();
prod->use();
```

**实例**

<!--![](/images/posts/java/loger.jpg)-->

**优缺点**

使用工厂方法模式的另一个优点是在系统中加入新产品时，无须修改抽象工厂和抽象产品提供的接口，无须修改客户端，
也无须修改其他的具体工厂和具体产品，而只要添加一个具体工厂和具体产品就可以了。这样，系统的可扩展性也就变得非常好，
完全符合“开闭原则”。

在添加新产品时，需要编写新的具体产品类，而且还要提供与之对应的具体工厂类，系统中类的个数将成对增加，
在一定程度上增加了系统的复杂度，有更多的类需要编译和运行，会给系统带来一些额外的开销。

由于考虑到系统的可扩展性，需要引入抽象层，在客户端代码中均使用抽象层进行定义，增加了系统的抽象性和理解难度，
且在实现时可能需要用到DOM、反射等技术，增加了系统的实现难度。


### 抽象方法 (两个维度)



在工厂方法模式中具体工厂负责生产具体的产品，每一个具体工厂对应一种具体产品，工厂方法也具有唯一性，一般情况下，
一个具体工厂中只有一个工厂方法或者一组重载的工厂方法。但是有时候我们需要一个工厂可以提供多个产品对象，而不是单一的产品对象。

**产品等级结构:** 产品等级结构即产品的继承结构，如一个抽象类是电视机，其子类有海尔电视机、海信电视机、TCL电视机，
则抽象电视机与具体品牌的电视机之间构成了一个产品等级结构，抽象电视机是父类，而具体品牌的电视机是其子类。

**产品族:** 在抽象工厂模式中，产品族是指由同一个工厂生产的，位于不同产品等级结构中的一组产品，
如海尔电器工厂生产的海尔电视机、海尔电冰箱，海尔电视机位于电视机产品等级结构中，海尔电冰箱位于电冰箱产品等级结构中。

![](/images/posts/java/loger.jpg)

```cpp
AbstractProductA * ConcreteFactory1::createProductA(){
	return new ProductA1();
}


AbstractProductB * ConcreteFactory1::createProductB(){
	return new ProductB1();
}

AbstractFactory * fc = new ConcreteFactory1();
AbstractProductA * pa =  fc->createProductA();
AbstractProductB * pb = fc->createProductB();
pa->use();
pb->eat();
	
AbstractFactory * fc2 = new ConcreteFactory2();
AbstractProductA * pa2 =  fc2->createProductA();
AbstractProductB * pb2 = fc2->createProductB();
pa2->use();
pb2->eat();
```

## 责任链模式

责任链模式是一种对象的行为模式。在责任链模式里，很多对象由每一个对象对其下家的引用而连接起来形成一条链。请求在这个链上传递，
直到链上的某一个对象决定处理此请求。发出这个请求的客户端并不知道链上的哪一个对象最终处理这个请求，这使得系统可以在不影响客户端的情
况下动态地重新组织和分配责任, 包括增加节点, 减少节点或者改变链的顺序。

解决**判断问题**的一个很好的办法就是责任链模式, 
另外一个对于根据数据的内容, 而不是业务逻辑来做判断时, 责任链模式特别有效

```java
final class ApplicationFilterChain implements FilterChain {
    private ApplicationFilterConfig[] filters = new ApplicationFilterConfig[0];
    
    private void internalDoFilter(ServletRequest request, ServletResponse response)
        

        // Call the next filter if there is one
        if (pos < n) {
            ApplicationFilterConfig filterConfig = filters[pos++];
            try {
                Filter filter = filterConfig.getFilter();

                filter.doFilter(request, response, this);
         
         // after everything filtered       
         servlet.service(request, response);
```


### 状态模式

![](/images/posts/java/StatePattern.jpg)

**TCP 使用状态模式**

状态模式属于行为模式, 不同的状态下, 相同的动作名会有不同的行为方式。

![](/images/posts/java/State_eg.jpg)

```cpp
class Context
{

public:
	Context();
	virtual ~Context();

	void changeState(State * st);
	void request();

private:
	State *m_pState;
};

Context::Context(){
	//default is a
	m_pState = ConcreteStateA::Instance();
}

Context::Context(){
	//default is a
	m_pState = ConcreteStateA::Instance();
}

Context::~Context(){
}

void Context::changeState(State * st){
	m_pState = st;
}

void Context::request(){
	m_pState->handle(this);
}

```

### 模板模式

又叫模板方法模式，在一个方法中定义一个算法的骨架，而将一些步骤延迟到子类中。模板方法使得子类可以在不改变算法结构的情冴下，重新定义算法中的某些步骤。

```java
public abstract class Beverage {  
    public final void create(){  
        boilWater();//把水煮沸  
        brew();//用沸水冲泡...  
        pourInCup();//把...倒进杯子  
        addCoundiments();//加...  
    }  
  
    public abstract void addCoundiments();  
  
    public abstract void brew();  
      
    public void boilWater() {  
        System.out.println("煮开水");  
    }  
    public void pourInCup() {  
        System.out.println("倒进杯子");  
    }  
}  
```

1. 模板模式定义了算法的步骤，把这些步骤的实现延迟到子类
2. 模板模式为我们提供了一个代码复用的技巧
3. 模板抽象类中可以定义具体方法, 抽象方法和钩子方法
4. 为了防止子类改变模板中的算法, 可以将模板方法声明为final
5. 钩子是一种方法, 它在抽象类中不做事, 或只做默认的事, 子类可以选择要不要实现它

### 访问者模式

定义（源于GoF《Design Pattern》）：表示一个作用于某**对象结构**中的各元素的操作。它使你可以在
不改变各元素类的前提下定义作用于这些元素的新操作。从定义可以看出结构对象是使用访问者模式必备
条件，而且这个结构对象必须存在遍历自身各个对象的方法。


![](/images/posts/designpattern/Visitor-1.jpg)

下面这个例子少了一个 Visitable 接口

```java
public class Body {  
    public void accept(IVisitor visitor) {  
        visitor.visit(this);  
    }  
}  

public class Engine {  
    public  void accept(IVisitor visitor) {  
             visitor.visit(this);  
     }  
}  


public class Wheel {  
    private String name;  
    public Wheel(String name) {  
        this.name = name;  
    }  
    String getName() {  
        return this.name;  
    }  
    public  void accept(IVisitor visitor) {  
        visitor.visit(this);  
    }  
}  


public class Car {  
    private Engine  engine = new Engine();  
    private Body    body   = new Body();  
    private Wheel[] wheels   
      = { new Wheel("front left"), new Wheel("front right"),  
            new Wheel("back left") , new Wheel("back right")  };  
    
    public void accept(IVisitor visitor) {  
        visitor.visit(this);  
        engine.accept(visitor);  
        body.accept(visitor) ;   
        for (int i = 0; i < wheels.length; ++ i)  
            wheels[i].accept(visitor);  
    }  
}  

public interface IVisitor {  
    void visit(Wheel wheel);  
    void visit(Engine engine);  
    void visit(Body body);  
    void visit(Car car);  
}  

public class PrintVisitor implements IVisitor {  
  
    @Override  
    public void visit(Wheel wheel) {  
        System.out.println("Visiting " + wheel.getName() + " wheel");  
    }  
  
    @Override  
    public void visit(Engine engine) {  
        System.out.println("Visiting engine");  
    }  
  
    @Override  
    public void visit(Body body) {  
        System.out.println("Visiting body");  
    }  
  
    @Override  
    public void visit(Car car) {  
        System.out.println("Visiting car");  
    }  
}  
```


SOA 对 Velocity 的修改, 是一种 visitor 模式, 我所做的事情就是修改 visit(literalReference) 动作, 每次遇到 LiteralReference 就把该变量
存到一个容器里。后面会遍历容器, 把变量替换为值

## 代理模式

代理模式是一种应用非常广泛的设计模式，当客户端代码需要调用某个对象时，客户端实际上不关心是否准确得到该对象，
它只要一个能提供该功能的对象即可，此时我们就可返回该对象的代理（Proxy)。**往往用于增强代理的功能。**

借助于Java提供的Proxy和InvocationHandler，可以实现在运行时生成动态代理的功能，而动态代理对象就可以作为目标对象使用，而且增强了目标对象的功能。如：

Hibernate默认启用延迟加载，当系统加载A实体时，A实体关联的B实体并未被加载出来，A实体所关联的B实体全部是代理对象——只有等到A实体真正需要访问B实体时，系统才会去数据库里抓取B实体所对应的记录。

```java
public class ImageProxy implements Image {
    //组合一个image实例，作为被代理的对象
    private Image image;
    
    //使用抽象实体来初始化代理对象
    public ImageProxy(Image image) {
       this.image = image;
    }
    
    /**
     * 重写Image接口的show()方法
     * 该方法用于控制对被代理对象的访问，
     * 并根据需要负责创建和删除被代理对象
     */
    public void show() {
       //只有当真正需要调用 image 的 show 方法时才创建被代理对象
       if (image == null) {
           image = new BigImage();
       }
       image.show();
    }
}
```


## 命令模式

某个方法需要完成某一个功能，完成这个功能的大部分步骤已经确定了，但可能有少量具体步骤无法确定，必须等到执行该方法时才可以确定。
（在某些编程语言如Ruby、Perl里，允许传入一个代码块作为参数。但Jara暂时还不支持代码块作为参数）。在Java中，
传入该方法的是一个对象，该对象通常是某个接口的匿名实现类的实例，该接口通常被称为命令接口，这种设计方式也被称为命令模式。

```java
public interface Command {
    //接口里定义的process方法用于封装“处理行为”
    void process(int[] target);
}

public class ProcessArray {
    //定义一个each()方法，用于处理数组，
    publicvoid each(int[] target , Command cmd) {
       cmd.process(target);
    }
}

public class TestCommand {
    public static void main(String[] args) {
       ProcessArray pa = new ProcessArray();
       int[] target = {3, -4, 6, 4};
       //第一次处理数组，具体处理行为取决于Command对象
       
       pa.each(target , new Command() {
           //重写process()方法，决定具体的处理行为
           publicvoid process(int[] target) {
              for (int tmp : target ) {
                  System.out.println("迭代输出目标数组的元素:" + tmp);
              }
           }
       });
       
       System.out.println("------------------");
       //第二次处理数组，具体处理行为取决于Command对象
       pa.each(target , new Command() {
           //重写process方法，决定具体的处理行为
           publicvoid process(int[] target) {
              int sum = 0;
              for (int tmp : target ) {
                  sum += tmp;         
              }
              System.out.println("数组元素的总和是:" + sum);
           }});}}
```

好像和 template pattern 很像

HibernateTemplate使用了executeXxx()方法弥补了HibernateTemplate的不足，该方法需要接受一个HibernateCallback接口，该接口的代码如下：

```java
public interface HibernateCallback {
       Object doInHibernate(Session session);
}
```

## 门面模式

随着系统的不断改进和开发，它们会变得越来越复杂，系统会生成大量的类，这使得程序流程更难被理解。门面模式可为这些类提供一个简化的接口，从而简化访问这些类的复杂性。

门面模式（Facade）也被称为正面模式、外观模式，这种模式用于将一组复杂的类包装到一个简单的外部接口中。

**原来的方式**

```java
       // 依次创建三个部门实例
       Payment pay = new PaymentImpl();
       Cook cook = new CookImpl();
       Waiter waiter = new WaiterImpl();
       // 依次调用三个部门实例的方法来实现用餐功能
       String food = pay.pay();
       food = cook.cook(food);
       waiter.serve(food);
```

**门面模式**

```java
publi cclass Facade {
    // 定义被Facade封装的三个部门
    Payment pay;
    Cook cook;
    Waiter waiter;
 
    // 构造器
    public Facade() {
       this.pay = new PaymentImpl();
       this.cook = new CookImpl();
       this.waiter = new WaiterImpl();
    }
 
    publicvoid serveFood() {
       // 依次调用三个部门的方法，封装成一个serveFood()方法
       String food = pay.pay();
       food = cook.cook(food);
       waiter.serve(food);
    }
}

Facade f = new Facade();
f.serveFood();
```

<<<<<<< HEAD
### Command and Query Responsibility Segregation (CQRS) Pattern

[12306 design](http://chuansong.me/n/2433036)

Segregate operations that read data from operations that update data by using separate interfaces. 
This pattern can maximize performance, scalability, and security; support evolution of the system over time 
through higher flexibility; and prevent update commands from causing merge conflicts at the domain level.

我觉得12306这样的业务场景，非常适合使用CQRS架构；因为首先它是一个查多写少、但是写的业务逻辑非常复杂的系统。
所以，非常适合做架构层面的读写分离，即采用CQRS架构。而且应该使用数据存储也分离的CQRS。这样CQ两端才可以完全不需要顾及对方的问题，
各自优化自己的问题即可。我们可以在C端使用DDD领域模型的思路，用良好设计的领域模型实现复杂的业务规则和业务逻辑。
而Q端则使用分布式缓存方案，实现可伸缩的查询能力。

通过 CQRS 架构，由于 CQ 两端是事件驱动的，当C端有任何状态变化，都会产生对应的事件去通知Q端，所以我们几乎可以做到Q端的准实时更新。
同时由于 CQ 两端的完全解耦，Q 端我们可以设计多种存储，如数据库和缓存 (Redis等); 数据库用于线下维护关系型数据，缓存用户实时查询。
数据库和缓存的更新速度相互不受影响，因为是并行的。对同一个事件，可以10台机器负责更新缓存，100台机器负责更新数据库。即便数据库的更新很慢，
也不会影响缓存的更新进度。这就是 CQRS 架构的好处，CQ 的架构完全不同，且我们随时可以重建一种新的 Q 端存储.

**Event Sourcing and CQRS**

The CQRS pattern is often used in conjunction with the Event Sourcing pattern. CQRS-based systems use separate 
read and write data models, each tailored to relevant tasks and often located in physically separate stores. 
When used with Event Sourcing, the store of events is the write model, and this is the authoritative source of 
information. The read model of a CQRS-based system provides materialized views of the data, typically as 
highly denormalized views. These views are tailored to the interfaces and display requirements of the 
application, which helps to maximize both display and query performance.

Using the stream of events as the write store, rather than the actual data at a point in time, 
avoids update conflicts on a single aggregate and maximizes performance and scalability. The events can 
be used to asynchronously generate materialized views of the data that are used to populate the read store.

When using CQRS combined with the Event Sourcing pattern, consider the following:

* As with any system where the write and read stores are separate, systems based on this pattern are 
  only eventually consistent. There will be some delay between the event being generated and the data 
  store that holds the results of operations initiated by these events being updated.
* The pattern introduces additional complexity because code must be created to initiate and handle events, 
  and assemble or update the appropriate views or objects required by queries or a read model. The inherent 
  complexity of the CQRS pattern when used in conjunction with Event Sourcing can make a successful 
  implementation more difficult, and requires relearning of some concepts and a different approach to 
  designing systems. However, Event Sourcing can make it easier to model the domain, and makes it easier 
  to rebuild views or create new ones because the intent of the changes in the data is preserved.
* Generating materialized views for use in the read model or projections of the data by replaying and 
  handling the events for specific entities or collections of entities may require considerable processing 
  time and resource usage, especially if it requires summation or analysis of values over long time periods, 
  because all the associated events may need to be examined. This may be partially resolved by implementing 
  snapshots of the data at scheduled intervals, such as a total count of the number of a specific action that 
  have occurred, or the current state of an entity.

```c#
// Query interface
namespace ReadModel {
  public interface ProductsDao {
    ProductDisplay FindById(int productId);
    IEnumerable<ProductDisplay> FindByName(string name);
    IEnumerable<ProductInventory> FindOutOfStockProducts();
    IEnumerable<ProductDisplay> FindRelatedProducts(int productId);
  }

  public class ProductDisplay {
    public int ID { get; set; }
    public string Name { get; set; }
    public string Description { get; set; }
    public decimal UnitPrice { get; set; }
    public bool IsOutOfStock { get; set; }
    public double UserRating { get; set; }
  }

  public class ProductInventory {
    public int ID { get; set; }
    public string Name { get; set; }
    public int CurrentStock { get; set; }
  }
}
```

```c#
public interface Icommand {
  Guid Id { get; }
}

public class RateProduct : Icommand {
  public RateProduct() {
    this.Id = Guid.NewGuid();
  }
  public Guid Id { get; set; }
  public int ProductId { get; set; }
  public int rating { get; set; }
  public int UserId {get; set; }
}

API
public interface ProductsDomain {
  void AddNewProduct(int id, string name, string description, decimal price);
  void RateProduct(int userId int rating);
  void AddToInventory(int productId, int quantity);
  void ConfirmItemsShipped(int productId, int quantity);
  void UpdateStockFromInventoryRecount(int productId, int updatedQuantity);
}
```

```c#
public class ProductsCommandHandler : 
    ICommandHandler<AddNewProduct>,
    ICommandHandler<RateProduct>,
    ICommandHandler<AddToInventory>,
    ICommandHandler<ConfirmItemShipped>,
    ICommandHandler<UpdateStockFromInventoryRecount> {
  private readonly IRepository<Product> repository;

  public ProductsCommandHandler (IRepository<Product> repository) {
    this.repository = repository;
  }

  void Handle (AddNewProduct command) {
    ...
  }

  void Handle (RateProduct command) {
    var product = repository.Find(command.ProductId);
    if (product != null) {
      product.RateProuct(command.UserId, command.rating);
      repository.Save(product);
    }
  }

  void Handle (AddToInventory command) {
    ...
  }

  void Handle (ConfirmItemsShipped command) {
    ...
  }

  void Handle (UpdateStockFromInventoryRecount command) {
    ...
  }
}
```

### Distributed Domain Driven Design Pattern

[cqrs 总结与例子](http://www.hyperlambda.com/posts/cqrs-es-in-scala/)
[github example](https://github.com/boldradius/akka-dddd-template#master)
 
=======

# scala design pattern 与 Java 的比较

### 责任链模式

```scala
case class Event(source: String)

type EventHandler = PartialFunction[Event, Unit]

val defaultHandler: EventHandler = PartialFunction(_ => ())

val keyboardHandler: EventHandler = {
  case Event("keyboard") =>
}

val mouseHandler(delay: Int): EventHandler = {
  case Event("mouse") =>
}

keyboardHandler.orElse(mouesHandler(100).orElse(defaultHandler)
```

```java
public abstract class EventHandler {
  private EventHandler next;
  
  void setNext(EventHandler handler) {next = handler}
  public void handle(Event event) {
    if(canHandle(event) doHandle(event)
    else if(next != null) next.handle(event);
  }  
  
  abstract protected boolean canHandle(Event event);
  abstract proctected void doHandle(Event event);
}

public class KeyBoardHandler extends EventHandler {
  canHandle(Event event) 
    return "keyboard".equals(event.getSource);
}

KeyboardHandler handler = new KeyBoardHandler();
handler.setNext(new MouseHandler());
```

### 命令模式

```scala
object Invoker {
  private var history: Seq[() => Unit] = Seq.empty
  
  def invoke(command :=> Unit) { //by-name parameter 也就是说不会执行
      command 
      history :+= command _
  }
}

Invoke.invoke(println("foo")
Invoker.invoke {
  println("bar1")
  println("bar2")
}
```

### 适配器模式

```scala
//interface we want to use
trait Log {
  def warning(msg)
  def error(msg)
}

//existing interface we can use
final class Logger {
  def log(logLevel, msg)
}

implicit class LoggerToLogAdapter(logger: Logger) extends Log {
  def warning = logger.log(WARNING, msg)
}
```

### 策略模式

```scala
type Strategy = (Int, Int) => Int

class Context(computer: Strategy) {
  def use(a: Int, b: Int) {compute(a, b)}
}

val add: Strategy = _ + _
val multiply: Strategy: Strategy = _ * _

new Context(multiply).use(2, 3)
```

### 依赖注入

好处不太明显

```scala
trait Repository {
  def save(user: User)
}

trait DatabaseRepository extends Repository {
	....
}

trait UserService {self: Repository => // requires repository
    def create(user: User) save(user)
}

new UserService with DataBaseRepository
```

```java
public interface Repository {
  void save(User use);
}

public class DatabaseRepository implements Repository {...}

public class UserService {
  private final Repository repository;
  
  UserService(Repository repository) {
	this.repository = repository;
  }
	
  void create(User user) {
    repository.save(user);
  }
}

new UserService(new DataBaseRepository());
```

看不出太明显的好处

### Decorator

super + trait 混入

```scala
trait Buffering extends Outputstream {
  abstract override def write(b: Byte) {
    ////something
    super.write(buffer)
  }
}

new Fileoutputstream(foo.txt) with buffering
```

这里例子换成 Log 更好一些, 但是有一个缺点是 log 必须先继承 Outputstream 才好
>>>>>>> 8e6db178be591b65e2cc2f613d57e946d7b0ccc1
