---
layout: post
title:  "Spring security (2)"
date:   "2016-09-07 00:00:00"
categories: Java
keywords: Java, Spring, Security
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

## 配置不同的数据库




