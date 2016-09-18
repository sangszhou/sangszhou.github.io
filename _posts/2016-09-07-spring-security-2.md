---
layout: post
title:  "Spring security (2) configuration"
date:   "2016-09-07 00:00:00"
categories: spring
keywords: java, spring, security
---

talking about CORS, CSRF, and how to add filters

## Add More authenticationManager

```java
@Override
protected void configure(AuthenticationManagerBuilder builder) {
    builder.parentAuthenticationManager(new OAuth2AuthenticationManager());
}
```

在 WebsecurityConfigurer 中可以配置可用额外的 authenticationManager, 其实大多数
情况下没有必要配置额外的 authenticationManager, 只要添加 AuthenticationProvider 即可

```java
public interface AuthenticationManager
    Authentication authenticate(Authentication authentication) throws AuthenticationException;

public interface AuthenticationProvider
    Authentication authenticate(Authentication authentication) throws AuthenticationException;
```

![AuthenticationManager](/images/posts/spring/security/AuthenticationManager.png)

从上面看, AuthenticationManager 与 AuthenticationProvider 的函数签名都是一样的, 想不出哪种场景
是必须用 AuthenticationManager 而不是 AuthenticationProvider 的 (OAuth2AuthenticationManager 是
AuthenticationManager 的子类, 与 ProviderManager 平级)

## add more SSO client

假设 oauth2 client 有多种 Oauth2ClientFilter 需要添加

```java
protected void configure(HttpSecurity http) throws Exception {
		
    http.antMatcher("/**").authorizeRequests()
        .antMatchers("/", "/login**", "/webjars/**").permitAll()
        .anyRequest().authenticated().and()
		.exceptionHandling()
		    .authenticationEntryPoint(new LoginUrlAuthenticationEntryPoint("/")).and().logout()
		.logoutSuccessUrl("/").permitAll().and()
		.csrf().csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse()).and()
		.addFilterBefore(ssoFilter(), BasicAuthenticationFilter.class);

private Filter ssoFilter()
	CompositeFilter filter = new CompositeFilter();
	List<Filter> filters = new ArrayList<>();
	filters.add(ssoFilter(facebook(), "/login/facebook"));
	filters.add(ssoFilter(github(), "/login/github"));
	filter.setFilters(filters);
	return filter;

private Filter ssoFilter(ClientResources client, String path) {
	OAuth2ClientAuthenticationProcessingFilter filter = new OAuth2ClientAuthenticationProcessingFilter(path);
	OAuth2RestTemplate template = new OAuth2RestTemplate(client.getClient(), oauth2ClientContext);
	filter.setRestTemplate(template);
	filter.setTokenServices(new UserInfoTokenServices(
	    client.getResource().getUserInfoUri(), client.getClient().getClientId()));
	return filter;
	
```

compositeFilter 的工作原理, 

```java
public class CompositeFilter implements Filter {
    private List<? extends Filter> filters = new ArrayList();
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
        (new CompositeFilter.VirtualFilterChain(chain, this.filters)).doFilter(request, response);
        
```

## enable remember me service

The following configuration demonstrates how to allow token based remember me authentication.
Upon authenticating if the HTTP parameter named "remember-me" exists, then the user will be remembered
even after their session expires.

```java
public class RememberMeSecurityConfig extends WebSecurityConfigurerAdapter {
    void configure(HttpSecurity http) throws Exception {
        auth.rememberMe();    
    }
}
```

![](/images/posts/spring/security/remember_me_flow.png)

![](/images/posts/spring/security/remember_me_arch.png)

**TokenBasedRememberMeService**

cookie 中记录的数据为:

```
Base64(username + expirationTime + md5Hex(username + expirationTime + password + key) 组成的
```

用户登录时，用 base64.decode，然后比较用户名密码是否正确

**PersistentTokenBasedRememberMeService**

如上图所示, 他有两种实现形式, 分别是 `JDBCTokenRepositoryImpl` 和 `InMemoryTokenRepositoryImpl`

InMemory 一般作为测试, 用户名密码往往存放到 Map 中

Jdbc 的实现需要创建一张表 persistent_logins

```
persistent_logins
    username varchar
    series varchar primary key
    token varchar
    last_used timestamp
```

这个时候 cookie 中记录的是 seris 和 username, 登录成功后 refresh token

```java
try
    tokenRepository.updateToken(newToken.getSeries(), newToken.getTokenValue(),newToken.getDate());
    addCookie(newToken, request, response);
```

验证成功后, 填充 SecurityContextHolder

```java
rememberMeAuth = authenticationManager.authenticate(rememberMeAuth);
SecurityContextHolder.getContext().setAuthentication(rememberMeAuth);
onSuccessfulAuthentication(request, response, rememberMeAuth);
```

**summary**

1) When the user successfully logs in with Remember Me checked, a login cookie is generated in addition to the standard session management cookie.

2) The login cookie contains the user's username and a random number and a current date time(this cookie is also called Token). The username and token are stored as a pair in a database table.

3) And this cookie is sent back in user response header. And this cookie is associated in every request sent to server.

4) When a non-logged-in user visits the site and presents a login cookie, the username and token are looked up in the database.
If the pair is present, the user is considered authenticated. The used token is removed from the database. A new token is generated, stored in database with the username, and issued to the user via a new login cookie.

5) If the pair is not present, the login cookie is ignored. Then redirect to login operations, then user must first successfully submit a normal username/password login form.If username is successfully authenticated then a new token is generated and persists username & token are stored as a pair in a database table and sent back new token to user as in response.

参考
[Remember-Me Authentication in Spring Security](http://sunilkumarpblog.blogspot.com/2016/01/remember-me-authentication-in-spring.html)

## logout 流程

RememberMe service 会影响到 logout 么?

Polls a series of LogoutHandlers. The handlers should be specified in the order they are required. Generally you will
want to call logout handlers TokenBasedRememberMeServices and SecurityContextLogoutHandler.

After logout, a redirect will be performed to the URL determined by either the configured LogoutSuccessHandler or
the logoutSuccessUrl, depending on which constructor was used.

```java
doFilter
    if(requiresLogout) // url pattern matching
        for(LogoutHandler handler: handlers)
            handler.logout(request, response, auth)
        logoutSuccessHandler.onLogoutSuccess(request, response, auth)

// 一个 onLogoutSuccess 的例子

AbstractAuthenticationTargetUrlRequestHandler
    RedirectStrategy redirectStrategy = new DefaultRedirectStrategy
    
    handle
        targetUrl = determineTargetUrl(request, response)
        redirectStrategy.sendRedirect(request, response, targetUrl)
```

没看到 Session 和 Logout 之间的配置, 只能通过单步调试来走了


## AccessDecisionManager

```java
// 获取这个 request 对应的 role 也可能是别的东西
Collection<ConfigAttribute> attributes = this.obtainSecurityMetadataSource()
   .getAttributes(object);

this.accessDecisionManager.decide(authenticated, object, attributes);
```

![](/images/posts/spring/security/Interceptor.png)

默认情况下使用 AffirmativeBased, 也就是说主要符合一个要求，就能成功。Voter 除了 Role 以外，还有
WebExpressionVoter, ACLVoter, DenyVoter, RoleHierarchyVoter 等等，用的比较多的 scope 应该是 WebExpressionVoter 的范畴，scope 定义在 client 信息中

Voter 里最常见的 Voter 是 Role voter, 它的认证过程非常简单，从 Authentication 中拿出 grantedAuthorities，如果包含 ConfigAttribute，那么认证通过，如果一个都没有则返回未通过

常用的 Voter 还有 ScopeVoter，在 Oauth2 的代码里提供，WebExpressionVoters

## Session 处理

share session, xss 可以放到另外一篇 blog 中

@todo

## 配置不同的数据库




