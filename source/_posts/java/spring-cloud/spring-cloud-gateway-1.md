title: ApiGateway整合Swagger2
author: Figthing
tags:
  - java
  - spring-cloud
categories:
  - java
  - spring-cloud
date: 2019-10-09 18:15:00
---
### ApiGateway整合Swagger2

#### 概述
最近在项目中尝试使用Spring Cloud.Greenwich版整合Swagger2。发现Swagger并不支持以WebFlux为底层的Gateway，无法集成，运行报错。下面分享我的解决思路，和关键代码。

#### 引入依赖
```xml
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-gateway</artifactId>
		</dependency>

		<!-- Swagger2 -->
		<dependency>
			<groupId>io.springfox</groupId>
			<artifactId>springfox-swagger2</artifactId>
			<version>2.9.2</version>
		</dependency>

		<dependency>
			<groupId>io.springfox</groupId>
			<artifactId>springfox-swagger-ui</artifactId>
			<version>2.9.2</version>
		</dependency>	
```

<!--more-->

#### 核心代码
配置SwaggerProvider，获取Api-doc，即SwaggerResources。`SwaggerProvider.java`

```java
@Component
@Primary
@AllArgsConstructor
public class SwaggerProvider implements SwaggerResourcesProvider {

	public static final String API_URI = "/v2/api-docs";
	private final RouteLocator routeLocator;
	private final GatewayProperties gatewayProperties;

	@Override
	public List<SwaggerResource> get() {
		List<SwaggerResource> resources = new ArrayList<>();
		List<String> routes = new ArrayList<>();
		//取出gateway的route
		routeLocator.getRoutes().subscribe(route -> routes.add(route.getId()));
		//结合配置的route-路径(Path)，和route过滤，只获取有效的route节点
		gatewayProperties.getRoutes()
				.stream()
				.filter(routeDefinition -> routes.contains(routeDefinition.getId()))
				.forEach(routeDefinition -> routeDefinition.getPredicates().stream()
						.filter(predicateDefinition -> ("Path").equalsIgnoreCase(predicateDefinition.getName()))
						.forEach(predicateDefinition -> {
							resources.add(swaggerResource(
									routeDefinition.getId(),
									predicateDefinition.getArgs().get(NameUtils.GENERATED_NAME_PREFIX + "0").replace("/**", API_URI)
							));
						}));
		return resources;
	}

	private SwaggerResource swaggerResource(String name, String location) {
		SwaggerResource swaggerResource = new SwaggerResource();
		swaggerResource.setName(name);
		swaggerResource.setLocation(location);
		swaggerResource.setSwaggerVersion("1.0");
		return swaggerResource;
	}
}
```

因为Gateway里没有配置SwaggerConfig，而运行Swagger-ui又需要依赖一些接口，所以我的想法是自己建立相应的swagger-resource端点`SwaggerHandler.java`

```java
@RestController
@ConditionalOnExpression("#{'true'.equals(environment['neusoft.hype.swagger.enable'])}")
@RequestMapping("/swagger-resources")
public class SwaggerHandler {

	@Autowired(required = false)
	private SecurityConfiguration securityConfiguration;

	@Autowired(required = false)
	private UiConfiguration uiConfiguration;

	private final SwaggerResourcesProvider swaggerResources;

	@Autowired
	public SwaggerHandler(SwaggerResourcesProvider swaggerResources) {
		this.swaggerResources = swaggerResources;
	}

	@GetMapping("/configuration/security")
	public Mono<ResponseEntity<SecurityConfiguration>> securityConfiguration() {
		return Mono.just(new ResponseEntity<>(
				Optional.ofNullable(securityConfiguration).orElse(SecurityConfigurationBuilder.builder().build()), HttpStatus.OK));
	}

	@GetMapping("/configuration/ui")
	public Mono<ResponseEntity<UiConfiguration>> uiConfiguration() {
		return Mono.just(new ResponseEntity<>(
				Optional.ofNullable(uiConfiguration).orElse(UiConfigurationBuilder.builder().build()), HttpStatus.OK));
	}

	@GetMapping("")
	public Mono<ResponseEntity> swaggerResources() {
		return Mono.just((new ResponseEntity<>(swaggerResources.get(), HttpStatus.OK)));
	}
}
```