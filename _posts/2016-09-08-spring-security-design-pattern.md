---
layout: post
title:  "Spring security design pattern"
date:   "2016-09-08 00:00:00"
categories: Java
keywords: Java, Spring, Security
---

Nothing here for now

## 模板方法

把变化的从不变中抽取出来, 避免重复逻辑

```java
class AbstractAuthenticationProcessingFilter
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
        Authentication authResult;
        try {
            authResult = attemptAuthentication(request, response);
        	    if (authResult == null) {
        		    // return immediately as subclass has indicated that it hasn't completed
        			// authentication
        			return;
        	sessionStrategy.onAuthentication(authResult, request, response);
        
        catch (InternalAuthenticationServiceException failed)
        catch (AuthenticationException failed)
    
    public abstract Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response)

// 子类
// OAuth2ClientAuthenticationProcessingFilter
// ClientCredentialsTokenEndpointFilter
// UsernamePasswordAuthenticationFilter
```

`AbstractAuthenticationProcessingFilter` 定义了认证的逻辑, 认证成功调用 session 逻辑, 认证失败之后调用
 异常处理逻辑, 下面的三个子类分别对 attemptAuthentication 有自己的实现
 
## 策略模式
 
可以重用上面的例子, 在上面的例子里, sessionStrategy 也就是 `SessionAuthenticationStrategy` 的实例,
明显是策略模式的例子, 它的可替换实现可以使 `ChangeSessionIdAuthenticationStrategy`, 
`CsrfAuthenticationStrategy`, `RegisterSessionAuthenticationStrategy`, 
`ConcurrentSessionControlAuthenticationStrategy` 等等。当然可以替换为自定义的
实现。
 
## 工厂模式
 
暂时只想到一个 ThreadPool 的工厂, executors
 
```java
 public class Executors
    static ExecutorService newFixedThreadPool(int nThreads)
    static ExecutorService newWorkStealingPool(int parallelism)
    static ExecutorService newSingleThreadExecutor()
    static ExecutorService newCachedThreadPool(ThreadFactory threadFactory)
```

## 责任链模式

spring filter 就是典型的责任链模式, 一个 filter 处理完以后可以传递给下一个 filter, 也可以直接返回(出错)

```java
public interface Filter
        public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            ...
            chain.filter(request, response)
```

filterChain 中有游标变量, 指向下一个 filter 的位置, 当调用 chain 的 filter 函数时, 会执行下一个变量
 
<!--## 代理模式-->
  <!---->

 <!---->
<!--## Builder 模式-->
 <!---->
 
# 我写过的设计模式

## 责任链模式

**对 alert 的读写权限**
一个用户对 alert 是否有访问权限会根据很多条件来确定, 假如用户是 admin, 或者对这种类型的 adoption
 具有读写权限, 那么它可以访问 alert, 此外, 如果它是产生 alert 的 manager, 它也会有读权限, 如果它
 操作过这条 alert, 或者被作为动作的接受者, 它是 involved user, 此时具有访问权限, 如果它带着 alert
 对应的 token, 那么他有读权限

****
 
## 策略模式


