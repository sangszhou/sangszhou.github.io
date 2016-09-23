---
layout: post
title: Design pattern
categories: [design pattern]
keywords: design pattern
---

## 观察者模式

观察者模式定义了对象之间的一对多依赖，这样一来，当一个对象改变状态时，它的所有依赖者都会收到通知并自动更新。

![](/images/posts/designpattern/ObserverPattern.png)

## 桥梁模式与策略模式

桥接(Bridge)模式是结构型模式的一种，而策略(strategy)模式则属于行为模式。以下是它们的UML结构图。 

![](/images/posts/java/bridgePattern.jpg)

在桥接模式中，Abstraction通过聚合的方式引用Implementor

![](/images/posts/java/strategyPattern.jpg)

**策略模式:** 我要画圆，要实心圆，我可以用solidPen来配置，画虚线圆可以用dashedPen来配置。这是strategy模式。 

**桥接模式:** 同样是画圆，我是在windows下来画实心圆，就用windowPen+solidPen来配置，在unix下画实心圆就用unixPen+solidPen来配置。
如果要再windows下画虚线圆，就用windowsPen+dashedPen来配置，要在unix下画虚线圆，就用unixPen+dashedPen来配置

画圆方法中，策略只是考虑算法的替换，而桥接考虑的则是不同平台下需要调用不同的工具，接口只是定义一个方法，而具体实现则由具体实现类完成。 

**桥接模式:** 不仅Implementor具有变化（ConcreteImplementor），而且Abstraction也可以发生变化（RefinedAbstraction），
而且两者的变化是完全独立的，RefinedAbstraction与ConcreateImplementor之间松散耦合，它们仅仅
通过Abstraction与Implementor之间的关系联系起来。强调Implementor接口仅提供基本操作，而Abstraction则基于这些基
本操作定义更高层次的操作

**策略模式:** 并不考虑Context的变化，只有算法的可替代性。强调Strategy抽象接口的提供的是一种算法，
一般是无状态、无数据的，Context简单调用这些算法完成其操作

所以相对策略模式，桥接模式要表达的内容要更多，结构也更加复杂。 

桥接模式表达的主要意义其实是接口隔离的原则，即把本质上并不内聚的两种体系区别开来，使得它们可以松散的组合，
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

![](/images/posts/java/loger.jpg)

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