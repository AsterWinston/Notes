
# 1. 项目结构

采用多模块 Maven 项目，父 POM 统一管理依赖：

```
dubbo-demo/
├── pom.xml                    (父 POM)
├── dubbo-demo-api/
│   └── pom.xml                (API 模块)
│   └── src/main/java/org/example/demo/api/
│       └── UserService.java   (服务接口)
│       └── User.java          (传输对象)
├── dubbo-demo-provider/
│   └── pom.xml                (Provider 模块)
│   └── src/main/java/org/example/demo/provider/
│       └── UserServiceImpl.java (服务实现)
│       └── ProviderApplication.java
│   └── src/main/resources/
│       └── application.yml
└── dubbo-demo-consumer/
    └── pom.xml                (Consumer 模块)
    └── src/main/java/org/example/demo/consumer/
        └── UserController.java
        └── ConsumerApplication.java
    └── src/main/resources/
        └── application.yml
```

# 2. 父 POM

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
    </parent>

    <groupId>org.example</groupId>
    <artifactId>dubbo-demo</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>

    <modules>
        <module>dubbo-demo-api</module>
        <module>dubbo-demo-provider</module>
        <module>dubbo-demo-consumer</module>
    </modules>

    <properties>
        <java.version>17</java.version>
        <dubbo.version>3.3.0</dubbo.version>
        <spring-cloud.version>2023.0.0</spring-cloud.version>
        <spring-cloud-alibaba.version>2023.0.1.0</spring-cloud-alibaba.version>
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
            <dependency>
                <groupId>com.alibaba.cloud</groupId>
                <artifactId>spring-cloud-alibaba-dependencies</artifactId>
                <version>${spring-cloud-alibaba.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
```

# 3. API 模块

```java
// User.java
package org.example.demo.api;

import java.io.Serializable;

public class User implements Serializable {
    private static final long serialVersionUID = 1L;
    private Long id;
    private String name;
    private Integer age;

    public User() {}
    public User(Long id, String name, Integer age) {
        this.id = id;
        this.name = name;
        this.age = age;
    }

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public Integer getAge() { return age; }
    public void setAge(Integer age) { this.age = age; }

    @Override
    public String toString() {
        return "User{id=" + id + ", name='" + name + "', age=" + age + "}";
    }
}
```

```java
// UserService.java
package org.example.demo.api;

public interface UserService {
    User getUser(Long id);
    User createUser(User user);
}
```

API 模块 POM：

```xml
<project>
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.example</groupId>
        <artifactId>dubbo-demo</artifactId>
        <version>1.0.0</version>
    </parent>
    <artifactId>dubbo-demo-api</artifactId>
    <!-- 无需额外依赖，仅定义接口和传输对象 -->
</project>
```

# 4. Provider 模块

```xml
<!-- Provider POM -->
<project>
    <parent>
        <groupId>org.example</groupId>
        <artifactId>dubbo-demo</artifactId>
        <version>1.0.0</version>
    </parent>
    <artifactId>dubbo-demo-provider</artifactId>
    <dependencies>
        <dependency>
            <groupId>org.example</groupId>
            <artifactId>dubbo-demo-api</artifactId>
            <version>1.0.0</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        </dependency>
        <dependency>
            <groupId>org.apache.dubbo</groupId>
            <artifactId>dubbo-spring-boot-starter</artifactId>
            <version>${dubbo.version}</version>
        </dependency>
    </dependencies>
</project>
```

```java
// UserServiceImpl.java
package org.example.demo.provider;

import org.apache.dubbo.config.annotation.DubboService;
import org.example.demo.api.User;
import org.example.demo.api.UserService;

@DubboService
public class UserServiceImpl implements UserService {

    @Override
    public User getUser(Long id) {
        return new User(id, "user-" + id, 25);
    }

    @Override
    public User createUser(User user) {
        user.setId(System.currentTimeMillis());
        return user;
    }
}
```

```java
// ProviderApplication.java
package org.example.demo.provider;

import org.apache.dubbo.config.spring.context.annotation.EnableDubbo;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
@EnableDubbo
public class ProviderApplication {
    public static void main(String[] args) {
        SpringApplication.run(ProviderApplication.class, args);
    }
}
```

```yaml
# application.yml
server:
  port: 8081

spring:
  application:
    name: dubbo-demo-provider

  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
        namespace: dev

dubbo:
  application:
    name: dubbo-demo-provider
    logger: slf4j
  protocol:
    name: dubbo
    port: 20880
  registry:
    address: nacos://127.0.0.1:8848
    parameters:
      namespace: dev
  scan:
    base-packages: org.example.demo.provider
```

# 5. Consumer 模块

```xml
<!-- Consumer POM -->
<project>
    <parent>
        <groupId>org.example</groupId>
        <artifactId>dubbo-demo</artifactId>
        <version>1.0.0</version>
    </parent>
    <artifactId>dubbo-demo-consumer</artifactId>
    <dependencies>
        <dependency>
            <groupId>org.example</groupId>
            <artifactId>dubbo-demo-api</artifactId>
            <version>1.0.0</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        </dependency>
        <dependency>
            <groupId>org.apache.dubbo</groupId>
            <artifactId>dubbo-spring-boot-starter</artifactId>
            <version>${dubbo.version}</version>
        </dependency>
    </dependencies>
</project>
```

```java
// UserController.java
package org.example.demo.consumer;

import org.apache.dubbo.config.annotation.DubboReference;
import org.example.demo.api.User;
import org.example.demo.api.UserService;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/users")
public class UserController {

    @DubboReference
    private UserService userService;

    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.getUser(id);
    }

    @PostMapping
    public User createUser(@RequestBody User user) {
        return userService.createUser(user);
    }
}
```

```java
// ConsumerApplication.java
package org.example.demo.consumer;

import org.apache.dubbo.config.spring.context.annotation.EnableDubbo;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
@EnableDubbo
public class ConsumerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConsumerApplication.class, args);
    }
}
```

```yaml
# application.yml
server:
  port: 8082

spring:
  application:
    name: dubbo-demo-consumer

  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
        namespace: dev

dubbo:
  application:
    name: dubbo-demo-consumer
    logger: slf4j
  registry:
    address: nacos://127.0.0.1:8848
    parameters:
      namespace: dev
  consumer:
    check: false          # 不强制启动时检查 Provider
    timeout: 3000          # 超时时间
    retries: 2             # 重试次数
```

# 6. 启动验证

1. 启动 Nacos（确保 8848 端口可用）
2. 启动 ProviderApplication，观察日志中 `[DUBBO] Export dubbo service` 信息
3. 启动 ConsumerApplication，观察日志中 `[DUBBO] Subscribe` 信息
4. 调用 Consumer：

```bash
curl http://localhost:8082/users/1
# 返回: {"id":1,"name":"user-1","age":25}

curl -X POST http://localhost:8082/users \
  -H "Content-Type: application/json" \
  -d '{"name":"new-user","age":30}'
# 返回: {"id":1700000000000,"name":"new-user","age":30}
```

# 7. 关键注解说明

| 注解 | 作用 | 位置 |
|------|------|------|
| `@EnableDubbo` | 启用 Dubbo 自动配置 | 启动类 |
| `@DubboService` | 标记为 Dubbo 服务提供者，会暴露服务并注册 | Provider 实现类 |
| `@DubboReference` | 引用 Dubbo 服务，注入代理对象 | Consumer 字段/setter |

# 8. 配置要点

```yaml
dubbo:
  application:
    name: app-name              # 应用名，必填
  registry:
    address: nacos://host:port  # 注册中心地址，格式: <type>://<host>:<port>
  protocol:
    name: dubbo                 # 暴露协议，默认 dubbo
    port: 20880                 # 协议端口，默认 20880
  provider:
    timeout: 3000               # 服务端超时
    retries: 0                  # 服务端重试（与消费端重试相乘）
    version: 1.0.0              # 服务版本
    group: dev                  # 服务分组
    threads: 200                # 业务线程池大小
  consumer:
    timeout: 3000               # 调用超时
    retries: 2                  # 失败重试次数
    check: false                # 启动时不检查 Provider 可用性
    version: 1.0.0              # 服务版本
```
