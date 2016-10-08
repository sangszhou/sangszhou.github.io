---
layout: post
title: spring extensions
categories: [spring]
keywords: spring
---

### 1. How do you setup LDAP Authentication using Spring Security

add dependency

```java
<groupId>org.springframework</groupId>
<artifactId>gs-authenticating-ldap</artifactId>
<version>0.1.0</version>

@Configuration
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

	@Override
	protected void configure(HttpSecurity http) throws Exception {
		http
			.authorizeRequests()
				.anyRequest().fullyAuthenticated()
				.and()
			.formLogin();}

	@Autowired
	public void configureGlobal(AuthenticationManagerBuilder auth) throws Exception {
		auth
			.ldapAuthentication()
				.userDnPatterns("uid={0},ou=people")
				.groupSearchBase("ou=groups")
				.contextSource().ldif("classpath:test-server.ldif");
	}}
```

**More providers**

```java
@Override
protected void configure(AuthenticationManagerBuilder auth) throws Exception {
    auth  // you have to prepare following beans also
        .authenticationProvider(getLdapAuthenticationProvider())
        .authenticationProvider(getWebAuthenticationProvider());
}
```

**Add more AuthenticationManager**

addParentAuthenticationManager, 如果默认的 authenticationManager 认证失败了, 那么就用这个

其实那个默认的 JDBC 连接并不是为了返回 JDBCAuthenticationManager, 它是为了创建 AuthenticationProvider
的子类: DaoAuthenticationProvider,  DAO 需要 UserDetailsService, JDBC 连接提供的配置正好用来创建
UserDetailsService 的子类 JdbcUserDetailsManager

所以说, 加认证机制有两种做法, 第一种做法是直接添加 AuthenticationManager, 另一种做法是添加 AuthenticationProvider。
其中第二种做法更灵活些, 因为添加额外的 AuthenticationManager 好像就只能加一个, 但是 AuthenticationProvider 在
ProviderManager 中是以 list 的形式存在的, 所以可以添加很多个

```java
public interface AuthenticationManager {
    Authentication authenticate(Authentication authentication) throws AuthenticationException;
}

public class MyDbAuthenticationManager implements AuthenticationManager {
    private LoginDb loginDb;
    private final Logger log = LoggerFactory.getLogger(getClass());
    public IcamDbAuthenticationManager(LoginDb db) {
        this.loginDb = db;
    }

    // 没有 throw exception 即为判断通过
    @Override
    public Authentication authenticate(Authentication auth) {
        if (auth instanceof UsernamePasswordAuthenticationToken) {
            Object principle = auth.getPrincipal();
            LoginMapping agent = loginDb.search((String)principle);
            try {
                if (agent != null && agent.getPassword().equals(auth.getCredentials())) {
                    return new IcamUserAuthentication(agent.getType(), agent.getTenant(), agent.getId(), agent.getAgentId());
                }
            } catch (Exception e) {
                log.error("Database error during login database lookup", e);
                return null;
            }
        }
        return null;
    }
}

// 添加到 WebSecurity 中
protected void configure(HttpSecurity http) throws Exception {
    http.addFilterBefore(new IcamAuthenticationFilter(), BasicAuthenticationFilter.class);
}

    @Autowired
public void configureGlobal(AuthenticationManagerBuilder auth) throws Exception {
    auth.jdbcAuthentication().dataSource(dataSource); // 默认的认证器, 但是生成的是 JdbcUserDetailsManager
    auth.parentAuthenticationManager(new IcamDbAuthenticationManager(loginDb));
}
```

另外加上一个 Filter, 它的作用是

LDAP authenticationManager 是怎么实现的呢?

非常复杂, 不是一个 AuthenticationManager 可以解决的掉的

### 2. How do you control concurrent Active session using Spring Security

If you wish to place constraints on a single user’s ability to log in to your application, 
Spring Security supports this out of the box with the following simple additions. First you need to add the 
following listener to your web.xml file to keep Spring Security updated about session lifecycle events:

```java
<listener>
<listener-class>
	org.springframework.security.web.session.HttpSessionEventPublisher
</listener-class>
</listener>
```

Then add the following lines to your application context:

```java
<http>
<session-management>
	<concurrency-control max-sessions="1" error-if-maximum-exceeded="true" />
</session-management>
</http>
```

The second login will then be rejected. By "rejected", we mean that the user will be sent to the 
authentication-failure-url if form-based login is being used. If the second authentication takes place 
through another non-interactive mechanism, such as "remember-me", an "unauthorized" (401) error will be 
sent to the client. If instead you want to use an error page, you can add the attribute 
session-authentication-error-url to the session-management element.

Adds support for concurrent session control, allowing limits to be placed on the number of active sessions 
a user can have. A `ConcurrentSessionFilter` will be created, and a `ConcurrentSessionControlStrategy` will be used 
with the `SessionManagementFilter`. If a form-login element has been declared, the strategy object will 
also be injected into the created authentication filter. An instance of SessionRegistry (a SessionRegistryImpl 
instance unless the user wishes to use a custom bean) will be created for use by the strategy.


**Session Fixation Attack Protection**

Session fixation attacks are a potential risk where it is possible for a malicious attacker to create 
a session by accessing a site, then persuade another user to log in with the same 
session (by sending them a link containing the session identifier as a parameter, for example). 
Spring Security protects against this automatically by creating a new session or otherwise changing the 
session ID when a user logs in. If you don’t require this protection, or it conflicts with some other 
requirement, you can control the behavior using the session-fixation-protection attribute on <session-management>, 
which has four options.







