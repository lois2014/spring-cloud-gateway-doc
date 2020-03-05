[TOC]

## Spring cloud gateway版本依赖

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
## Eureka配置
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
## 路由下发配置(java版配置)
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
下发路由基础配置:

1. routeId配置，自定义就行
2. uri("lb://example-client"), uri匹配，lb://开头是指负载均衡（loadbalance）的协议，后面是注册到注册中心的服务名称：application name, 如果是特定uri,可以直接配置uri， 如：http://localhost:8081
3. order(5) 是指路由的匹配顺序

断言配置：

1. method(HttpMethod.POST), 方法路由断言（Method Route Predicate），只有post请求可匹配
2. path("/exampleclient/**"), 路径路由断言（Path Route Predicate），路径正则匹配， 相当于/exampleclient/a/b  或者 /exampleclient/aa?a=ww等前缀是/exampleclient的路由
3. remoteAddr("0.0.0.0/0")， 限制ip只有配置的ip可访问，即ip白名单。参数是CIDR-notation，这个是ip的归类方法，相当于ip段，可配置多个，用逗号","隔开，如0.0.0.0/0代表所有ip段

过滤器配置：

1. stripPrefix(1) /exampleclient/example/getInfo/, gateway下发路由就会变成 /example/getInfo，即去掉路由前缀的一个参数，这样配置路由是为了区分多个系统下的下发路由可以相互不影响
2. retry(config -> {}), 当出现某些异常时的重试机制。
    1. 配置参数如下：
        1. retries: 重试次数, 默认值：3
        2. statuses: http status 如 404， 500， 302等，可参考org.springframework.http.HttpStatus 默认值：无
        3. methods: 请求方式，post，get等，可参考org.springframework.http.HttpMethod， 默认值:GET
        4. exceptions: 抛出的异常， 默认值：IOException ， TimeoutException
        5. series: http状态码系列，如 5xx为一类状态码，可参考org.springframework.http.HttpStatus.Series，默认值： 5XX series
        6. backoff: 两个重试发送中间等待配置，如：
           backoff:
              firstBackoff: 10ms //第一次等待时间
              maxBackoff: 50ms //最大等待时机
              factor: 2 // 指数，等待时间指数级增长  
              basedOnPreviousValue: false //是否基于前面等待的时间
6. histrix(config -> {}), 熔断，
    配置如下：
        1. 设置指令名 setName()
        2. 设置降级uri setFallbackUri("forward:/exampleFallback"), 只要熔断就会定向到匹配"/exampleFallback"的controller 也可以使用lb://xxxa使用负载            
        
##自定义全局请求日志过滤器

 1. 自定义全局过滤器
 ```java
    @Component
    public class CustomGlobalFilter implements GlobalFilter, Ordered {

        @Override
        public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
            log.info("custom global filter");
            return chain.filter(exchange);
        }

        @Override
        public int getOrder() {
            return -1;
        }
    }
 ```
 2. 获取请求参数
 ```java
    ServerHttpRequest request = exchange.getRequest();
    Object requestBody = "";
    //post cachedRequestBodyObject
    if (request.getMethod() != null
            && HttpMethod.POST.toString().equals(request.getMethod().toString())) {
        requestBody = exchange.getAttribute("cachedRequestBodyObject");
    }
    String queryString = request.getQueryParams().toString();   
 ```
 3. 获取响应参数
     1. 重新构造response, ServerHttoResonseDecorator
     2. 写入outputStream后要释放原来dataBuffer，不然会造成直接内存溢出问题
     3. 参考自 [Spring Cloud Gateway 内存溢出解决过程](https://blog.csdn.net/live501837145/article/details/99446673)
 ```java
 private ServerHttpResponseDecorator handleResponse(ServerWebExchange exchange) {
        log.debug(" ----- response start  ------");
        ServerHttpResponse response = exchange.getResponse();
        DataBufferFactory bufferFactory = response.bufferFactory();
        ServerHttpResponseDecorator decoratedResponse = new ServerHttpResponseDecorator(response) {
            @Override
            public Mono<Void> writeWith(Publisher<? extends DataBuffer> body) {
                if (body instanceof Flux) {
                    Flux<? extends DataBuffer> fluxBody = (Flux<? extends DataBuffer>) body;
                    return super.writeWith(fluxBody.buffer().map(dataBuffers -> {
                        byte[] content = retrieveBytes(dataBuffers);
                        String responseResult = new String(content, Charset.forName("UTF-8"));
                        log.info("Response:[ Status: {} Header: {} ResponseResult: {}] ",
                                this.getStatusCode(), this.getHeaders().toString(), responseResult);
                        return bufferFactory.wrap(content);
                    }));
                }
                return super.writeWith(body);
            }
        };
        log.debug(" ----- response end  ------");
        return decoratedResponse;
    }

    //FIX OutOfDirectMemory
    public byte[] retrieveBytes(List<? extends DataBuffer> dataBuffers) {
        byte[] retrieveBytes = null;
        try (ByteArrayOutputStream outputStream = new ByteArrayOutputStream()) {
            dataBuffers.forEach(i -> processBuffer(i, outputStream));
            retrieveBytes = outputStream.toByteArray();
        } catch (IOException e) {
            log.error("IOException closing ByteArrayOutputStream", e);
        } catch (Exception e) {
            log.error("TaskException processing DataBuffer", e);
        }
        return retrieveBytes;
    }

    private void processBuffer(DataBuffer dataBuffer, ByteArrayOutputStream outputStream) throws RuntimeException {
        int count;
        while ((count = dataBuffer.readableByteCount()) > 0) {
            byte[] array = new byte[count];
            dataBuffer.read(array);
            try {
                outputStream.write(array);
            } catch (IOException e) {
                DataBufferUtils.release(dataBuffer);
                throw new RuntimeException(e);
            }
        }
        DataBufferUtils.release(dataBuffer);
    }    
 ```
完整代码
 ```java

    @Slf4j
    @Component
    public class AccessLogGlobalFilter implements GlobalFilter, Ordered {

        @Override
        public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {

            log.info("init memory {}", getDirectMemory());
            long startTime = new Date().getTime();
            ServerHttpRequest request = exchange.getRequest();
            Object requestBody = "";
            //post cachedRequestBodyObject
            if (request.getMethod() != null
                    && HttpMethod.POST.toString().equals(request.getMethod().toString())) {
                requestBody = exchange.getAttribute("cachedRequestBodyObject");
            }

            Gson gson = new GsonBuilder().serializeNulls().create();
            log.info("Request[ RemoteAddress: {}, Method: {}, URL: {}, Params: {}, RequestBody: {} ]",
                    getIpAddress(request),
                    request.getMethod() == null ? "" : request.getMethod().toString(),
                    request.getURI().getRawPath(),
                    request.getQueryParams().toString(),
                    gson.toJson(requestBody));
            ServerHttpResponseDecorator responseDecorator = this.handleResponse(exchange, startTime);
            return chain.filter(exchange.mutate().response(responseDecorator).build());
        }

        @Override
        public int getOrder() {
            return -2;
        }

        public static class Config {
            //Put the configuration properties for your filter here
        }

        private ServerHttpResponseDecorator handleResponse(ServerWebExchange exchange, long startTime) {
            log.debug(" ----- response start  ------");
            long endTime = new Date().getTime();
            ServerHttpResponse response = exchange.getResponse();
            DataBufferFactory bufferFactory = response.bufferFactory();
            ServerHttpResponseDecorator decoratedResponse = new ServerHttpResponseDecorator(response) {
                @Override
                public Mono<Void> writeWith(Publisher<? extends DataBuffer> body) {
                    if (body instanceof Flux) {
                        Flux<? extends DataBuffer> fluxBody = (Flux<? extends DataBuffer>) body;
                        return super.writeWith(fluxBody.buffer().map(dataBuffers -> {
                            byte[] content = retrieveBytes(dataBuffers);
                            String responseResult = new String(content, Charset.forName("UTF-8"));
                            log.info("Response:[ Status: {} Header: {} ResponseResult: {} Time: {} ms] ",
                                    this.getStatusCode(), this.getHeaders().toString(), responseResult, (endTime - startTime));
                            return bufferFactory.wrap(content);
                        }));
                    }
                    return super.writeWith(body);
                }
            };
            log.debug(" ----- response end  ------");
            return decoratedResponse;
        }

        public static String getIpAddress(ServerHttpRequest request) {
            HttpHeaders headers = request.getHeaders();
            String ip = headers.getFirst("x-forwarded-for");
            if (ip != null && ip.length() != 0 && !"unknown".equalsIgnoreCase(ip)) {
                // 多次反向代理后会有多个ip值，第一个ip才是真实ip
                if (ip.contains(",")) {
                    ip = ip.split(",")[0];
                }
            }
            if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
                ip = headers.getFirst("Proxy-Client-IP");
            }
            if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
                ip = headers.getFirst("WL-Proxy-Client-IP");
            }
            if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
                ip = headers.getFirst("HTTP_CLIENT_IP");
            }
            if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
                ip = headers.getFirst("HTTP_X_FORWARDED_FOR");
            }
            if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
                ip = headers.getFirst("X-Real-IP");
            }
            if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
                ip = request.getRemoteAddress() == null
                        ? "" : request.getRemoteAddress().getAddress().getHostAddress();
            }
            return ip == null ? "" : ip;
        }

        private long getDirectMemory() {
            Field field = ReflectionUtils.findField(PlatformDependent.class, "DIRECT_MEMORY_COUNTER");
            field.setAccessible(true);
            try {
                    AtomicLong atomicLong = (AtomicLong) field.get(PlatformDependent.class);
                    return atomicLong.get() / 1024;
                } catch (Exception e) {
                    return 0;
                }
            }

            //FIX OutOfDirectMemory
            public byte[] retrieveBytes(List<? extends DataBuffer> dataBuffers) {
                byte[] retrieveBytes = null;
                try (ByteArrayOutputStream outputStream = new ByteArrayOutputStream()) {
                    dataBuffers.forEach(i -> processBuffer(i, outputStream));
                    retrieveBytes = outputStream.toByteArray();
                } catch (IOException e) {
                    log.error("IOException closing ByteArrayOutputStream", e);
                } catch (Exception e) {
                    log.error("TaskException processing DataBuffer", e);
                }
                return retrieveBytes;
            }

            private void processBuffer(DataBuffer dataBuffer, ByteArrayOutputStream outputStream) throws RuntimeException {
                int count;
                while ((count = dataBuffer.readableByteCount()) > 0) {
                    byte[] array = new byte[count];
                    dataBuffer.read(array);
                    try {
                        outputStream.write(array);
                    } catch (IOException e) {
                        DataBufferUtils.release(dataBuffer);
                        throw new RuntimeException(e);
                    }
                }
                DataBufferUtils.release(dataBuffer);

            }
 ```        




