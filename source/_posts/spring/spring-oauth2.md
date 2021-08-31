title: Spring OAuth2基础
author: Figthing
tags:
  - java
  - spring
  - oauth2
categories:
  - java
  - spring
date: 2019-05-04 14:27:00
---
### Spring OAuth2.0 提供者实现原理

Spring OAuth2.0提供者实际上分为

- 授权服务 Authorization Service.
- 资源服务 Resource Service.

虽然这两个提供者有时候可能存在同一个应用程序中，但在Spring Security OAuth中你可以把他它们各自放在不同的应用上，而且你可以有多个资源服务，它们共享同一个中央授权服务。

配置一个授权服务必须要实现的endpoints：

- AuthorizationEndpoint：用来作为请求者获得授权的服务，默认的URL是/oauth/authorize.
- TokenEndpoint：用来作为请求者获得令牌（Token）的服务，默认的URL是/oauth/token.

配置一个资源服务必须要实现的过滤器：

- OAuth2AuthenticationProcessingFilter：用来作为认证令牌（Token）的一个处理流程过滤器。只有当过滤器通过之后，请求者才能获得受保护的资源

<!--more-->

### 授权服务配置

配置一个授权服务，你需要考虑几种授权类型（Grant Type），不同的授权类型为客户端（Client）提供了不同的获取令牌（Token）方式，为了实现并确定这几种授权，需要配置使用 ClientDetailsService 和 TokenService 来开启或者禁用这几种授权机制。到这里就请注意了，不管你使用什么样的授权类型（Grant Type），每一个客户端（Client）都能够通过明确的配置以及权限来实现不同的授权访问机制。这也就是说，假如你提供了一个支持"client_credentials"的授权方式，并不意味着客户端就需要使用这种方式来获得授权。

- authorization_code：授权码类型。
- implicit：隐式授权类型。
- password：资源所有者（即用户）密码类型。
- client_credentials：客户端凭据（客户端ID以及Key）类型。
- refresh_token：通过以上授权获得的刷新令牌来获取新的令牌。

#### 配置客户端详情信息
ClientDetailsServiceConfigurer (AuthorizationServerConfigurer 的一个回调配置项，见上的概述) 能够使用内存或者JDBC来实现客户端详情服务（ClientDetailsService），有几个重要的属性如下列表：

- clientId：（必须的）用来标识客户的Id。
- secret：（需要值得信任的客户端）客户端安全码，如果有的话。
- scope：用来限制客户端的访问范围，如果为空（默认）的话，那么客户端拥有全部的访问范围。
- authorizedGrantTypes：此客户端可以使用的授权类型，默认为空。
- authorities：此客户端可以使用的权限（基于Spring Security authorities）。

#### 配置授权类型
授权是使用 AuthorizationEndpoint 这个端点来进行控制的，你能够使用 AuthorizationServerEndpointsConfigurer 这个对象的实例来进行配置(AuthorizationServerConfigurer 的一个回调配置项，见上的概述) ，如果你不进行设置的话，默认是除了资源所有者密码（password）授权类型以外，支持其余所有标准授权类型的（RFC6749），我们来看一下这个配置对象有哪些属性可以设置吧，如下列表：

- authenticationManager：认证管理器，当你选择了资源所有者密码（password）授权类型的时候，请设置这个属性注入一个 AuthenticationManager 对象。
- userDetailsService：如果啊，你设置了这个属性的话，那说明你有一个自己的 UserDetailsService 接口的实现，或者你可以把这个东西设置到全局域上面去（例如 GlobalAuthenticationManagerConfigurer 这个配置对象），当你设置了这个之后，那么 "refresh_token" 即刷新令牌授权类型模式的流程中就会包含一个检查，用来确保这个账号是否仍然有效，假如说你禁用了这个账户的话。
- authorizationCodeServices：这个属性是用来设置授权码服务的（即 AuthorizationCodeServices 的实例对象），主要用于 "authorization_code" 授权码类型模式。
- implicitGrantService：这个属性用于设置隐式授权模式，用来管理隐式授权模式的状态。
- tokenGranter：这个属性就很牛B了，当你设置了这个东西（即 TokenGranter 接口实现），那么授权将会交由你来完全掌控，并且会忽略掉上面的这几个属性，这个属性一般是用作拓展用途的，即标准的四种授权模式已经满足不了你的需求的时候，才会考虑使用这个。

#### 配置授权端点的URL
AuthorizationServerEndpointsConfigurer 这个配置对象(AuthorizationServerConfigurer 的一个回调配置项，见上的概述) 有一个叫做 pathMapping() 的方法用来配置端点URL链接，它有两个参数：

- 第一个参数：String 类型的，这个端点URL的默认链接。
- 第二个参数：String 类型的，你要进行替代的URL链接。

以上的参数都将以 "/" 字符为开始的字符串，框架的默认URL链接如下列表，可以作为这个 pathMapping() 方法的第一个参数：

- /oauth/authorize：授权端点。
- /oauth/token：令牌端点。
- /oauth/confirm_access：用户确认授权提交端点。
- /oauth/error：授权服务错误信息端点。
- /oauth/check_token：用于资源服务访问的令牌解析端点。

### 资源服务配置
一个资源服务（可以和授权服务在同一个应用中，当然也可以分离开成为两个不同的应用程序）提供一些受token令牌保护的资源，Spring OAuth提供者是通过Spring Security authentication filter 即验证过滤器来实现的保护，你可以通过 @EnableResourceServer 注解到一个 @Configuration 配置类上，并且必须使用 ResourceServerConfigurer 这个配置对象来进行配置（可以选择继承自 ResourceServerConfigurerAdapter 然后覆写其中的方法，参数就是这个对象的实例），下面是一些可以配置的属性：

- tokenServices：ResourceServerTokenServices 类的实例，用来实现令牌服务。
- resourceId：这个资源服务的ID，这个属性是可选的，但是推荐设置并在授权服务中进行验证。
- 其他的拓展属性例如 tokenExtractor 令牌提取器用来提取请求中的令牌。
- 请求匹配器，用来设置需要进行保护的资源路径，默认的情况下是受保护资源服务的全部路径。
- 受保护资源的访问规则，默认的规则是简单的身份验证（plain authenticated）。
- 其他的自定义权限保护规则通过 HttpSecurity 来进行配置。