
# 1. 环境要求

- JDK 1.8+
- Spring Boot 2.0.x ~ 2.3.x（Spring Cloud Hoxton 及其之前的版本）
- Spring Cloud Hoxton.SR12（推荐，最后一个支持全面 Netflix 组件的发行版）
- Maven 3.3+ 或 Gradle 5.x+

> **注意**：Spring Cloud 2020.0（Ilford）以后，Netflix 系列组件均已移除或标记 deprecated，请勿在高版本 Spring Cloud 中使用 Zuul。

# 2. 项目搭建

## 2.1 父工程依赖管理

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.3.12.RELEASE</version>
    <relativePath/>
</parent>

<properties>
    <java.version>1.8</java.version>
    <spring-cloud.version>Hoxton.SR12</spring-cloud.version>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

## 2.2 Zuul 网关工程依赖

```xml
<dependencies>
    <!-- Zuul 网关核心 -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
    </dependency>

    <!-- Eureka 客户端（服务发现） -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>

    <!-- Spring Boot Actuator（监控端点） -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

`spring-cloud-starter-netflix-zuul` 传递依赖包含：
- `zuul-core`：Netflix Zuul 核心库
- `spring-cloud-starter-netflix-hystrix`：Hystrix 熔断器
- `spring-cloud-starter-netflix-ribbon`：Ribbon 负载均衡
- `spring-cloud-starter-netflix-archaius`：配置管理
- `spring-boot-starter-web`：内嵌 Tomcat

## 2.3 启动类

```java
package com.example.zuul;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.zuul.EnableZuulProxy;

@SpringBootApplication
@EnableZuulProxy
public class ZuulGatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(ZuulGatewayApplication.class, args);
    }
}
```

# 3. 基础配置

## 3.1 application.yml

```yaml
server:
  port: 8080

spring:
  application:
    name: zuul-gateway

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true

zuul:
  routes:
    # 路由规则 1：基于 serviceId 的面向服务路由
    user-service:
      path: /user/**
      serviceId: user-service
      stripPrefix: false  # 不剥离 /user 前缀

    # 路由规则 2：简化写法（serviceId 省略，默认取 Map 的 key）
    order-service:
      path: /order/**
      # 等价于 serviceId: order-service

    # 路由规则 3：基于 URL 的物理路由
    external-api:
      path: /external/**
      url: http://external-service.example.com

  # 全局敏感 Header 配置
  sensitive-headers:
    - Cookie
    - Set-Cookie
```

## 3.2 自动路由机制

当注册中心（如 Eureka）中存在服务时，Zuul 会**自动创建默认路由规则**，规则为：
```
/{serviceId}/** → serviceId
```

例如，Eureka 中注册了 `user-service`，则自动生成路由：
```
/user-service/** → user-service
```

可以通过配置关闭此行为：

```yaml
zuul:
  ignored-services: "*"  # 忽略所有自动路由，只使用显式配置的路由
```

## 3.3 路由规则优先级

Zuul 路由匹配的优先级从高到低：

1. **显式配置的 URL 路由**（`url` 不为空）
2. **显式配置的 serviceId 路由**（`serviceId` 不为空且非 `url` 路由）
3. **自动生成的 serviceId 路由**（Eureka 中的服务自动映射）
4. **通配符路由**（兜底路由）

```yaml
zuul:
  routes:
    # 优先级最高
    legacy-api:
      path: /api/v1/**
      url: http://legacy-server:8080

    # 优先级次之
    user-api:
      path: /api/v2/user/**
      serviceId: user-service

    # 自动路由优先级最低
    # Eureka 中的 product-service → /product-service/**
```

# 4. 后端服务模拟

## 4.1 用户服务（user-service）

```java
@SpringBootApplication
@EnableDiscoveryClient
@RestController
public class UserServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(UserServiceApplication.class, args);
    }

    @GetMapping("/user/{id}")
    public String getUser(@PathVariable Long id) {
        return "User-" + id + " from port:" + getPort();
    }

    @GetMapping("/user/list")
    public List<String> listUsers() {
        return Arrays.asList("Alice", "Bob", "Charlie");
    }

    private String getPort() {
        return System.getProperty("server.port", "8081");
    }
}
```

```yaml
# user-service 的 application.yml
server:
  port: 8081
spring:
  application:
    name: user-service
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

## 4.2 订单服务（order-service）

```java
@SpringBootApplication
@EnableDiscoveryClient
@RestController
public class OrderServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }

    @GetMapping("/order/{id}")
    public String getOrder(@PathVariable Long id) {
        return "Order-" + id + " from port:" + System.getProperty("server.port", "8082");
    }
}
```

```yaml
# order-service 的 application.yml
server:
  port: 8082
spring:
  application:
    name: order-service
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

# 5. 验证路由

启动 Eureka Server，再启动 user-service、order-service 和 zuul-gateway。

```bash
# 直接请求 user-service（绕过网关）
curl http://localhost:8081/user/1
# 输出: User-1 from port:8081

# 通过网关请求
curl http://localhost:8080/user/1
# 输出: User-1 from port:8081

# 自动路由方式（/user-service/** → user-service）
curl http://localhost:8080/user-service/user/1
# 输出: User-1 from port:8081

# 请求 order-service
curl http://localhost:8080/order/101
# 输出: Order-101 from port:8082

# 查看已生效的路由表
curl http://localhost:8080/actuator/routes
```

# 6. 项目目录结构

```
zuul-gateway/
├── pom.xml
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/example/zuul/
│   │   │       ├── ZuulGatewayApplication.java       # 启动类
│   │   │       ├── filter/                            # 自定义过滤器
│   │   │       │   ├── PreRequestLogFilter.java       # Pre 过滤器
│   │   │       │   └── PostResponseLogFilter.java     # Post 过滤器
│   │   │       ├── fallback/                          # 降级处理
│   │   │       │   └── UserServiceFallback.java
│   │   │       └── config/                            # 配置类
│   │   │           └── CorsConfig.java
│   │   └── resources/
│   │       ├── application.yml                        # 主配置
│   │       └── bootstrap.yml                          # 引导配置
│   └── test/
│       └── java/
│           └── com/example/zuul/
│               └── ZuulGatewayApplicationTests.java
```

# 7. 命令行启动方式

## 7.1 IDE 启动

在 IDE 中直接运行各服务的 `main()` 方法，注意修改各服务端口避免冲突：

```
Eureka Server   → localhost:8761
Zuul Gateway    → localhost:8080
User Service    → localhost:8081
Order Service   → localhost:8082
```

## 7.2 Maven 命令行启动不同实例

```bash
# 启动 user-service 实例 1
mvn spring-boot:run -Dspring-boot.run.arguments="--server.port=8081"

# 启动 user-service 实例 2（多实例，观察负载均衡）
mvn spring-boot:run -Dspring-boot.run.arguments="--server.port=8083"
```

启动后访问 Eureka Dashboard（`http://localhost:8761`），可以看到注册的服务列表。再通过 `http://localhost:8080/actuator/routes` 查看 Zuul 已生效的路由规则。
