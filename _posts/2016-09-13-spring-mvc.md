---
layout: post
title: Spring MVC
categories: [java, spring]
keywords: java, spring
---

## Spring MVC

###指定 Spring mvc 的入口程序 (web.xml)

```xml
<servlet>
    <servlet-name> dispatcher </servlet-name>
    <servlet-class> org.springframework.web.servlet.DispatcherServlet </servlet-class>
    <load-on-startup>1</load-on-startup>
</servlet>

<servlet-mapping>
    <servlet-name> dispatcher </servlet-name>
    <url-pattern> /** </url-pattern>
</servlet-mapping>
```

DispatcherServlet 是核心分派器, 是 Spring MVC 最重要的类之一, 之后我们会对其单独展开分析

### 编写 Spring mvc 的核心配置文件 ([servlet-name]-servlet.xml)

xml 中前面的 beans 就不写了

```xml
    <!-- Enables the Spring MVC @Controller programming model -->  
    <mvc:annotation-driven />  
    
    <context:component-scan base-package="com.abc" />
    
    <bean class = "org.springframework.web.servlet.view.InternalResourceViewResolver">
        <property name = "prefix" value = "/" />
        <property name = "suffix" value = ".jsp" />
    </bean>
```

用于配置 view 的位置以及扫描范围

### 编写 Controller 代码

```java
@Controller
@RequestMapping
public class UserController {
    @RequestMapping("login")
    public ModelAndView login(String name, String password) {
        return new ModelAndView("success");
    }
}
```

### 核心分派器

Servlet 本质上是一个 Java 类, 通过 servlet 接口定义的 HttpServletRequest 对象,
我们可以处理整个请求生命周期的数据, 通过 HttpServletResponse 对象, 我们可以处理 Http 响应行为。

```java
try 
    mappedHandler = getHandler(processRequest, false);
    if(mappedHandler == null || mappedHandler.getHandelr() == null)
        noHandlerFound(processRequest, response)
        return;
    
    HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler())
    
    mv = ha.handle(processedRequest, response, mappedHandler.getHandler())
  catch (ModelAndViewDefinitionException)
    mv = ex.getModelAndView
  catch (Exception)
    Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null)
    mv = processHandlerException(processedRequest, response, handler, ex);
    errorView = (mv != null)
```

HandlerMapping 与其子类:(需要举例子)

**BeanNameUrlHandlerMapping:** 根据 spring 容器中的 bean 的定义来指定请求映射关系

**SimpleUrlHandlerMapping:** 直接指定 URL 与 Controller 的映射关系, 其中 URL 支持 Ant 风格

**DefaultAnnotationHandlerMapping:** 支持通过直接扫描 Controller 类中的 Annotation 来确定
请求映射关系

**RequestMappingHandlerMapping** 通过扫描 Controller 类中的 Annotation 来确定请求映射关系的另一个实现类

## 原理

![](/images/posts/spring/springmvc-prin.jpg)


## 基础

### 注解

**@Autowired**


示例1, autowire 参数, 并根据名字过滤, 这两个注解可以替换为 Resource(name="redBean")

```java

// 产生一个 Bean
@Qualifier("redBean") 
class Red implements Color { // Class code here }

or 
<bean id="redBean" class="com.mycompany.movies.Red"/>

@Autowire
@Qualifier("redBean")
public void setColor(Color color) {
    this.color = color;
}
```

autowire 成员变量

```
@Controller // defines that this class is a spring bean 
@RequestMapping("/users") 
public class SomeController { 

    // tells the application context to inject an instance of UserService here 
    @Autowired private UserService userService;
```




## Spring mvc 的设计原则

### Open for extension closed for modification

**使用 final 关键字来限定核心组件的核心方法**

Some methods in the core classes of spring web mvc are marked as final. This has
not been done arbitrarily, but specifically with the principle in mind.

HandlerAdapter 实现类 RequestMappingHandlerAdapter 中, 核心方法 `handleInternal` 就被定义为 final

**大量使用 private 方法**






