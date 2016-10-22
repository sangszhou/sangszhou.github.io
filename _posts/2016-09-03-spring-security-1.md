---
layout: post
title:  "Spring security (1) how it works"
date:   "2016-09-06 00:00:00"
categories: spring
keywords: java, spring, Security
---

### CSRFFilter

在web应用中增加前置过滤器，对需要验证的请求验证是否包含csrf的token信息，如果不包含，则报错。这样攻击网站无法获取到token信息，则跨域提交的信息都无法通过过滤器的校验。

CookieCsrfTokenRepository persist the CSRF token in a cookie named "XSRF-TOKEN" and reads from the header "X-XSRF-TOKEN" following the conventions of AngularJS. 除了 Cookie 以外，还可以保存在 HttpSessionCsrfTokenRepository 中

判断逻辑很简单， 就是查看 token 和记录的是否一致

```java
protected void doFilterInternal(HttpServletRequest request,
			HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {

    request.setAttribute(HttpServletResponse.class.getName(), response);

		CsrfToken csrfToken = this.tokenRepository.loadToken(request);
		final boolean missingToken = csrfToken == null;
		if (missingToken) {
			csrfToken = this.tokenRepository.generateToken(request);
			this.tokenRepository.saveToken(csrfToken, request, response);
		}
		request.setAttribute(CsrfToken.class.getName(), csrfToken);
		request.setAttribute(csrfToken.getParameterName(), csrfToken);

		if (!this.requireCsrfProtectionMatcher.matches(request)) {
			filterChain.doFilter(request, response);
			return;
		}

		String actualToken = request.getHeader(csrfToken.getHeaderName());
		if (actualToken == null) {
			actualToken = request.getParameter(csrfToken.getParameterName());
		}
		if (!csrfToken.getToken().equals(actualToken)) {
			if (this.logger.isDebugEnabled()) {
				this.logger.debug("Invalid CSRF token found for "
						+ UrlUtils.buildFullRequestUrl(request));
			}
			if (missingToken) {
				this.accessDeniedHandler.handle(request, response,
						new MissingCsrfTokenException(actualToken));
			}
			else {
				this.accessDeniedHandler.handle(request, response,
						new InvalidCsrfTokenException(csrfToken, actualToken));
			}
			return;
		}

		filterChain.doFilter(request, response);
	}
```

前端用法

```html
<meta name="_csrf" content="${_csrf.token}"/>  
<meta name="_csrf_header" content="${_csrf.headerName}"/>  

var token = $("meta[name='_csrf']").attr("content");  
var header = $("meta[name='_csrf_header']").attr("content");  
$(document).ajaxSend(function(e, xhr, options) {  
    xhr.setRequestHeader(header, token);  
});  
```

meta 从哪获取的数据呢？

### RememberMeAuthenticationFilter

Detects if there is no {@code Authentication} object in the {@code SecurityContext},
and populates the context with a remember-me authentication token if a
{@link RememberMeServices} implementation so requests.

```java
if (SecurityContextHolder.getContext().getAuthentication() == null) {
  Authentication rememberMeAuth = rememberMeServices.autoLogin(request,
      response);
```

这个 Filter 和 SecurityContextPersistentFilter 的区别在于，PersistentFilter 总是从 HttpSession 或者 Cookies 中
登录，但是它从 RememberMeService 中登录

RememberMeService 以前总结过，他要借助 cookie 中的数据，在某些情况下，cookie 直接保存用户名密码，另外的情况下，它保存 username 和
一个序列号，通过对比序列号和 username 获得想要的值

### LogoutFilter

Polls a series of {@link LogoutHandler}s. The handlers should be specified in the order
they are required. Generally you will want to call logout handlers
<code>TokenBasedRememberMeServices</code> and <code>SecurityContextLogoutHandler</code>
(in that order).

After logout, a redirect will be performed to the URL determined by either the
configured <tt>LogoutSuccessHandler</tt> or the <tt>logoutSuccessUrl</tt>, depending on
which constructor was used.

注释写的很清楚了

### AnonymousAuthenticationFilter

Detects if there is no {@code Authentication} object in the
{@code SecurityContextHolder}, and populates it with one if needed.

### ExceptionTranslationFilter

Handles any <code>AccessDeniedException</code> and <code>AuthenticationException</code>
thrown within the filter chain.

If an {@link AuthenticationException} is detected, the filter will launch the
<code>authenticationEntryPoint</code>. This allows common handling of authentication
failures originating from any subclass of AbstractSecurityInterceptor

If an {@link AccessDeniedException} is detected, the filter will determine whether or
not the user is an anonymous user. If they are an anonymous user, the
<code>authenticationEntryPoint</code> will be launched. If they are not an anonymous
user, the filter will delegate to the AccessDeniedHandler.

<li><code>authenticationEntryPoint</code> indicates the handler that should commence
the authentication process if an <code>AuthenticationException</code> is detected. Note
that this may also switch the current protocol from http to https for an SSL login.
<li><tt>requestCache</tt> determines the strategy used to save a request during the
authentication process in order that it may be retrieved and reused once the user has
authenticated. The default implementation is {@link HttpSessionRequestCache}.




往往这个时候已经认证过了，它可以返回 403 Forbidden, 因为要访问的权限不足，或者弹出页面让用户登录

This post gonna be long

## spring security 原理

Spring security 在访问 resource 时要预先通过一个个的过滤器，如果过滤器有对用户名密码的验证
过程，就相当于增加了对资源的保护，这就是 spring security 的实现原理。

拦截器里，最先访问到的 ApplicationFilterChain, Spring security 相关的逻辑一般写在 SpringSecurityProxy
中，

<!--## 使用 web.xml 配置安全-->

## User name password 登录流程

### 首次登陆到 localhost:8080

[源代码](https://github.com/dsyer/spring-security-angular/tree/master/single)

用户权限为 AnonymousUser



要访问的资源为 `/user`, 通过 debug 单步调试看看 request 内部的数据都有什么

* cookie 中的数据为 null
* authType, session, parameterMap all null
* userPrincipal null

```java
OrderedCharacterEncodingFilter
OrderedHiddenHttpMethodFilter
OrderedHttpPutFormContentFilter
OrderedRequestContextFilter
SpringSecurityFilter: DelegatingFilterProxyRegistrationBean
WsFilter
```

从 ApplicationFilterChain 可以直接跳到 FilterChainProxy 中，因为上面 5 个 filter 中
只有第四个与安全相关，不知道为什么 SpringSecurityFilter 是 Bean 类型的，它应该是 FilterChainProxy
的子类啊

FilterChainProxy 通过 getFilters 获取与 url 相关的 filter

```java
private List<Filter> getFilters(HttpServletRequest request) {
  for (SecurityFilterChain chain : filterChains) {
    if (chain.matches(request)) {
      return chain.getFilters();
    }
  }
  return null;
}

private List<SecurityFilterChain> filterChains;

public interface SecurityFilterChain {

	boolean matches(HttpServletRequest request);

	List<Filter> getFilters();
}
```
filterChain 是 url 与 filter 的对应关系

```java
0 = {DefaultSecurityFilterChain@5323} "[ Ant [pattern='/css/**'], []]"
1 = {DefaultSecurityFilterChain@5324} "[ Ant [pattern='/js/**'], []]"
2 = {DefaultSecurityFilterChain@5325} "[ Ant [pattern='/images/**'], []]"
3 = {DefaultSecurityFilterChain@5326} "[ Ant [pattern='/webjars/**'], []]"
4 = {DefaultSecurityFilterChain@5327} "[ Ant [pattern='/**/favicon.ico'], []]"
5 = {DefaultSecurityFilterChain@5328} "[ Ant [pattern='/error'], []]"
6 = {DefaultSecurityFilterChain@5329} "[ org.springframework.security.web.util.matcher.AnyRequestMatcher@1,

    org.springframework.security.web.context.request.async.WebAsyncManagerIntegrationFilter@6ca8fcf3,
    org.springframework.security.web.context.SecurityContextPersistenceFilter@18371d89,
    org.springframework.security.web.header.HeaderWriterFilter@643d2dae,
    org.springframework.security.web.csrf.CsrfFilter@2f860823,
    org.springframework.security.web.authentication.logout.LogoutFilter@4a8e6e89,
    org.springframework.security.web.authentication.www.BasicAuthenticationFilter@4483d35,
    org.springframework.security.web.savedrequest.RequestCacheAwareFilter@4832f03b,
    org.springframework.security.web.servletapi.SecurityContextHolderAwareRequestFilter@5115f590,
    org.springframework.security.web.authentication.AnonymousAuthenticationFilter@69f0b0f4,
    org.springframework.security.web.session.SessionManagementFilter@c017175,
    org.springframework.security.web.access.ExceptionTranslationFilter@69afa141,
    org.springframework.security.web.access.FilterSecurityInterceptor"

7 = {DefaultSecurityFilterChain@5330} "[ OrRequestMatcher [requestMatchers=[Ant [pattern='/**']]],
    org.springframework.security.web.context.request.async.WebAsyncManagerIntegrationFilter@3481ff98,
    org.springframework.security.web.context.SecurityContextPersistenceFilter@1fc713c9,
    org.springframework.security.web.header.HeaderWriterFilter@1e236278,
    org.springframework.security.web.authentication.logout.LogoutFilter@4d6ccc97,
    org.springframework.security.web.authentication.www.BasicAuthenticationFilter@6a12c7a8,
    org.springframework.security.web.savedrequest.RequestCacheAwareFilter@7301eebe,
    org.springframework.security.web.servletapi.SecurityContextHolderAwareRequestFilter@76b47204,
    org.springframework.security.web.authentication.AnonymousAuthenticationFilter@2ddb3ae8,
    org.springframework.security.web.session.SessionManagementFilter@44fff386,
    org.springframework.security.web.access.ExceptionTranslationFilter@8cc8cdb,
    org.springframework.security.web.access.intercept.FilterSecurityInterceptor@17ba57f0]]"
```

`/user` 定位到第六个 SecurityFilterChain, 他下面共有 12 个 filter，与安全相关的应该是 SecurityContextPersistenceFilter, BasicAuthenticationFilter, SessionManagementFilter

其中 SecurityContextPersistenceFilter 是用来 `load` 已登录用户的信息的，默认情况下使用 Session。
因为第一次还没登录，所以 SecurityContext 是空的，authentication 信息为 null

虽然在 Request 的数据中，没看到 Header 信息，但是从 BasicAuthenticationFilter 中还是返回了信息的

```java
String header = request.getHeader("Authorization");
header: "Basic dXN....."
```

Basic 是 Base64 加密的，直接解密即可

```java
String[] tokens = extractAndDecodeHeader(header, request);
String username = tokens[0];

if (authenticationIsRequired(username)) {
  UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(username, tokens[1]);
  authRequest.setDetails(this.authenticationDetailsSource.buildDetails(request));
  Authentication authResult = this.authenticationManager.authenticate(authRequest);

 this.logger.debug("Authentication success: " + authResult);

  SecurityContextHolder.getContext().setAuthentication(authResult);

  // 登录成功后记录 loginSuccess, 根据 rememberMeService 实现的不同, 写向 cookie 的数据也不同, 后面有讲
  this.rememberMeServices.loginSuccess(request, response, authResult);
  onSuccessfulAuthentication(request, response, authResult);
}
```

拿到用户名密码以后验证其是否有效，这里用到了 authenticationManager, 如果有效填充 SecurityContextHolder
和 rememberMeServices, 最后回调 onSuccessfulAuthentication。

authenticationManager 在另外的 post 里讲

### SessionManagementFilter

用户验证后，若 Authentication 不空，则尝试填充 SecurityContext, Session 的操作还有很多，可以通过回调函数扩展

```java
if (!securityContextRepository.containsContext(request)) {
  Authentication authentication = SecurityContextHolder.getContext().getAuthentication();

  if (authentication != null && !trustResolver.isAnonymous(authentication)) {
    // The user has been authenticated during the current request, so call the
    // session strategy
    try {
      sessionAuthenticationStrategy.onAuthentication(authentication,request, response);
    }
    catch (SessionAuthenticationException e) {
      // The session strategy can reject the authentication  
      SecurityContextHolder.clearContext();
      failureHandler.onAuthenticationFailure(request, response, e);
      return;
    }
```

认证完毕后访问 `/resource`。 resource 在认证完毕后访问, 由于 session 已经保存了 principal 信息，
在 SecurityContextPersistenceFilter 可以直接拿到用户名，BasicAuthenticationFilter 中
由于没有 Header Basic 就直接跳过 filter

```java
@RequestMapping("/resource")
public Map<String, Object> home() {
  Map<String, Object> model = new HashMap<String, Object>();
  model.put("id", UUID.randomUUID().toString());
  model.put("content", "Hello World");
  return model;
}
```

**SecurityContextPersistentFilter**

Populate the SecurityContextHolder with information obtained from the configured SecurityContextRepository prior to the
request and stores it back in the repository once the request has completed and clearing the context holder. By default,
it use HttpSessionSecurityContextRepository.


```java
SecurityContext contextBeforeChainExecution = repo.loadContext(holder)
chain.doFilter(holder.getRequest, holder.getResponse)
SecurityContextHolder.clearCotnext
repo.saveContext()

interface SecurityContextRepository
    SecurityContext loadContext(HttpRequestResponseHolder requestResponseHolder)

    // 放到 Session 中, 下次用户访问直接获取
    void saveContext(SecurityContext context, HttpServletRequest request, HttpServletResponse response);

HttpSession load SecurityContext
    Object contextFromSession = httpSession.getAttribute("SPRING_SECURITY_CONTEXT");

```


## Oauth2 token 登录流程

[源代码](https://github.com/dsyer/spring-security-angular/tree/master/oauth2)

Oauth2 是在 spring security 的基础上添加了若干 filter, 先不说代码，考虑我们要实现 oauth2 应该
加怎样的 filter. 首先是从 token 到 user principal 的映射， 这个可能是通过 filter 自己查表实现，
或者通过 RestTemplate 请求登录信息。认证的过程应该发生在 AbstractAuthenticationFilter 附近，
很可能就是它的子类

和用户名密码登录相比, oauth2 token 方式登录的 filter 只是把 BasicAuthenticationFilter 替换为
`OAuth2AuthenticationProcessingFilter`

```java
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException
    Authentication authentication = tokenExtractor.extract(request);
    logger.debug("No token in request, will continue chain.");
    request.setAttribute(OAuth2AuthenticationDetails.ACCESS_TOKEN_VALUE, authentication.getPrincipal());

    if (authentication instanceof AbstractAuthenticationToken)
        AbstractAuthenticationToken needsDetails = (AbstractAuthenticationToken) authentication;
    	needsDetails.setDetails(authenticationDetailsSource.buildDetails(request));

    Authentication authResult = authenticationManager.authenticate(authentication);

    eventPublisher.publishAuthenticationSuccess(authResult);
	SecurityContextHolder.getContext().setAuthentication(authResult);		

    catch (OAuth2Exception failed) {
        SecurityContextHolder.clearContext();
        eventPublisher.publishAuthenticationFailure(new BadCredentialsException(failed.getMessage(), failed),
            new PreAuthenticatedAuthenticationToken("access-token", "N/A"));
        authenticationEntryPoint.commence(request, response,
        	new InsufficientAuthenticationException(failed.getMessage(), failed));
```
这个 filter 做得事情就是从 header 或者 parameter 中拿到 token, 然后用 authenticationManager 验证 token 的有效性。

**拿 token**

```java
protected String extractToken(HttpServletRequest request)
    // first check the header...
	String token = extractHeaderToken(request);
	// bearer type allows a request parameter as well
	if (token == null)
		token = request.getParameter(OAuth2AccessToken.ACCESS_TOKEN);
	else
		request.setAttribute(OAuth2AuthenticationDetails.ACCESS_TOKEN_TYPE, OAuth2AccessToken.BEARER_TYPE);
	return token;
```

**认证**

```java
public Authentication authenticate(Authentication authentication) throws AuthenticationException
    String token = (String) authentication.getPrincipal();
    OAuth2Authentication auth = tokenServices.loadAuthentication(token);

    if (auth == null) throw new InvalidTokenException("Invalid token: " + token);

    Collection<String> resourceIds = auth.getOAuth2Request().getResourceIds();
    if (resourceId != null && resourceIds != null && !resourceIds.isEmpty() && !resourceIds.contains(resourceId))
        throw new OAuth2AccessDeniedException("Invalid token does not contain resource id (" + resourceId + ")");

    // check client detail

    client = clientDetailsService.loadClientByClientId(auth.getOAuth2Request().getClientId());
    Set<String> allowed = client.getScope();

    for (String scope : auth.getOAuth2Request().getScope()) {
        if (!allowed.contains(scope)) {
    	    throw new OAuth2AccessDeniedException("Invalid token contains
    	        disallowed scope (" + scope + ") for this client");    				
```

如果验证成功, 填充到 authentication 中的信息与之前就一致了


## Oauth2 SSO 实现流程

[link](/ws/github/tut-spring-boot-oauth2)

前端的请求, 到 `/user` 下查询信息

```java
<script type="text/javascript">
	angular.module("app", []).controller("home", function($http) {
		var self = this;
		$http.get("/user").success(function(data) {
			self.user = data.userAuthentication.details.name;
			self.authenticated = true;
		}).error(function() {
			self.user = "N/A";
			self.authenticated = false;
		});
	});
</script>
```

```java
Oauth2RestTemplate.acquireAccessToken
accessToken = accessTokenProvider.obtainAccessToken(resource, accessTokenRequest);
```
accessTokenProvider 由 4 种 provider 构成, 默认是哪一种, 或者如何配置暂时不知道, 在这里例子里, 提供的是
AuthorizationCodeAccessTokenProvider

```java
obtainAccessToken
    AuthorizationCodeResourceDetails resource = (AuthorizationCodeResourceDetails) details;
    if (request.getAuthorizationCode() == null) {
        if (request.getStateKey() == null) {
    		throw getRedirectForAuthorization(resource, request);
    	}
        obtainAuthorizationCode(resource, request);
    }
    return retrieveToken(request, resource, getParametersForTokenRequest(resource, request),
        getHeadersForTokenRequest(request));
```

throw getRedirectForAuthorization 异常后到了哪里?

```java
getRedirectForAuthorization
    requestParameters.put("response_type", "code") // oauth2 spec, section 3
    requestParameters.put("client_id", resource.getClientId())
    String redirectUri = resource.getRedirectUri(request)
    if (redirectUri != null)
        requestParameters.put("redirect_uri", redirectUri)

    if (resource.isScoped())
        StringBuilder builder = new StringBuilder()
        List<String> scope = resource.getScope()
        ...
    requestParameters.put("scope", builder.toString())
    String stateKey = stateKeyGenerator.generateKey(resource)
    redirectException.setStateKey(stateKey)
    request.setStateKey(stateKey)
    redirectException.setStateToPreserve(redirectUri)
    request.setPreservedState(redirectUri)

// 继续创建 request 写入需要的一切信息
obtainAuthorizationCode
    AuthorizationCodeResourceDetails resource = (AuthorizationCodeResourceDetails) details;

    if (request.containsKey(OAuth2Utils.USER_OAUTH_APPROVAL)) {
        form.set(OAuth2Utils.USER_OAUTH_APPROVAL, request.getFirst(OAuth2Utils.USER_OAUTH_APPROVAL));
    	for (String scope : details.getScope()) {
    		form.set(scopePrefix + scope, request.getFirst(OAuth2Utils.USER_OAUTH_APPROVAL));
    		final ResponseExtractor<ResponseEntity<Void>> delegate = getAuthorizationResponseExtractor();

    // 创建回调函数
    ResponseExtractor<ResponseEntity<Void>> extractor = new ResponseExtractor<ResponseEntity<Void>>() {
        public ResponseEntity<Void> extractData(ClientHttpResponse response) throws IOException {
    	if (response.getHeaders().containsKey("Set-Cookie")) {
    		copy.setCookie(response.getHeaders().getFirst("Set-Cookie"));
    		return delegate.extractData(response);

    ResponseEntity<Void> response = getRestTemplate().execute(resource.getUserAuthorizationUri(), HttpMethod.POST,
        getRequestCallback(resource, form, headers), extractor, form.toSingleValueMap());

    // 抛出异常, 让后续的 filte 拦截
    // 如果不需要用户确认, 返回 status code 是多少
    if (response.getStatusCode() == HttpStatus.OK) {
    			// Need to re-submit with approval...
        throw getUserApprovalSignal(resource, request);
    // throw exception 后面还有代码, 这个在哪里执行呢? 上面这个 exception 是不是 Runtime exception

    URI location = response.getHeaders().getLocation();
    String query = location.getQuery();
    // checking if state is correct
    String code = map.get("code");
    request.set("code", code);

retrieveToken: AccessToken
    // Prepare headers and form before going into rest template call in case the URI is affected by user
    // authenticationScheme: query 这个在 下一行的函数里用到了
    authenticationHandler.authenticateTokenRequest(resource, form, headers);
    return getRestTemplate().execute(getAccessTokenUri(resource, form), getHttpMethod(),
        getRequestCallback(resource, form, headers), extractor , form.toSingleValueMap());
```

需要用户手动点确认时, 抛出的异常在哪处理。 异常类型为 `UserApprovalRequiredException`
异常一直跳转到 Oauth2ClientContextFilter, 在里面处理跳转异常

```java
this.redirectStrategy.sendRedirect(request, response, builder.build().encode().toUriString());

public void sendRedirect(HttpServletRequest request, HttpServletResponse response, String url) throws IOException
    String redirectUrl = calculateRedirectUrl(request.getContextPath(), url);
	redirectUrl = response.encodeRedirectURL(redirectUrl);
	response.sendRedirect(redirectUrl);
```

当用户确认提交时, 怎么返回当前状态?


oauth2 client 的实现原理

```java
WebAsyncManagerIntegrationFilter
SecurityContextPersisteceFilter
HeaderWriterFilter
CsrfFilter
LogoutFilter
OAuth2ClientAuthenticationProcessingFilter
与 authentication server 相比，这个 filter 是多出来的
An oauth2 client filter than can be used to acquire an oauth2 access token from an authorization server,      and load an authentication object into security context
RequestCacheAwareFilter
SecurityContextHolderAwareRequestFilter
AnonymousAuthenticationFilter
SessionManagementFilter
ExceptionTraslationFilter
FilterSecurityInterceptor
```

对比 BasicAuthenticationFilter，OAuth2ClientAuthenticationProcessingFilter 是通过 token 换取 Principal

```java
accessToken = restTemplate.getAccessToken();
OAuth2Authentication result = tokenServices.loadAuthentication(accessToken.getValue());
if (authenticationDetailsSource!=null)
	request.setAttribute(OAuth2AuthenticationDetails.ACCESS_TOKEN_VALUE, accessToken.getValue());
	request.setAttribute(OAuth2AuthenticationDetails.ACCESS_TOKEN_TYPE, accessToken.getTokenType());
	result.setDetails(authenticationDetailsSource.buildDetails(request));

// Acquire or renew an access token for the current context if necessary. This method will be called automatically
// when a request is executed (and the result is cached), but can also be called as a standalone method to
// pre-populate the token.

public OAuth2AccessToken getAccessToken() throws UserRedirectRequiredException {

OAuth2AccessToken accessToken = context.getAccessToken();

// 如果没有自带 token, 就用 client 获取
if (accessToken == null || accessToken.isExpired()) {
	try {
		accessToken = acquireAccessToken(context);
	}
  // 如果需要重定向
	catch (UserRedirectRequiredException e) {
		context.setAccessToken(null); // No point hanging onto it now
		accessToken = null;
		String stateKey = e.getStateKey();
		if (stateKey != null) {
			Object stateToPreserve = e.getStateToPreserve();
			if (stateToPreserve == null) {
				stateToPreserve = "NONE";
			}
			context.setPreservedState(stateKey, stateToPreserve);
			}
			throw e;
	}
}
  return accessToken
}

```

SSO 写的不够详细, 不够好, 但是它是 Low priority, 没时间的话就不要看了

## 和用户认证相关的 filter 都有哪些, 每个做什么用

**OAuth2AuthenticationProcessingFilter**

Extracts an OAuth2 token from the incoming request and uses it to populate the Spring Security context with an OAuth2Authentication

**AbstractAuthenticationProcessingFilter**

抽象类, 抽象方法是 attemptAuthentication, 子类包括

> ClientCredentialsTokenEndpointFilter 直接通过 request 中的 client_id 和 client_secret 认证

> UsernamePasswordAuthenticationFilter 直接通过 request 中的 username 和 password 认证

> OAuth2ClientAuthenticationProcessingFilter 使用 RestTemplate 向授权服务器请求 token, 再从 tokenservice 获取 OAuth2Authentication

**BasicAuthenticationFilter**

Processes a HTTP request's BASIC authorization headers, putting the result into the SecurityContextHolder

```
// Base64 加密
Authorization: Basic QWxhZGRpbjpvcGVuIHNlc2FtZQ==
```

和 UsernamePasswordAuthenticationFilter 的区别是什么?

**SecurityContextPersistentFilter**

从 session 中登录

**RememberMeService**

从 cookie 中获得用户名或者用户名密码的信息, 填充 SecurityContextHolder

**SessionManagementFilter**

处理用户登录成功后的处理, 比如用户能否同时登录, 最多同时登录几个的问题




**认证后**

认证成功

SessionAuthenticationStrategy 回调, EventPublication

回调

setAuthenticationSuccessHandler, savedRequestAwareAuthenticationSuccessHandler

认证失败

AuthenticationFailureHandler

## 配置模板项目
