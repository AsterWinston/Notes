# 1. 知识点
- **Feign**：Netflix开源项目，声明式HTTP客户端，通过接口注解方式调用HTTP API，已停更进入维护模式。
- **OpenFeign**：Spring Cloud社区从Feign分支继续维护发展的版本，增加了Spring MVC注解支持、请求/响应压缩、负载均衡集成等特性。
- **声明式HTTP调用**：通过`@FeignClient`注解标记接口，无需编写HTTP请求代码，OpenFeign自动生成动态代理，像调用本地方法一样调用远程服务。
- **@FeignClient注解**：核心注解，包含属性`name`（服务名）、`url`（直连地址）、`path`（路径前缀）、`fallback`（降级类）、`configuration`（自定义配置类）。
- **动态代理生成**：`FeignClientFactoryBean`实现了`FactoryBean`接口，通过JDK动态代理为`@FeignClient`接口生成代理对象，拦截方法调用转为HTTP请求。
- **BeanDefinition注册**：`@EnableFeignClients`触发包扫描，将`@FeignClient`接口的BeanDefinition注册到Spring容器，每个接口对应一个`FeignClientFactoryBean`。
- **核心执行流程**：接口方法调用 → 动态代理拦截 → Contract解析方法元数据 → 生成RequestTemplate → RequestInterceptor拦截器链 → Encoder编码请求体 → Client执行HTTP → Decoder解码响应。
- **请求拦截器**：实现`RequestInterceptor`接口，在请求发送前修改RequestTemplate，常用于添加认证Token、TraceId、自定义Header等。
- **拦截器应用场景**：统一添加Authorization认证头、透传TraceId实现链路追踪、添加版本号/环境标签、多线程上下文传递（通过RequestContextHolder）。
- **Encoder编码器**：将Java对象序列化为HTTP请求体（JSON/XML等），Spring集成默认使用SpringEncoder（基于HttpMessageConverters）。
- **Decoder解码器**：将HTTP响应体反序列化为Java对象，Spring集成默认使用SpringDecoder（基于HttpMessageConverters）。
- **Jackson vs FastJson**：OpenFeign默认使用Jackson作为JSON序列化工具，也可通过自定义Encoder/Decoder替换为FastJson或其他序列化框架。
- **SpringDecoder原理**：内部使用`HttpMessageConverters`列表，根据响应的Content-Type选择合适的`HttpMessageConverter`进行反序列化。
- **日志级别**：NONE（不记录）、BASIC（记录请求方法、URL、响应状态码、执行时间）、HEADERS（BASIC + 请求和响应头信息）、FULL（HEADERS + 请求体和响应体）。
- **超时配置**：`feign.client.config.default.connectTimeout`（连接超时，默认10秒）、`feign.client.config.default.readTimeout`（读超时，默认60秒）。
- **按服务名超时配置**：`feign.client.config.{服务名}.connectTimeout`可针对特定服务单独配置超时时间。
- **HttpClient5超时配置**：当底层使用Apache HttpClient5时，超时优先取HttpClient的连接池超时配置，Feign的配置可能不生效。
- **ErrorDecoder**：自定义错误解码器，实现`ErrorDecoder`接口，根据HTTP状态码解析不同异常，实现细粒度的错误处理。
- **FallbackFactory**：实现`FallbackFactory<T>`接口，可以获取异常信息，在降级逻辑中记录降级原因，比直接写Fallback类更灵活。
- **Gzip压缩**：配置`feign.compression.request.enabled=true`和`feign.compression.response.enabled=true`，减少传输数据量，可设置压缩阈值和MIME类型。
- **负载均衡集成**：Feign默认集成Spring Cloud LoadBalancer（已替代Ribbon），通过服务名调用时自动实现负载均衡。
- **Feign与LoadBalancer整合原理**：创建Feign Client时生成`LoadBalancerCommand`，在请求时通过LoadBalancer获取实例，再执行HTTP调用。
- **HttpClient连接池**：默认使用JVM的URLConnection（无连接池），生产环境建议替换为Apache HttpClient5或OkHttp，支持连接池复用。
- **OkHttp vs Apache HttpClient5**：OkHttp默认支持HTTP/2，连接池管理简单；Apache HttpClient5功能更全面，但配置相对复杂。
- **连接池KeepAlive**：通过设置KeepAlive策略保持长连接，减少TCP握手开销，需配合服务端KeepAlive超时配置。
- **Sentinel整合**：配置`feign.sentinel.enabled=true`开启Sentinel集成，Feign接口自动成为Sentinel资源，支持流量控制和熔断降级。
- **Fallback与Sentinel Fallback**：OpenFeign的fallback和Sentinel的fallback可并存；Feign fallback处理降级逻辑，Sentinel fallback可记录降级原因。
- **Contract契约接口**：定义如何解析接口上的注解，OpenFeign扩展了SpringMvcContract，支持Spring MVC注解（@GetMapping、@PostMapping、@RequestParam等）。
- **MethodMetadata**：每个接口方法解析后生成一个MethodMetadata对象，包含请求方法、URL模板、参数列表、Header等元数据。
- **RequestTemplate**：将MethodMetadata和实际参数值合并生成的可执行请求模板，后续由Encoder和HTTP Client处理。
- **方法参数注解**：支持`@RequestParam`、`@PathVariable`、`@RequestBody`、`@RequestHeader`等Spring MVC标准参数注解。
- **OpenFeign自定义拦截器**: (1) 实现`RequestInterceptor`接口; (2) 重写`apply(RequestTemplate template)`方法添加自定义逻辑(如header注入token/添加traceId/记录日志); (3) 将拦截器类注册到Spring容器。常见用途: 全局Token注入/请求日志/链路追踪traceId传递

# 2. 面试题

## 2.1 Feign 和 OpenFeign 有什么区别？
- **Feign**（Netflix）：最初由Netflix开源，2016年后基本停止更新，不支持Spring MVC注解，需要自定义注解@RequestLine、@Param等，字段解析和编码器扩展性有限。
- **OpenFeign**（Spring Cloud社区）：基于Feign持续维护和升级的分支：
  1. 支持Spring MVC注解（@GetMapping、@PostMapping、@PathVariable等），降低学习成本
  2. 增加了RequestInterceptor支持
  3. 增加了请求/响应Gzip压缩支持
  4. 深度集成Spring Cloud LoadBalancer
  5. 更好的Bean生命周期管理和配置自动装配
  6. 支持@QueryMap动态查询参数
  7. 支持FallbackFactory，可获取异常原因
- Java 11+支持方面，OpenFeign后续版本提供了对HttpClient5、OkHttp的适配。

## 2.2 OpenFeign 的底层实现原理是怎样的？
1. **启动阶段（接口注册）**：
   - `@EnableFeignClients`通过`@Import(FeignClientsRegistrar.class)`引入注册器
   - 扫描指定包下的`@FeignClient`接口
   - 为每个接口注册`FeignClientFactoryBean`的BeanDefinition

2. **实例化阶段（代理创建）**：
   - `FeignClientFactoryBean.getObject()` 创建代理对象
   - 构建Feign.Builder，配置Contract、Encoder、Decoder、Logger、RequestInterceptor
   - 调用`Feign.target()`创建JDK动态代理

3. **调用阶段（请求执行）**：
   - 调用FeignClient接口方法
   - 被`FeignInvocationHandler.invoke()`拦截
   - `SynchronousMethodHandler`执行请求：
     a. `Contract`解析方法注解 → 生成`MethodMetadata`
     b. 将参数值填入 → 生成`RequestTemplate`
     c. `RequestInterceptor`拦截器链处理
     d. `Encoder`编码请求体
     e. `Client.execute()`发送HTTP请求
     f. `Decoder`解码响应体

## 2.3 OpenFeign 的日志级别有哪些？如何配置？
**四种日志级别**（`Logger.Level`）：
- `NONE`：不记录任何日志（默认值，生产环境推荐）
- `BASIC`：记录请求方法、URL、响应状态码、执行时间
- `HEADERS`：BASIC + 请求头信息、响应头信息
- `FULL`：HEADERS + 请求体内容、响应体内容

**配置方式**：
```java
@Configuration
public class FeignConfig {
    @Bean
    public Logger.Level feignLoggerLevel() {
        return Logger.Level.BASIC; // 或 FULL / HEADERS / NONE
    }
}

// 在@FeignClient上引用
@FeignClient(name = "order-service", configuration = FeignConfig.class)
```

```yaml
logging:
  level:
    com.example.feign.OrderFeignClient: DEBUG  # Feign接口的日志级别必须设为DEBUG
```

## 2.4 如何配置 OpenFeign 的超时时间？
**方式一：全局配置**（针对所有服务）
```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          default:
            connectTimeout: 5000   # 连接超时（毫秒）
            readTimeout: 10000     # 读取超时（毫秒）
```

**方式二：按服务名配置**（针对特定服务）
```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          order-service:          # 服务名
            connectTimeout: 3000
            readTimeout: 5000
```

**方式三：使用HttpClient5时**，超时配置在HttpClient层面：
```java
@Bean
public CloseableHttpClient httpClient() {
    return HttpClientBuilder.create()
        .setConnectionTimeToLive(10, TimeUnit.SECONDS)
        .build();
}
```

**注意**：当底层使用其他HTTP客户端（OkHttp、Apache HttpClient）时，Feign的timeout配置可能不生效，需要在该客户端的配置中设置。

## 2.5 RequestInterceptor 的作用和典型使用场景？
`RequestInterceptor`用于在HTTP请求发送前修改`RequestTemplate`：

```java
@Component
public class FeignAuthInterceptor implements RequestInterceptor {
    @Override
    public void apply(RequestTemplate template) {
        // 1. 添加认证Token
        template.header("Authorization", getToken());
        // 2. 透传TraceId实现链路追踪
        template.header("X-Trace-Id", MDC.get("traceId"));
        // 3. 添加自定义请求头
        template.header("X-Source", "gateway-service");
    }
}
```

**典型场景**：
1. 统一认证：自动添加JWT Token、OAuth2 Bearer Token
2. 链路追踪：透传TraceId/SpanId，保持整条调用链路可追踪
3. 灰度标识：透传灰度标签（version、env等），实现全链路灰度
4. 版本控制：添加API版本号，路由到对应后端服务
5. 多线程传递：使用`RequestContextHolder`将主线程的请求上下文传递到Feign线程

## 2.6 生产环境中 OpenFeign 的性能优化有哪些？
1. **替换HTTP客户端**（最重要）：
   - 默认URLConnection不支持连接池（每次新建连接）
   - 替换为Apache HttpClient5（连接池、KeepAlive）
   - 替换为OkHttp（支持HTTP/2、连接池轻量）
```yaml
spring:
  cloud:
    openfeign:
      httpclient:
        hc5:
          enabled: true
```

2. **连接池配置**（HttpClient5）：
```java
@Bean
public PoolingHttpClientConnectionManager poolingConnectionManager() {
    PoolingHttpClientConnectionManager manager = new PoolingHttpClientConnectionManager();
    manager.setMaxTotal(200);           // 最大连接数
    manager.setDefaultMaxPerRoute(100);  // 每个路由最大连接数
    manager.setConnectionTimeToLive(30, TimeUnit.SECONDS);
    return manager;
}
```

3. **启用Gzip压缩**：
```yaml
feign:
  compression:
    request:
      enabled: true
      mime-types: application/json
      min-request-size: 2048
    response:
      enabled: true
```

4. **合理设置超时**：connectTimeout不宜过长（3-5秒），readTimeout根据业务调整
5. **日志级别**：生产环境设NONE或BASIC，避免FULL级别记录Body造成性能损耗
6. **连接KeepAlive**：配置长连接复用，减少TCP三次握手开销
7. **异步调用**：对非关键路径的接口使用Feign异步调用（CompletableFuture）

## 2.7 OpenFeign 如何处理错误和降级？
**方式一：ErrorDecoder自定义错误处理**
```java
public class CustomErrorDecoder implements ErrorDecoder {
    @Override
    public Exception decode(String methodKey, Response response) {
        // 根据状态码返回对应异常
        if (response.status() == 404) {
            return new NotFoundException("资源不存在");
        }
        if (response.status() >= 500) {
            return new ServiceUnavailableException("服务端异常");
        }
        return new Default().decode(methodKey, response);
    }
}
```

**方式二：Fallback降级类**
```java
@FeignClient(name = "order-service", fallback = OrderFallback.class)
public interface OrderFeignClient {
    @GetMapping("/orders/{id}")
    Order getOrder(@PathVariable Long id);
}

@Component
public class OrderFallback implements OrderFeignClient {
    @Override
    public Order getOrder(Long id) {
        return new Order(); // 返回默认值
    }
}
```

**方式三：FallbackFactory（推荐）**
```java
@FeignClient(name = "order-service", fallbackFactory = OrderFallbackFactory.class)
public interface OrderFeignClient { ... }

@Component
public class OrderFallbackFactory implements FallbackFactory<OrderFeignClient> {
    @Override
    public OrderFeignClient create(Throwable cause) {
        log.error("调用order-service异常", cause);
        return id -> new Order(); // 可记录异常原因
    }
}
```

## 2.8 Feign 如何与 Sentinel 整合实现熔断降级？
**步骤一**：添加依赖
```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
```

**步骤二**：开启Sentinel支持
```yaml
feign:
  sentinel:
    enabled: true  # 开启Sentinel对Feign的支持
```

**步骤三**：配置Fallback
```java
@FeignClient(
    name = "order-service",
    fallback = OrderFeignFallback.class,          // 降级类
    fallbackFactory = OrderFeignFallbackFactory.class  // 可捕获异常
)
```

**原理**：
- `feign.sentinel.enabled=true`后，Feign的代理对象会被Sentinel包装
- 每个Feign方法自动成为Sentinel的@SentinelResource资源
- 资源名格式：`GET:http://order-service/orders/{id}`
- 通过Sentinel控制台可对Feign接口配置流控、降级规则
- Fallback类中的方法作为blockHandler的降级逻辑

## 2.9 OpenFeign 请求调用过程中的详细步骤是什么？
1. **方法调用**：调用`@FeignClient`接口的方法（如`orderClient.getOrderById(1001L)`）
2. **动态代理拦截**：`FeignInvocationHandler.invoke()`拦截方法调用
3. **方法匹配**：根据方法签名查找对应的`MethodHandler`（`SynchronousMethodHandler`）
4. **构建RequestTemplate**：
   - `Contract`解析方法注解生成`MethodMetadata`
   - 提取方法参数值（`@PathVariable`、`@RequestParam`、`@RequestBody`）
   - 将元数据和参数值合并为`RequestTemplate`对象
5. **拦截器处理**：遍历所有`RequestInterceptor`，依次调用`apply(template)`，添加Header等
6. **负载均衡**：通过`LoadBalancerCommand`从注册中心获取服务实例列表，选择一个实例
7. **URL拼接**：将选中的实例地址（如`http://192.168.1.100:8080`）与`RequestTemplate`的路径和参数拼接为完整URL
8. **Encoder编码**：如果有`@RequestBody`，通过`Encoder`将Java对象序列化为JSON/XML字符串
9. **执行请求**：通过HTTP Client发送请求（URLConnection/OkHttp/HttpClient5）
10. **Decoder解码**：收到响应后，通过`Decoder`将响应Body反序列化为返回值类型
11. **异常处理**：如果响应状态码非2xx，通过`ErrorDecoder`解析为业务异常
12. **返回结果**：将反序列化后的对象返回给调用方

## 2.10 Openfeign如何自定义拦截器？
自定义拦截器的核心是实现 `RequestInterceptor` 接口并注册到Spring容器：

**步骤一**：实现 `RequestInterceptor` 接口
```java
@Component
public class CustomFeignInterceptor implements RequestInterceptor {
    @Override
    public void apply(RequestTemplate template) {
        // 1. 全局Token注入
        ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        if (attributes != null) {
            String token = attributes.getRequest().getHeader("Authorization");
            if (StringUtils.hasText(token)) {
                template.header("Authorization", token);
            }
        }
        
        // 2. 链路追踪traceId传递
        String traceId = MDC.get("traceId");
        if (StringUtils.hasText(traceId)) {
            template.header("X-Trace-Id", traceId);
        }
        
        // 3. 请求日志记录
        log.info("Feign请求: method={}, url={}", template.method(), template.url());
    }
}
```

**步骤二**：注册到Spring容器（`@Component`、`@Configuration` 或 `@Bean` 方式均可）

**步骤三**（可选）：配置特定FeignClient使用的拦截器
```java
@Configuration
public class SpecificFeignConfig {
    @Bean
    public RequestInterceptor specificInterceptor() {
        return template -> template.header("X-Custom", "value");
    }
}

@FeignClient(name = "order-service", configuration = SpecificFeignConfig.class)
public interface OrderClient { ... }
```

**常见用途**：
| 场景 | 说明 |
|------|------|
| 全局Token注入 | 从请求上下文获取JWT/OAuth2 Token，自动添加到Feign请求头 |
| 链路追踪 | 透传TraceId/SpanId，保证调用链路完整可追踪 |
| 请求日志 | 记录Feign调用的方法、URL、耗时等 |
| 灰度标识 | 透传版本号/环境标签，实现全链路灰度路由 |
| 签名认证 | 自动计算并添加请求签名Header |
