[TOC]

## Spring cloud gateway搭建版本依赖

spring boot版本
```xml
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.7.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
```
spring cloud版本
```xml
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>Greenwich.SR4</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
```
spring cloud gateway版本
```xml
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>
```
spring cloud eureka版本
```xml
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        <version>2.0.2.RELEASE</version>
    </dependency>
```

spring cloud hystrix版本
```xml
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
    </dependency>
```

spring boot webflux版本
```xml
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webflux</artifactId>
    </dependency>
```
spring 仓库
```xml 
    <repositories>
        <repository>
            <id>spring-milestones</id>
            <name>Spring Milestones</name>
            <url>https://repo.spring.io/milestone</url>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </repository>
    </repositories>
```
## 配置
### Eureka配置
```yml
    eureka:
      client:
        serviceUrl:
          defaultZone: http://${peer1.host}:${peer1.port}/eureka/, http://${peer2.host}:${peer2.port}/eureka/
        enabled: true
      instance:
        prefer-ip-address: true
```
解析：
    1 serviceUrl.defaultZone -- 注册中心地址
    2 instance.prefer-ip-address=true -- 用当前ip注册，默认是hostname注册，原因是无法通过hostname访问其他机器，用ip注册可以通过内网访问其他机器
### 路由转发配置(java版配置)
```java
@Bean
public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
            return builder.routes()
                .route(routeId, r ->
                                r.remoteAddr("0.0.0.0/0")
                                        .and().method(HttpMethod.POST).and()
                                        .readBody(Object.class, readBody -> {
                                            return true;
                                        })
                                        .and()
                                        .path("/exampleclient/**")
                                        .filters(f -> f
                                                .hystrix(config -> {
                                                    config.setFallbackUri("forward:/fallback/serviceFailurePage").setName("fallbackCmd");
                                                })
                                                .retry(retryConfig -> {
                                                    retryConfig.setExceptions(java.io.IOException.class, java.util.concurrent.TimeoutException.class)
                                                            .setRetries(3)
                                                            .setSeries(SERVER_ERROR, CLIENT_ERROR);
                                                })
                                                .stripPrefix(1)
                                        )
                                        .uri("lb://example-client")
                )
                .route(routeId, r ->
                                r.remoteAddr("0.0.0.0/0")
                                        .and()
                                        .path("/exampleclient/**")
                                        .filters(f -> f
                                                .hystrix(config -> {
                                                    config.setFallbackUri("forward:/fallback/serviceFailurePage").setName("fallbackCmd");
                                                })
                                                .retry(retryConfig -> {
                                                    retryConfig.setExceptions(java.io.IOException.class, java.util.concurrent.TimeoutException.class)
                                                            .setRetries(3)
                                                            .setSeries(SERVER_ERROR, CLIENT_ERROR);
                                                })
                                                .stripPrefix(1)
                                        )
                                        .uri("lb://example-client")
                                        .order(5)
                )
                .build();
}
```
配置简单转发:

```java
        builder.routes()
                .route(routeId, r ->
                                r.method(HttpMethod.POST)                                 
                                 .and()
                                 .path("/exampleclient/**")
                                 .filters(f -> f
                                   .stripPrefix(1)
                                 )
                                 .uri("lb://example-client")
                ).build();
```

1. method(HttpMethod.POST), 方法路由断言（Method Route Predicate），只有post请求可匹配
2. path("/exampleclient/**"), 路径路由断言（Path Route Predicate），路径正则匹配， 相当于/exampleclient/a/b  或者 /exampleclient/aa?a=ww等前缀是/exampleclient的路由
3. uri("lb://example-client"), uri匹配，lb://开头是指负载均衡（loadbalance）的协议，后面是注册到注册中心的服务名称：application name, 如果是特定uri,可以直接配置uri， 如：http://localhost:8081


