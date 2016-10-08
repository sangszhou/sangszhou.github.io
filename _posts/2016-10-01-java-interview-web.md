---
layout: post
title: interview web
categories: [interview]
keywords: web, interview
---


### How do you setup LDAP Authentication using Spring Security? 

This is a very popular Spring Security interview question as Spring provides out of the box support to 
connect Windows Active directory for LDAP authentication and with few configuration in Spring config file 
you can have this feature enable. See How to perform LDAP authentication in Java using Spring Security for detailed code
explanation and sample.

### How do you control concurrent Active session using Spring Security?

Another Spring interview question which is based on Out of box feature provided by Spring framework. You can easily control How many active session a user can have with a Java application by using Spring Security. See this spring security example of how to control concurrent session in Java and Spring for exact details.

### Question1: What is IOC or inversion of control? (answer)
    
Answer: This Spring interview question is the first step towards Spring framework and many interviewers starts Spring 
interview from this question. As the name implies Inversion of control means now we have inverted the control of creating 
the object from our own using new operator to container or framework. Now it’s the responsibility of container to create 
an object as required. We maintain one XML file where we configure our components, services, all the classes and their 
property. We just need to mention which service is needed by which component and container will create the object for us. 
This concept is known as dependency injection because all object dependency (resources) is injected into it by the framework.

```xml
<bean id="createNewStock" class="springexample.stockMarket.CreateNewStockAccont"> 
        <property name="newBid"/>
  </bean>
```

In this example, CreateNewStockAccont class contain getter and setter for newBid and container will instantiate newBid and set the value automatically when it is used. This whole process is also called wiring in Spring and by using annotation it can be done automatically by Spring, refereed as auto-wiring of bean in Spring.

### Question 2: Explain the Spring Bean-LifeCycle.

Ans: Spring framework is based on IOC so we call it as IOC container also So Spring beans reside inside the IOC container. Spring beans are nothing but Plain old java object (POJO).
Following steps explain their life cycle inside the container.
1. The container will look the bean definition inside configuration file (e.g. bean.xml).
2. using reflection container will create the object and if any property is defined inside the bean definition then it will also be set.
3. If the bean implements the BeanNameAware interface, the factory calls setBeanName() passing the bean’s ID.
4. If the bean implements the BeanFactoryAware interface, the factory calls setBeanFactory(), passing an instance of itself.
5. If there are any BeanPostProcessors associated with the bean, their post- ProcessBeforeInitialization() methods will be called before the properties for the Bean are set.
6. If an init() method is specified for the bean, it will be called.
7. If the Bean class implements the DisposableBean interface, then the method destroy() will be called when the Application no longer needs the bean reference.
8. If the Bean definition in the Configuration file contains a 'destroy-method' attribute, then the corresponding method definition in the Bean class will be called.

### Question 3: what is Bean Factory, have you used XMLBeanFactory?
    
Ans: BeanFactory is factory Pattern which is based on IOC design principles.it is used to make a clear separation between application configuration and dependency from actual code. The XmlBeanFactory is one of the implementations of bean Factory which we have used in our project. The org.springframework.beans.factory.xml.XmlBeanFactory is used to create bean instance defined in our XML file.

BeanFactory factory = new XmlBeanFactory(new FileInputStream("beans.xml"));
Or

ClassPathResource resorce = new ClassPathResource("beans.xml"); 
XmlBeanFactory factory = new XmlBeanFactory(resorce);

### Question 4: What are the difference between BeanFactory and ApplicationContext in spring? (answer)

Answer: This one is very popular spring interview question and often asks in entry level interview. ApplicationContext is the preferred way of using spring because of functionality provided by it and interviewer wanted to check whether you are familiar with it or not.


### Question 5: What are different modules in spring?

1.      The Core container module
2.      Application context module
3.      AOP module (Aspect Oriented Programming)
4.      JDBC abstraction and DAO module
5.      O/R mapping integration module (Object/Relational)
6.      Web module
7.      MVC framework module

### Question 6: What is the difference between singleton and prototype bean?

Ans: This is another popular spring interview questions and an important concept to understand. Basically, a bean has scopes which define their existence on the application
Singleton: means single bean definition to a single object instance per Spring IOC container.
Prototype: means a single bean definition to any number of object instances.
Whatever beans we defined in spring framework are singleton beans. There is an attribute in bean tag named ‘singleton’ if specified true then bean becomes singleton and if set to false then the bean becomes a prototype bean. By default, it is set to true. So, all the beans in spring framework are by default singleton beans.


```xml
<bean id="createNewStock"     class="springexample.stockMarket.CreateNewStockAccont" singleton=”false”> 
        <property name="newBid"/> 
  </bean>
```

### Question 7: What type of transaction Management Spring support?

Ans: This spring interview questions is little difficult as compared to previous questions just because transaction management is a complex concept and not every developer familiar with it. Transaction management is critical in any applications that will interact with the database. The application has to ensure that the data is consistent and the integrity of the data is maintained.  Following two type of transaction management is supported by spring:

1. Programmatic transaction management
2. Declarative transaction management.


### Question 8: What is AOP?

Answer: The core construct of AOP is the aspect, which encapsulates behaviors affecting multiple classes into reusable modules. AOP is a programming technique that allows a developer to modularize crosscutting concerns,  that cuts across the typical divisions of responsibility, such as logging and transaction management. Spring AOP, aspects are implemented using regular classes or regular classes annotated with the @Aspect annotation. You can also check out these Spring MVC interview questions for more focus on Java web development using Spring framework. 

### Question 9: Explain Advice?

Answer: It’s an implementation of aspect; advice is inserted into an application at join points. Different types of advice include “around,” “before” and “after” advice

### Question 10: What is joint Point and point cut?

Ans: This is not really a spring interview questions I would say an AOP one.  Similar to Object oriented programming, AOP is another popular programming concept which complements OOPS. A join point is an opportunity within the code for which we can apply an aspect. In Spring AOP, a join point always represents a method execution.

Pointcut: a predicate that matches join points. A point cut is something that defines at what join-points an advice should be applied.

### Question 11: Difference between the setter and constructor injection in Spring? (answer)

[link](http://javarevisited.blogspot.com/2012/11/difference-between-setter-injection-vs-constructor-injection-spring-framework.html)

### Question 12: How to implement Role base Access Control (RBAC) using Spring Security?

http://javarevisited.blogspot.com/2013/07/role-based-access-control-using-spring-security-ldap-authorities-mapping-mvc.html

### Question 13: How to call the stored procedure from Java using Spring Framework?
    
http://javarevisited.blogspot.com/2013/04/spring-framework-tutorial-call-stored-procedures-from-java.html
    
### Question 14: How to setup JDBC Database connection pool in Spring Web application? (answer)
    
http://javarevisited.blogspot.com/2012/06/jdbc-database-connection-pool-in-spring.html

### Question 15: Difference between Factory pattern and Dependency injection in Java? (answer)

http://javarevisited.blogspot.com/2015/06/difference-between-dependency-injection.html

### What is REST and RESTful web services ?

this is the first REST interview question on most of interviews as not everybody familiar with REST and also
start discussion based on candidates response. Anyway REST stands for REpresentational State Transfer (REST) its a 
relatively new concept of writing web services which enforces a stateless client server design where web services are 
treated as resource and can be accessed and identified by there URL unlike SOAP web services which were defined by WSDL.

Web services written by apply REST Architectural concept are called RESTful web services which focus on System resources 
and how state of Resource should be transferred over http protocol to a different clients written in different languages. 
In RESTful web services http methods like GET, PUT, POST and DELETE can can be used to perform CRUD operations.

### What is differences between RESTful web services and SOAP web services

Though both RESTful web series and SOAP web service can operate cross platform they are architecturally different to each other, here is some of differences between REST and SOAP:

1) REST is more simple and easy to use than SOAP
2) REST uses HTTP protocol for producing or consuming web services while SOAP uses XML.
3) REST is lightweight as compared to SOAP and preferred choice in mobile devices and PDA's.
4) REST supports different format like text, JSON and XML while SOAP only support XML.
5) REST web services call can be cached to improve performance.

### What is Resource in REST framework

it represent a "resource" in REST architecture. on RESTLET API it has life cycle methods like init(), handle() 
and release() and contains a Context, Request and Response corresponding to specific target resource. This is now 
deprecated over ServerResource class and you should use that. see Restlet documentation for more details.

### What happens if RestFull resources are accessed by multiple clients ? do you need to make it thread-safe?

Since a new Resource instance is created for every incoming Request there is no need to make it thread-safe or 
add synchronization. multiple client can safely access RestFull resources concurrently.

That’s all on REST interview questions , I will add couple of  more REST Interview questions whenever I got them 
from my friend circle.