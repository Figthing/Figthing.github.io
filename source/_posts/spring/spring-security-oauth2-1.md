title: Spring Security + OAuth2（一）
author: Figthing
tags:
  - java
  - spring
  - security
  - oauth2
categories:
  - java
  - spring
date: 2019-05-04 13:35:00
---
### 前言

经过前段时间对SSO单点登录的研究发现，有以下几类方案可以实现

1. Spring Session + Redis
2. Spring Security + JWT
3. Spring Security + OAuth2 + Redis

本文章将讲述`Spring Security + OAuth2 + Redis`，其他方式不做更多说明，直接上干货，下面是大概的流程。

![](https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/spring/security%2Boauth2/Spring%20Security%20%2B%20OAuth2.png)

<!-- more -->

### 目录结构
```lua
neusoft-sso-login 授权服务
├── AuthApplication
└── security
      ├── WebSecurityConfigurer
      ├── WebResponseException
      └── auth
            ├── AuthFailHandler
            ├── AuthRequestFilter 
            ├── AuthServerConfigurerAdapter
            └── AuthSuccessHandler
      └── service
            ├── ClientDetailsServiceImpl
            └── UserDetailsServiceImpl
```

### SQL脚本
```sql
CREATE TABLE `neusoft_sr_oper` (
  `OPER_ID` varchar(64) NOT NULL COMMENT '用户ID',
  `OPER_ACCOUNT` varchar(20) DEFAULT NULL COMMENT '账号',
  `OPER_PWD` varchar(128) DEFAULT NULL COMMENT '密码',
  `OPER_NAME` varchar(64) DEFAULT NULL COMMENT '账号名称',
  `CREATOR_ID` varchar(64) DEFAULT '0' COMMENT '创建者ID（0-注册用户，super-超级管理员，其他-子用户）',
  `LAST_LOGIN_IP` varchar(20) DEFAULT NULL COMMENT '最后登录IP',
  `LAST_LOGIN_TIME` datetime DEFAULT NULL COMMENT '最后登录时间',
  `LOGIN_COUNT` int(11) DEFAULT '0' COMMENT '登录次数',
  `STATUS` varchar(2) DEFAULT '0' COMMENT '0-正常，1-停用，2-锁定，99-注销（删除）',
  `PWD_MODIFY_DATE` datetime DEFAULT NULL COMMENT '密码更新时间',
  `STATUS_MODIFY_DATE` datetime DEFAULT NULL COMMENT '状态更新时间',
  `CREATE_TIME` datetime DEFAULT NULL COMMENT '创建时间',
  `UPDATE_TIME` datetime DEFAULT NULL COMMENT '更新时间',
  PRIMARY KEY (`OPER_ID`),
  UNIQUE KEY `OPER_ACCOUNT` (`OPER_ACCOUNT`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='用户基础信息表';

CREATE TABLE `neusoft_sys_oauth_client` (
  `CLIENT_ID` varchar(64) NOT NULL COMMENT '客户端ID',
  `CLIENT_NAME` varchar(32) DEFAULT NULL COMMENT '客户端名称',
  `CLIENT_SECRET` varchar(128) DEFAULT NULL COMMENT '客户端密匙',
  `CLIENT_SECRET_PLAIN` varchar(128) DEFAULT NULL COMMENT '客户端密匙明码',
  `SCOPE` varchar(32) DEFAULT NULL COMMENT '限定范围（read，write）',
  `AUTHORIZED_GRANT_TYPES` varchar(32) DEFAULT NULL COMMENT '授权类型（client_credentials,password,refresh_token）',
  `CREATE_TIME` datetime DEFAULT NULL COMMENT '创建时间',
  `UPDATE_TIME` datetime DEFAULT NULL COMMENT '更新时间',
  PRIMARY KEY (`CLIENT_ID`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='OAuth客户端';
```

### 自定义和服务实现
对Spring Security + OAuth2，自定义和服务实现主要是以下几个类
- WebSecurityConfigurerAdapter，授权拦截器配置
- WebResponseExceptionTranslator，异常捕获处理
- UsernamePasswordAuthenticationFilter，请求拦截器
- UserDetailsService，获取用户信息
- ClientDetailsService，获取客户端信息
- AuthenticationSuccessHandler，认证成功处理
- AuthenticationFailureHandler，认证失败处理
- AuthorizationServerConfigurerAdapter 认证服务配置

### CODING
#### AuthApplication.java
启动文件，因我使用的是Redis Cluster集群模式，所以要把自带的Redis加载去掉
```java
@EnableEurekaClient
@SpringBootApplication(exclude = {RedisAutoConfiguration.class})
public class AuthApplication {

	public static void main(String[] args) {
		SpringApplication.run(AuthApplication.class);
	}
}
```

pom.xml
```xml   
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-security</artifactId>
      <exclusions>
        <!--旧版本 redis操作有问题-->
        <exclusion>
          <artifactId>spring-security-oauth2</artifactId>
          <groupId>org.springframework.security.oauth</groupId>
        </exclusion>
      </exclusions>
    </dependency>
    <dependency>
      <groupId>org.springframework.security.oauth</groupId>
      <artifactId>spring-security-oauth2</artifactId>
      <version>2.3.5.RELEASE</version>
    </dependency>

    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
```

#### WebSecurityConfigurer.java
自定义`WebSecurityConfigurerAdapter`，过滤`MATCHER_URL`路径为不用鉴权，登录地址设置为`LOGIN_URL`，在`UsernamePasswordAuthenticationFilter`拦截器前面新增`AuthRequestFilter`，设定认证密码模式为`BCryptPasswordEncoder`
```java
@Configuration
public class WebSecurityConfigurer extends WebSecurityConfigurerAdapter {

	@Autowired
	private AuthFailHandler authFailHandler;

	@Autowired
	private AuthSuccessHandler authSuccessHandler;

	private final static String LOGIN_URL = "/sso/login";
	private final static String[] MATCHER_URL = {"/info", "/sso/*"};

	@Override
	protected void configure(HttpSecurity http) throws Exception {

		http
				.authorizeRequests()
				.antMatchers(MATCHER_URL).permitAll()
				.anyRequest().authenticated()
				.and().csrf().disable();

		http.addFilterAt(authRequestFilter(), UsernamePasswordAuthenticationFilter.class);
	}

	@Bean
	public AuthRequestFilter authRequestFilter() {
		AuthRequestFilter authRequestFilter = new AuthRequestFilter();
		authRequestFilter.setAuthenticationManager(authenticationManagerBean());
		authRequestFilter.setRequiresAuthenticationRequestMatcher(new AntPathRequestMatcher(LOGIN_URL, HttpMethod.POST.toString()));
		authRequestFilter.setAuthenticationFailureHandler(authFailHandler);
		authRequestFilter.setAuthenticationSuccessHandler(authSuccessHandler);
		return authRequestFilter;
	}

	@Bean
	@Override
	@SneakyThrows
	public AuthenticationManager authenticationManagerBean() {
		return super.authenticationManagerBean();
	}

	@Bean
	public PasswordEncoder passwordEncoder() {
		return new BCryptPasswordEncoder();
	}
}
```

#### AuthRequestFilter.java
自定义`UsernamePasswordAuthenticationFilter`，将默认的Form模式变更为JSON模式
```java
public class AuthRequestFilter extends UsernamePasswordAuthenticationFilter {

	private final static Logger LOGGER = LoggerFactory.getLogger(AuthRequestFilter.class);
	private final static String ACCOUNT_STR = "account";
	private final static String PASSWORD_STR = "password";

	@Override
	public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException {
		if (!request.getContentType().equals(MediaType.APPLICATION_JSON_UTF8_VALUE)
				&& !request.getContentType().equals(MediaType.APPLICATION_JSON_VALUE)) {
			return super.attemptAuthentication(request, response);
		}

		ObjectMapper mapper = new ObjectMapper();
		UsernamePasswordAuthenticationToken authRequest = null;

		try (InputStream is = request.getInputStream()) {
			Map authenticationBean = mapper.readValue(is, Map.class);
			authRequest = new UsernamePasswordAuthenticationToken(authenticationBean.get(ACCOUNT_STR), authenticationBean.get(PASSWORD_STR));
		} catch (IOException e) {
			LOGGER.error("JsonAuthenticationFilter form to json error!");
			authRequest = new UsernamePasswordAuthenticationToken("", "");
		}

		setDetails(request, authRequest);
		return this.getAuthenticationManager().authenticate(authRequest);
	}
}
```

#### AuthFailHandler.java
自定义`AuthenticationFailureHandler`,根据不同的异常信息进行返回处理
```java
@Component
public class AuthFailHandler implements AuthenticationFailureHandler {

	@Autowired
	private ObjectMapper objectMapper;

	@Override
	public void onAuthenticationFailure(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, AuthenticationException e) throws IOException, ServletException {

		String code = null;

		// 锁
		if (e instanceof LockedException) {
			code = ERROR_AUTH_SSO_LOGIN_LOCK;
		}

		// 停用
		if (e instanceof DisabledException) {
			code = ERROR_AUTH_SSO_LOGIN_EXPIRED;
		}

		// 账号密码错误
		if (e instanceof BadCredentialsException) {
			code = ERROR_AUTH_SSO_LOGIN;
		}

		if (code == null) {
			return;
		}

		httpServletResponse.setStatus(HttpStatus.OK.value());
		httpServletResponse.setContentType(MediaType.APPLICATION_JSON_UTF8_VALUE);
		httpServletResponse.getWriter().write(objectMapper.writeValueAsString(ResponseUtil.fail(code)));
	}
}
```

#### AuthSuccessHandler.java
自定义`AuthenticationSuccessHandler`，读取header中的AUTHORIZATION，校验客户端secret，校验客户端scope，生成Token并自动存储到Redis中
```java
@Component
public class AuthSuccessHandler implements AuthenticationSuccessHandler {

	@Autowired
	private ObjectMapper objectMapper;

	@Autowired
	private PasswordEncoder passwordEncoder;

	@Autowired
	private ClientDetailsService clientDetailsService;

	@Autowired
	private AuthorizationServerTokenServices defaultAuthorizationServerTokenServices;

	private final static Logger LOGGER = LoggerFactory.getLogger(AuthSuccessHandler.class);

	private static final String BASIC_ = "Basic ";

	@Override
	public void onAuthenticationSuccess(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Authentication authentication) throws IOException {

		String header = httpServletRequest.getHeader(HttpHeaders.AUTHORIZATION);

		if (header == null || !header.startsWith(BASIC_)) {
			HttpUtil.writeJson(httpServletResponse, ResponseUtil.fail(ERROR_AUTH_BASIC_CLIENT), objectMapper);
			return;
		}

		String[] tokens;

		try {
			tokens = extractAndDecodeHeader(header);
		} catch (Exception e) {
			HttpUtil.writeJson(httpServletResponse, ResponseUtil.fail(ERROR_AUTH_BASIC_CLIENT), objectMapper);
			return;
		}

		assert tokens.length == 2;
		String clientId = tokens[0];

		ClientDetails clientDetails = clientDetailsService.loadClientByClientId(clientId);

		//校验secret
		if (!passwordEncoder.matches(tokens[1], clientDetails.getClientSecret())) {
			HttpUtil.writeJson(httpServletResponse, ResponseUtil.fail(ERROR_AUTH_CLIENT), objectMapper);
			return;
		}

		TokenRequest tokenRequest = new TokenRequest(Maps.newConcurrentMap(), clientId, clientDetails.getScope(), "all");

		//校验scope
		new DefaultOAuth2RequestValidator().validateScope(tokenRequest, clientDetails);
		OAuth2Request oAuth2Request = tokenRequest.createOAuth2Request(clientDetails);
		OAuth2Authentication oAuth2Authentication = new OAuth2Authentication(oAuth2Request, authentication);
		OAuth2AccessToken oAuth2AccessToken = defaultAuthorizationServerTokenServices.createAccessToken(oAuth2Authentication);

		LOGGER.info("token builder [{}]", oAuth2AccessToken.getValue());

		Map<String, String> result = Maps.newConcurrentMap();
		result.put(HttpHeaders.ACCESS_TOKEN, oAuth2AccessToken.getValue());
		HttpUtil.writeJson(httpServletResponse, ResponseUtil.success(result), objectMapper);
	}

	@SneakyThrows
	private String[] extractAndDecodeHeader(String header) {

		byte[] base64Token = header.substring(6).getBytes(UTF_8);
		byte[] decoded;
		try {
			decoded = Base64.getDecoder().decode(base64Token);
		} catch (IllegalArgumentException e) {
			throw new RuntimeException(
					"Failed to decode basic authentication token");
		}

		String token = new String(decoded, UTF_8);

		int delim = token.indexOf(":");

		if (delim == -1) {
			throw new RuntimeException("Invalid basic authentication token");
		}

		return new String[]{token.substring(0, delim), token.substring(delim + 1)};
	}
}
```

#### ClientDetailsServiceImpl.java
接口`ClientDetailsService`实现类，获取客户端信息，并生成`ClientDetails`
```java
@Service
public class ClientDetailsServiceImpl implements ClientDetailsService {

	@Autowired
	private OAuthClientService clientService;

	private final static String SCOPE_LIMIT = ",";

	@Override
	public ClientDetails loadClientByClientId(String s) throws ClientRegistrationException {

		OAuthClient client = clientService.clientByName(s);

		if (null == client) {
			return new BaseClientDetails();
		}

		BaseClientDetails clientDetails = new BaseClientDetails();
		clientDetails.setClientId(client.getClientName());
		clientDetails.setClientSecret(client.getSecret());
		clientDetails.setScope(Arrays.asList(client.getScope().split(SCOPE_LIMIT)));
		clientDetails.setAuthorizedGrantTypes(Arrays.asList(client.getAuthorizedGrantTypes().split(SCOPE_LIMIT)));
		return clientDetails;
	}
}
```

#### UserDetailsServiceImpl.java
接口`UserDetailsService`实现类，获取用户信息，并生成`UserDetails`
```java
@Service
public class UserDetailsServiceImpl implements UserDetailsService {

	@Autowired
	private AuthService authService;

	@Override
	public UserDetails loadUserByUsername(String account) throws UsernameNotFoundException {
		if (Strings.isNullOrEmpty(account)) {
			throw new UsernameNotFoundException(ResponseCodeSso.ERROR_AUTH_SSO_LOGIN_ACCOUNT);
		}

		if (!authService.accountExtis(account)) {
			throw new UsernameNotFoundException(ResponseCodeSso.ERROR_AUTH_SSO_LOGIN_ACCOUNT);
		}

		return getUserDetails(account);
	}

	private UserDetails getUserDetails(String account) {

		Oper oper = authService.infoByWhere(account);

		if (null == oper) {
			throw new UsernameNotFoundException(ResponseCodeSso.ERROR_AUTH_SSO_LOGIN_ACCOUNT);
		}

		// 获取角色，未进行处理
		Collection<? extends GrantedAuthority> authorities = AuthorityUtils.createAuthorityList("ROLE_USER");

		User user = new User(
				oper.getOperAccont(),
				oper.getOperPwd(),
				oper.getStatus().startsWith("0"),     // 是否正常
				!oper.getStatus().startsWith("1"),    // 是否停用
				true,
				!oper.getStatus().startsWith("2"),    // 是否锁定
				authorities
		);
		return user;
	}
}
```

#### AuthServerConfigurerAdapter.java
Token存储模式设置为Redis Cluster存储，将`Lettuce`替代`Jedis`，重定义`/oauth/check_token`路径为`/sso/check`，嵌入自定义授权异常
```java
@Configuration
public class AuthServerConfigurerAdapter {

	@Autowired
	private RedisCacheConfig redisCacheConfig;

	@Autowired
	private CacheConfig cacheConfig;

	@Bean
	public LettuceConnectionFactory lettuceConnectionFactory() {
		RedisClusterConfig clusterConfig = new RedisClusterConfig(redisCacheConfig);
		RedisClusterConfiguration clusterConfiguration = new RedisClusterConfiguration(clusterConfig.getClusterNodes());
		return new LettuceConnectionFactory(clusterConfiguration);
	}

	@Bean(name = "redisTemplate")
	public RedisTemplate<String, Serializable> redisCacheTemplate() {
		RedisTemplate<String, Serializable> template = new RedisTemplate<>();
		template.setKeySerializer(new StringRedisSerializer());
		template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
		template.setConnectionFactory(lettuceConnectionFactory());
		return template;
	}

	@Bean
	public TokenStore tokenStore() {
		RedisTokenStore tokenStore = new RedisTokenStore(redisCacheTemplate().getConnectionFactory());
		tokenStore.setPrefix(cacheConfig.getRoot() + ":Token:");
		return tokenStore;
	}

	@Bean
	public TokenEnhancer tokenEnhancer() {
		return (accessToken, authentication) -> {
			final Map<String, Object> additionalInfo = new HashMap<>(1);
			additionalInfo.put("license", "web");
			((DefaultOAuth2AccessToken) accessToken).setAdditionalInformation(additionalInfo);
			return accessToken;
		};
	}

	@EnableAuthorizationServer
	protected static class AuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {

		@Autowired
		private AuthenticationManager authenticationManager;

		@Autowired
		private UserDetailsService userDetailsService;

		@Autowired
		private ClientDetailsServiceImpl clientDetailsService;

		@Autowired
		private TokenStore tokenStore;

		@Autowired
		private TokenEnhancer tokenEnhancer;

		@Autowired
		private WebResponseExceptionTranslator webResponseException;

		@Override
		public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
			clients.withClientDetails(clientDetailsService);
		}

		@Override
		public void configure(AuthorizationServerSecurityConfigurer oauthServer) throws Exception {
			oauthServer
					.allowFormAuthenticationForClients()
					.checkTokenAccess("permitAll()");
		}

		@Override
		public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
			endpoints
					.allowedTokenEndpointRequestMethods(HttpMethod.GET, HttpMethod.POST)
					.pathMapping("/oauth/check_token", "/sso/check")
					.tokenStore(tokenStore)
					.tokenEnhancer(tokenEnhancer)
					.userDetailsService(userDetailsService)
					.authenticationManager(authenticationManager)
					.reuseRefreshTokens(false)
					.exceptionTranslator(webResponseException);
		}
	}
}
```