---
layout: post
title: spring security core
categories: [spring]
keywords: spring
---

## Spring security Architecture

### technique overview

**Authentication in a Web Application**

1. You visit the home page, and click on a link.
2. A request goes to the server, and the server decides that you’ve asked for a protected resource.
3. As you’re not presently authenticated, the server sends back a response indicating that you must authenticate. The response will either be an HTTP response code, or a redirect to a particular web page.
4. Depending on the authentication mechanism, your browser will either redirect to the specific web page so that you can fill out the form, or the browser will somehow retrieve your identity (via a BASIC authentication dialogue box, a cookie, a X.509 certificate etc.).
5. The browser will send back a response to the server. This will either be an HTTP POST containing the contents of the form that you filled out, or an HTTP header containing your authentication details.
6. Next the server will decide whether or not the presented credentials are valid. If they’re valid, the next step will happen. If they’re invalid, usually your browser will be asked to try again (so you return to step two above).
7. The original request that you made to cause the authentication process will be retried. 
   Hopefully you’ve authenticated with sufficient granted authorities to access the protected resource. 
   If you have sufficient access, the request will be successful. Otherwise, you’ll receive back an HTTP 
   error code 403, which means "forbidden".

**Obtaining information about the current user**

Inside the SecurityContextHolder we store details of the principal currently interacting 
with the application. Spring Security uses an Authentication object to represent this information. 
You won’t normally need to create an Authentication object yourself, but it is fairly common for users 
to query the Authentication object.

```java
Object principal = SecurityContextHolder.getContext().getAuthentication().getPrincipal();

if (principal instanceof UserDetails) {
    String username = ((UserDetails)principal).getUsername();
} else {
    String username = principal.toString();}
```

The object returned by the call to getContext() is an instance of the SecurityContext interface. 
This is the object that is kept in thread-local storage. As we’ll see below, most authentication mechanisms 
withing Spring Security return an instance of **UserDetails** as the principal.

On successful authentication, UserDetails is used to build the Authentication object that is 
stored in the SecurityContextHolder (more on this below). The good news is that we provide a number 
of UserDetailsService implementations, including one that uses an in-memory map (InMemoryDaoImpl) and 
another that uses JDBC (JdbcDaoImpl). Most users tend to write their own, though, with their implementations 
often simply sitting on top of an existing Data Access Object (DAO) that represents their employees, 
customers, or other users of the application. Remember the advantage that whatever your UserDetailsService 
returns can always be obtained from the SecurityContextHolder using the above code fragment.

1. SecurityContextHolder, to provide access to the SecurityContext.
2. SecurityContext, to hold the Authentication and possibly request-specific security information.
3. Authentication, to represent the principal in a Spring Security-specific manner.
4. GrantedAuthority, to reflect the application-wide permissions granted to a principal.
5. UserDetails, to provide the necessary information to build an Authentication object from your application’s DAOs or other source of security data.
6. UserDetailsService, to create a UserDetails when passed in a String-based username (or certificate ID or the like).

**ExceptionTranslationFilter**

ExceptionTranslationFilter is a Spring Security filter that has responsibility for detecting any Spring Security 
exceptions that are thrown. Such exceptions will generally be thrown by an AbstractSecurityInterceptor, which is 
the main provider of authorization services.

the ExceptionTranslationFilter offers this service, with specific responsibility for either returning error code 403 (if the principal has been authenticated and therefore simply lacks sufficient access - as per step seven above), or launching an AuthenticationEntryPoint (if the principal has not been authenticated and therefore we need to go commence step three).

**AuthenticationEntryPoint**

As you can imagine, each web application will have a default authentication strategy (well, this can be configured like nearly everything else in Spring Security, but let’s keep it simple for now). Each major authentication system will have its own AuthenticationEntryPoint implementation, which typically performs one of the actions described in step 3.

**Storing the SecurityContext between requests**

Depending on the type of application, there may need to be a strategy in place to store the 
security context between user operations. In a typical web application, a user logs in once and is 
subsequently identified by their session Id. The server caches the principal information for the 
duration session. In Spring Security, the responsibility for storing the SecurityContext between 
requests falls to the SecurityContextPersistenceFilter, which by default stores the context as an 
HttpSession attribute between HTTP requests. It restores the context to the SecurityContextHolder 
for each request and, crucially, clears the SecurityContextHolder when the request completes. 
You shouldn’t interact directly with the HttpSession for security purposes. There is simply no 
justification for doing so - always use the SecurityContextHolder instead.

Many other types of application (for example, a stateless RESTful web service) do not use HTTP 
sessions and will re-authenticate on every request. However, it is still important that the 
SecurityContextPersistenceFilter is included in the chain to make sure that the SecurityContextHolder 
is cleared after each request.

### Access-Control (Authorization) in Spring Security

The main interface responsible for making access-control decisions in Spring Security is the AccessDecisionManager. 
It has a decide method which takes an Authentication object representing the principal requesting access, a "secure object" 
(see below) and a list of security metadata attributes which apply for the object (such as a list of roles which are required for access to be granted).

**Secure Objects and the AbstractSecurityInterceptor**

Each supported secure object type has its own interceptor class, which is a subclass 
of AbstractSecurityInterceptor. Importantly, by the time the AbstractSecurityInterceptor is called, 
the SecurityContextHolder will contain a valid Authentication if the principal has been authenticated.

AbstractSecurityInterceptor provides a consistent workflow for handling secure object requests, typically:

1. Look up the "configuration attributes" associated with the present request
2. Submitting the secure object, current Authentication and configuration attributes to the 
   AccessDecisionManager for an authorization decision
3. Optionally change the Authentication under which the invocation takes place
4. Allow the secure object invocation to proceed (assuming access was granted)
5. Call the AfterInvocationManager if configured, once the invocation has returned. 
   If the invocation raised an exception, the AfterInvocationManager will not be invoked.

![](/images/posts/spring/security/security-interception.png)

**What are Configuration Attributes?**

A "configuration attribute" can be thought of as a String that has special meaning to the classes used by AbstractSecurityInterceptor.

They are represented by the interface ConfigAttribute within the framework.
They may be simple role names or have more complex meaning, depending on the 
how sophisticated the AccessDecisionManager implementation is.

The AbstractSecurityInterceptor is configured with a SecurityMetadataSource which it uses to look 
up the attributes for a secure object. Usually this configuration will be hidden from the user.

Configuration attributes will be entered as annotations on secured methods or as access attributes on secured URLs. 
For example, when we saw something like <intercept-url pattern='/secure/**' access='ROLE_A,ROLE_B'/> in the namespace 
introduction, this is saying that the configuration attributes ROLE_A and ROLE_B apply to web requests matching the given pattern.


### Authorization Architecture

all Authentication implementations store a list of GrantedAuthority objects. These represent the 
authorities that have been granted to the principal. the GrantedAuthority objects are inserted into the 
Authentication object by the AuthenticationManager and are later read by AccessDecisionManager s when making 
authorization decisions.

AccessDecisionManager

```java
void decide(Authentication authentication, Object secureObject,
	Collection<ConfigAttribute> attrs) throws AccessDeniedException;

boolean supports(ConfigAttribute attribute);
boolean supports(Class clazz);
```

The supports(ConfigAttribute) method is called by the AbstractSecurityInterceptor at startup time to determine 
if the AccessDecisionManager can process the passed ConfigAttribute. The supports(Class) method is called by a 
security interceptor implementation to ensure the configured AccessDecisionManager supports the type of secure 
object that the security interceptor will present.

**Voting-Based AccessDecisionManager Implementations**

![](/images/posts/spring/security/access-decision-voting.png)

Using this approach, a series of AccessDecisionVoter implementations are polled on an authorization decision. 
The AccessDecisionManager then decides whether or not to throw an AccessDeniedException based on its assessment of the votes.

```java
int vote(Authentication authentication, Object object, Collection<ConfigAttribute> attrs);

boolean supports(ConfigAttribute attribute);
boolean supports(Class clazz);
```

Concrete implementations return an int, with possible values being reflected in the AccessDecisionVoter static 
fields ACCESS_ABSTAIN, ACCESS_DENIED and ACCESS_GRANTED. A voting implementation will return ACCESS_ABSTAIN if it has 
no opinion on an authorization decision. If it does have an opinion, it must return either ACCESS_DENIED or ACCESS_GRANTED.

**After Invocation Handling**

Whilst the AccessDecisionManager is called by the AbstractSecurityInterceptor before proceeding 
with the secure object invocation, some applications need a way of modifying the object actually returned by 
the secure object invocation. Whilst you could easily implement your own AOP concern to achieve this, 
Spring Security provides a convenient hook that has several concrete implementations that integrate with its ACL capabilities.

![](/images/posts/spring/security/after-invocation.png)


