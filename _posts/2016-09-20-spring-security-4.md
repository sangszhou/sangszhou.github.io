---
layout: post
title: spring security 4 (interfaces)
categories: [spring]
keywords: spring, security
---

It's all about spring security interfaces

### ClientDetails and UserDetails

```java
interface ClientDetails extends Serializable
    String getClientId();
    Set<String> getResourceIds();
    boolean isSecretRequired();
    String getClientSecret();
    Set<String> getScope();
    Set<String> getAuthorizedGrantTypes();
    Set<String> getRegisteredRedirectUri();
    Collection<GrantedAuthority> getAuthorities();
    boolean isAutoApprove(String scope);
    Map<String, Object> getAdditionalInformation();
    Integer getRefreshTokenValiditySeconds();
```
 

### User

```java
public interface UserDetails extends Serializable
    Collection<? extends GrantedAuthority> getAuthorities();
    String getPassword();
    String getUsername();
    boolean isAccountNonExpired();
```

```java
interface GrantedAuthority extends Serializable
    String getAuthority();
```

```java
interface ClientDetailsService
    ClientDetails loadClientByClientId(String clientId)

interface UserDetailsService
    UserDetails loadUserByUsername(String username)       
```

### Token

```java
interface Principal
    public String getName();

interface Authentication extends Principal
    Collection<? extends GrantedAuthority> getAuthorities();
    Object getCredentials();
    Object getDetails();
    Object getPrincipal();
    boolean isAuthenticated();

class OAuth2Authentication extends AbstractAuthenticationToken
    private final Authentication userAuthentication;
    private final OAuth2Request storedRequest;
	public boolean isAuthenticated() {
		return this.storedRequest.isApproved()
				&& (this.userAuthentication == null || this.userAuthentication.isAuthenticated());
	}
```

```java
interface TokenStore
    OAuth2Authentication readAuthentication(OAuth2AccessToken token);
    OAuth2Authentication readAuthentication(String token);
    void storeAccessToken(OAuth2AccessToken token, OAuth2Authentication authentication);
    OAuth2AccessToken readAccessToken(String tokenValue);
    void removeAccessToken(OAuth2AccessToken token);
```

### Oauth2 Endpoint

**Authorization code**

获取 code

```
GET /authorize?response_type=code&client_id=s6BhdRkqt3&state=xyz
    &redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb

using "application/x-www-form-urlencoded”, parameters
response_type: required. must be set to “code"
client_id: required
redirect_uri: optional
scope: optional
state: recommended, 为了解决 CSRF (cross site request forgery)
```

因为在获取 code 时, 往往已经需要用户登录, 所以需要的信息等于 client_id + username + password,
获取 code 的时候还需要用户点击确认按钮, 勾选授权的 scope

根据 code 获取 token

```
grant_type=authorization_code&code=SplxlOBeZQQYbYS6WxSbIA
     &redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb
```


**Password**

Request

```
POST /token HTTP/1.1
Host: server.example.com
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
Content-Type: application/x-www-form-urlencoded

grant_type=password&username=johndoe&password=A3ddj3w
```

Response

```
HTTP/1.1 200 OK
Content-Type: application/json;charset=UTF-8
Cache-Control: no-store
Pragma: no-cache
{
    "access_token":"2YotnFZFEjr1zCsicMWpAA",
    "token_type":"example",
    "expires_in":3600,
    "refresh_token":"tGzv3JOkF0XG5Qx2TlKWIA",
    "example_parameter":"example_value"
}
```

**client credentials**

```
POST /token HTTP/1.1
Host: server.example.com
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
```

response

```
HTTP/1.1 200 OK
Content-Type: application/json;charset=UTF-8
Cache-Control: no-store
Pragma: no-cache

{
    "access_token":"2YotnFZFEjr1zCsicMWpAA",
    "token_type":"example",
    "expires_in":3600,
    "example_parameter":"example_value"
}
```

**refresh token**

```
POST /token HTTP/1.1
Host: server.example.com
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token&refresh_token=tGzv3JOkF0XG5Qx2TlKWIA
```

这些信息都在哪处理呢