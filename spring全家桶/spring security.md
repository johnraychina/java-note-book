authentication认证、鉴权
authorization授权


https://developer.ibm.com/zh/technologies/web-development/articles/wa-spring-security-web-application-and-fingerprint-login/


# Spring security核心设计：
- SecurityContextHolder 是一个存储代理，存储security context对象。
- SecurityContext 当前用户的安全信息，负责获取Authentication
- Authentication 认证信息（我是谁，密码，我有哪些权限，主体，详情）
- AuthenticationManager 自己实现认证逻辑，负责校验Authentication对象，抛出对应的异常。
- Userdetails 用户信息（我是谁）

## AuthenticationManager
AuthenticationManager 负责执行认证，返回的Authentication会被设置到SecurityContextHolder中


基本校验流程
```java
AuthenticationManager amanager = new CustomAuthenticationManager();
Authentication namePwd = new CustomAuthentication("name”, "password”);
try {
       Authentication result = amanager.authenticate(namePwd);
       SecurityContextHolder.getContext.setAuthentication(result);
} catch(AuthenticationException e) {
       // TODO 验证失败
}
```

# Spring Security 在Web中的设计
https://docs.spring.io/spring-security/site/docs/5.3.4.RELEASE/reference/html5/

spring security 对servlet的支持是基于web容器的Filter实现的，为什么呢？
Spring Security integrates with the Servlet Container by using a standard Servlet Filter. 
This means it works with any application that runs in a Servlet Container. 
More concretely, you do not need to use Spring in your Servlet-based application to take advantage of Spring Security.


spring security在Web应用中的使用相对复杂一点，会涉及到很多组件：
- DelegatingFilterProxy + FilterChainProxy 作为web接入spring security的入口
ContextLoaderListener所创建，所以需要延迟查找。
- ProviderManager 它是最常用的AuthenticationManager实现类，代理模式，将校验功能交给AuthenticationProvider
- AuthenticationProvider 被ProviderManager调用，复制执行一个特定类型的Authentication校验
- ExceptionTranslationFilter 处理security exceptions
- AbstractAuthenticationProcessingFilter 认证的基础过滤器，这它在较高层次地说明了各个组件是怎样啮合的。

## FilterChainProxy 
DelegatingFilterProxy + FilterChainProxy 作为web接入spring security的入口
代理允许延迟查找，这非常重要：spring security认证要用到spring容器中的bean，但容器是由Filter之后的
而且FilterChainProxy可以实现动态选取SecurityFilterChain

## Username/Password Authentication 用户密码登录
## Session Management 会话管理
http sesion相关功能由SessionManagementFilter+SessionAuthenticationStrategy来处理。
常用于session个数保护，session超时检测，并发session数控制。

## Remember-Me Authentication 
## OpenID Support 开放id支持
## Anonynous Authentication 匿名认证


 # OAuth2登录怎么实现？




# 怎么实现这个逻辑？

1、未登录则跳转登录页面
2、登录时，认证
3、登录成功后，下次请求不需要再登录：通过session获取登录token做校验?


# 如何设计一个防爬的系统？
