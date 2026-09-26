# 1. 准备工作

## 1.1 环境要求

| 组件 | 版本要求 |
|------|----------|
| JDK | 17+ |
| Spring Boot | 3.2.x |
| Redis（如需） | 6.x+ |
| MySQL（如需） | 8.0+ |

## 1.2 创建 Spring Boot 项目

**方式一：Spring Initializr (IDEA)**

```
File → New → Project → Spring Initializr
  - Project: Maven
  - Language: Java
  - Spring Boot: 3.2.x
  - Dependencies: Spring Web, Spring Session, Spring Data Redis
```

**方式二：Maven 手动创建 pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
    </parent>

    <groupId>com.example</groupId>
    <artifactId>spring-session-demo</artifactId>
    <version>1.0.0</version>

    <properties>
        <java.version>17</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.session</groupId>
            <artifactId>spring-session-data-redis</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
    </dependencies>
</project>
```

---

# 2. 最简配置

## 2.1 application.yml

```yaml
server:
  port: 8080

spring:
  data:
    redis:
      host: localhost
      port: 6379
      # password: your-password
      # database: 0

  session:
    timeout: 30m                  # Session 超时 30 分钟
    redis:
      namespace: demo:session    # Redis key 命名空间
      flush-mode: on_save        # 刷新模式
```

## 2.2 启动类

```java
package com.example;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class SessionDemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(SessionDemoApplication.class, args);
    }
}
```

**启动后 Spring Boot 自动配置会做以下事情**：

1. 检测到 `spring-session-data-redis` 在 classpath
2. 自动创建 `RedisSessionRepository`
3. 自动注册 `SessionRepositoryFilter`
4. 替换默认的 Tomcat `HttpSession` 实现

## 2.3 验证

```java
package com.example.controller;

import jakarta.servlet.http.HttpSession;
import org.springframework.web.bind.annotation.*;

@RestController
public class SessionController {

    @GetMapping("/session/set")
    public String setSession(HttpSession session,
                              @RequestParam String key,
                              @RequestParam String value) {
        session.setAttribute(key, value);
        return "Session 属性已设置: " + key + " = " + value
                + ", Session ID: " + session.getId();
    }

    @GetMapping("/session/get")
    public Object getSession(HttpSession session,
                              @RequestParam String key) {
        return "Session 属性: " + key + " = "
                + session.getAttribute(key)
                + ", Session ID: " + session.getId();
    }

    @GetMapping("/session/info")
    public String sessionInfo(HttpSession session) {
        return String.format(
            "Session ID: %s\n创建时间: %s\n最后访问: %s\n最大非活动时间: %ds",
            session.getId(),
            session.getCreationTime(),
            session.getLastAccessedTime(),
            session.getMaxInactiveInterval()
        );
    }
}
```

**测试**：

```bash
# 设置 Session 属性
curl -c cookies.txt -b cookies.txt \
  "http://localhost:8080/session/set?key=username&value=zhangsan"

# 读取 Session 属性
curl -c cookies.txt -b cookies.txt \
  "http://localhost:8080/session/get?key=username"

# 在 Redis 中查看
redis-cli KEYS "demo:session:*"
redis-cli HGETALL "demo:session:sessions:<sessionId>"
```

---

# 3. 编程式配置

## 3.1 @EnableRedisHttpSession

对于非 Spring Boot 环境或需要自定义配置，使用注解：

```java
package com.example.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.session.data.redis.config.annotation.web.http.EnableRedisHttpSession;
import org.springframework.session.web.http.CookieHttpSessionIdResolver;
import org.springframework.session.web.http.HttpSessionIdResolver;

@Configuration
@EnableRedisHttpSession(
    maxInactiveIntervalInSeconds = 1800,   // 30 分钟
    redisNamespace = "demo:session",
    flushMode = FlushMode.ON_SAVE,
    saveMode = SaveMode.ON_SET_ATTRIBUTE
)
public class SessionConfig {

    @Bean
    public HttpSessionIdResolver httpSessionIdResolver() {
        CookieHttpSessionIdResolver resolver =
            new CookieHttpSessionIdResolver();
        // 设置 Cookie 属性
        resolver.setCookieName("MYSESSION");
        // 注意：Cookie 的其他属性（HttpOnly、Secure、SameSite 等）
        // 在 Spring Boot 中通过 server.servlet.session.cookie.* 配置更简便
        return resolver;
    }
}
```

## 3.2 自定义 RedisSerializer

```java
package com.example.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializer;

@Configuration
public class RedisConfig {

    @Bean
    public RedisSerializer<Object> springSessionDefaultRedisSerializer() {
        // 替换默认的 JDK 序列化为 JSON 序列化
        // 优点：可读性好，跨语言支持
        // 注意：需要处理类型安全问题
        return new GenericJackson2JsonRedisSerializer();
    }
}
```

---

# 4. 集成 Spring Security

## 4.1 添加依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

## 4.2 安全配置

```java
package com.example.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.core.userdetails.*;
import org.springframework.security.provisioning.InMemoryUserDetailsManager;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http)
            throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .formLogin(Customizer.withDefaults())
            .sessionManagement(session -> session
                .maximumSessions(1)                       // 同一用户最多 1 个会话
                .maxSessionsPreventsLogin(false)          // 超过时踢掉旧会话
                .expiredUrl("/login?expired")             // 会话过期跳转
            );

        return http.build();
    }

    @Bean
    public UserDetailsService userDetailsService() {
        UserDetails user = User.builder()
            .username("admin")
            .password("{noop}123456")
            .roles("ADMIN")
            .build();
        return new InMemoryUserDetailsManager(user);
    }
}
```

## 4.3 验证并发会话控制

```bash
# 第一次登录（正常）
curl -c cookies1.txt -X POST \
  "http://localhost:8080/login" \
  -d "username=admin&password=123456"

# 第二次登录（会踢掉第一次的会话）
curl -c cookies2.txt -X POST \
  "http://localhost:8080/login" \
  -d "username=admin&password=123456"

# 使用旧的 Cookie 访问 → 会话已过期，重定向到 /login?expired
curl -b cookies1.txt "http://localhost:8080/home"
```

---

# 5. 前后端分离架构（REST API）

## 5.1 Header 方式传递 Session ID

```java
package com.example.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.session.web.http.HeaderHttpSessionIdResolver;
import org.springframework.session.web.http.HttpSessionIdResolver;

@Configuration
public class RestSessionConfig {

    @Bean
    public HttpSessionIdResolver httpSessionIdResolver() {
        // 使用 Header 传递 Session ID，而非 Cookie
        return new HeaderHttpSessionIdResolver("X-Auth-Token");
    }
}
```

## 5.2 前端调用示例

```javascript
// 登录
async function login(username, password) {
    const response = await fetch('/login', {
        method: 'POST',
        headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
        body: `username=${username}&password=${password}`
    });

    // 获取响应头中的 X-Auth-Token
    const authToken = response.headers.get('X-Auth-Token');
    localStorage.setItem('X-Auth-Token', authToken);
}

// 后续请求携带 Token
async function fetchProfile() {
    const token = localStorage.getItem('X-Auth-Token');
    const response = await fetch('/api/profile', {
        headers: { 'X-Auth-Token': token }
    });
    return response.json();
}
```

---

# 6. Spring Boot 自动配置详解

## 6.1 自动配置类

Spring Boot 通过自动配置检测 classpath 上的依赖，按优先级选择后端：

```
检测顺序：
1. spring-session-data-redis      → RedisSessionRepository
2. spring-session-jdbc             → JdbcSessionRepository
3. spring-session-hazelcast        → HazelcastSessionRepository
4. spring-session-data-mongodb     → MongoSessionRepository
5. 都没有                            → MapSessionRepository (默认)
```

## 6.2 关键自动配置类

| 配置类 | 作用 |
|--------|------|
| `RedisSessionAutoConfiguration` | 配置 `RedisSessionRepository` |
| `JdbcSessionAutoConfiguration` | 配置 `JdbcSessionRepository` |
| `HazelcastSessionAutoConfiguration` | 配置 `HazelcastSessionRepository` |
| `SessionAutoConfiguration` | 通用配置（Filter 注册等） |
| `RedisReactiveSessionAutoConfiguration` | WebFlux 环境配置 |

## 6.3 禁用自动配置

```java
@SpringBootApplication(exclude = {
    org.springframework.boot.autoconfigure.session.SessionAutoConfiguration.class
})
public class Application {
    // ...
}
```

或者：

```yaml
spring:
  autoconfigure:
    exclude:
      - org.springframework.boot.autoconfigure.session.SessionAutoConfiguration
```

---

# 7. 验证 Session 已生效

## 7.1 Redis 验证

```bash
# 查看所有 Session key
redis-cli KEYS "demo:session:*"

# 按类型查看
redis-cli KEYS "demo:session:sessions:*"     # Session 数据 (HASH)
redis-cli KEYS "demo:session:expirations:*"  # 过期时间
redis-cli KEYS "demo:session:index:*"        # 用户索引

# 查看具体 Session
redis-cli HGETALL "demo:session:sessions:abc123"
redis-cli TTL "demo:session:sessions:abc123"
```

## 7.2 代码验证

```java
@RestController
public class CheckController {

    @Autowired
    private SessionRepository<?> sessionRepository;

    @GetMapping("/check/session-count")
    public String checkSessionCount() {
        // 此方法仅在 FindByIndexNameSessionRepository 实现中可用
        return "请通过 Redis CLI 或 JDBC 查询验证";
    }
}
```

## 7.3 多实例验证

```bash
# 启动实例 1
java -jar app.jar --server.port=8080

# 启动实例 2
java -jar app.jar --server.port=8081

# 在实例 1 上设置 Session
curl -c cookies.txt -b cookies.txt \
  "http://localhost:8080/session/set?key=test&value=hello"

# 在实例 2 上读取 Session（使用相同的 Cookie）
curl -b cookies.txt \
  "http://localhost:8081/session/get?key=test"
# 输出: Session 属性: test = hello  ← 成功跨实例共享
```
