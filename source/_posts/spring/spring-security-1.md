title: Spring Security基础
author: Figthing
tags:
  - java
  - spring
  - security
categories:
  - java
  - spring
  - ''
date: 2019-05-04 13:44:00
---
最近公司项目使用了Spring Security，下面整理一下基础知识

### 概述
一个能够为基于Spring的企业应用系统提供声明式的安全訪问控制解决方式的安全框架（简单说是对访问权限进行控制嘛），应用的安全性包括用户认证（Authentication）和用户授权（Authorization）两个部分。用户认证指的是验证某个用户是否为系统中的合法主体，也就是说用户能否访问该系统。用户认证一般要求用户提供用户名和密码。系统通过校验用户名和密码来完成认证过程。用户授权指的是验证某个用户是否有权限执行某个操作。在一个系统中，不同用户所具有的权限是不同的。比如对一个文件来说，有的用户只能进行读取，而有的用户可以进行修改。一般来说，系统会为不同的用户分配不同的角色，而每个角色则对应一系列的权限。   

spring security的主要核心功能为 认证和授权，所有的架构也是基于这两个核心功能去实现的。

<!-- more -->

### 框架原理
众所周知 想要对对Web资源进行保护，最好的办法莫过于Filter，要想对方法调用进行保护，最好的办法莫过于AOP。所以springSecurity在我们进行用户认证以及授予权限的时候，通过各种各样的拦截器来控制权限的访问，从而实现安全。

如下为其主要过滤器 

| 名称 | 功能
|--|
|ChannelProcessingFilter|可根据配置进行协议的重定向（HTTP与HTTPS）|
|SecurityContextPersistenceFilter|针对每个web请求开始时加载SecurityContext，并在web请求结束时将SecurityContext保存到HttpSession中|
|ConcurrentSessionFilter|功能一，刷新session中最后更新时间；功能二，取session信息看是否过期，如果过期执行登出操作。|
|UsernamePasswordAuthenticationFilter、CasAuthenticationFilter、BasicAuthenticationFilter、其他|根据需要选择一个合适的认证过滤器|
|SecurityContextHolderAwareRequestFilter|servlet相关实现的过滤器|
|JaasApiIntegrationFilter|Jaas相关过滤器|
|RememberMeAuthenticationFilter|提供“记住我”功能的过滤器|
|AnonymousAuthenticationFilter|匿名授权过滤器|
|ExceptionTranslationFilter|捕获（Spring Security）产生的异常|
|FilterSecurityInterceptor|URL的控制及访问拒绝时产生异常|

框架的核心组件

|名称| 功能|
|--|
|SecurityContextHolder|提供对SecurityContext的访问|
|SecurityContext|持有Authentication对象和其他可能需要的信息|
|AuthenticationManager|其中可以包含多个AuthenticationProvider|
|ProviderManager|对象为AuthenticationManager接口的实现类|
|AuthenticationProvider|主要用来进行认证操作的类 调用其中的authenticate()方法去进行认证操作|
|Authentication|Spring Security方式的认证主体|
|GrantedAuthority|对认证主题的应用层面的授权，含当前用户的权限信息，通常使用角色表示|
|UserDetails|构建Authentication对象必须的信息，可以自定义，可能需要访问DB得到|
|UserDetailsService|通过username构建UserDetails对象，通过loadUserByUsername根据userName获取UserDetail对象 （可以在这里基于自身业务进行自定义的实现  如通过数据库，xml,缓存获取等）|

HttpSecurity使用

|名称| 功能|
|--|
|antMatchers("/resources/**", "/signup", "/about").permitAll()|任何用户都可以访问以"/resources/","/signup", 或者 "/about"开头的URL。|
|antMatchers("/admin/**").hasRole("ADMIN")|以 "/admin/" 开头的URL只能让拥有 "ROLE_ADMIN"角色的用户访问。|
|antMatchers("/db/**").access("hasRole('ADMIN') and hasRole('DBA')")     |任何以"/db/" 开头的URL需要同时具有 "ROLE_ADMIN" 和 "ROLE_DBA"权限的用户才可以访问。|
|antMatchers("/db/**").hasAnyRole("ADMIN", "DBA")   |任何以"/db/" 开头的URL只需要拥有 "ROLE_ADMIN" 和 "ROLE_DBA"其中一个权限的用户才可以访问。|
|anyRequest().authenticated()|尚未匹配的任何URL都要求用户进行身份验证|
|logout().logoutUrl("/api/user/logout")|指定登出的url|
|defaultSuccessUrl("/index")|指定登录成功后跳转到/index页面|
|failureUrl("/login?error")|指定登录失败后跳转到/login?error页面|
|rememberMe().tokenValiditySeconds(1209600).key("mykey")|开启cookie储存用户信息，并设置有效期为14天，指定cookie中的密钥|
|formLogin().loginPage("/api/user/login")|通过formlogin方法登录，并设置登录url为/api/user/login|