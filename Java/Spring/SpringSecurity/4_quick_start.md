# 1. 准备工作

## 1.1 创建 Spring Boot 项目

**方式一：IDEA 新建项目**
```
File → New → Project → Spring Initializr
  - Project: Maven
  - Language: Java
  - Spring Boot: 3.2.x
  - Dependencies: Spring Web, Spring Security
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
    <artifactId>spring-security-demo</artifactId>
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
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

**方式三：Gradle（Spring Boot 3.x）**

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.2.0'
    id 'io.spring.dependency-management' version '1.1.4'
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-security'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

## 1.2 启动类

```java
package com.example;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class SecurityApplication {
    public static void main(String[] args) {
        SpringApplication.run(SecurityApplication.class, args);
    }
}
```

---

# 2. 引入 Spring Security

## 2.1 自动配置机制

添加 `spring-boot-starter-security` 后，Spring Boot 自动做了以下事情：

1. **启用安全过滤器链**：自动注册 `springSecurityFilterChain` Bean
2. **生成默认用户**：用户名 `user`，密码在启动日志随机生成
3. **生成默认登录页**：`/login`（GET）
4. **配置登录接口**：`/login`（POST）
5. **配置登出接口**：`/logout`（GET/POST）
6. **保护所有端点**：所有请求需要认证
7. **启用 CSRF 防护**
8. **默认 PasswordEncoder**：使用 `DelegatingPasswordEncoder`

```
Using generated security password: 8e4f6a3b-b5c2-4d7f-9a1e-2f8c6d3a1b5f
```

## 2.2 默认登录页和用户

启动后访问任何 URL，自动跳转到默认登录页：

```
http://localhost:8080/ → 302 → http://localhost:8080/login
```

默认登录页由 `DefaultLoginPageGeneratingFilter` 动态生成，显示如下：

```
┌──────────────────────────────┐
│       Please sign in         │
│                              │
│  User:  [________]           │
│  Password: [________]        │
│  [Sign in]                   │
└──────────────────────────────┘
```

## 2.3 测试默认安全效果

**创建测试 Controller：**

```java
package com.example.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {

    @GetMapping("/")
    public String home() {
        return "<h1>Welcome!</h1><a href='/user/hello'>User Page</a>";
    }

    @GetMapping("/user/hello")
    public String userHello() {
        return "Hello, " + SecurityContextHolder.getContext()
                   .getAuthentication().getName() + "!";
    }
}
```

**验证效果：**

| 操作 | 结果 |
|------|------|
| 直接访问 `/user/hello` | 302 重定向到 `/login` |
| 用 `user` + 随机密码登录 | 300+ 重定向回 `/user/hello` |
| 访问 `/logout` | 登出，清空 Session |

---

# 3. 自定义安全配置

## 3.1 基础 SecurityFilterChain 配置

```java
package com.example.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configurers.AbstractHttpConfigurer;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/public/**").permitAll()  // 公开访问
                .requestMatchers("/admin/**").hasRole("ADMIN")   // 需要 ADMIN 角色
                .anyRequest().authenticated()                     // 其余需要登录
            )
            .formLogin(form -> form
                .loginPage("/login")         // 自定义登录页
                .loginProcessingUrl("/login")// 表单提交地址
                .defaultSuccessUrl("/")      // 登录成功后跳转
                .permitAll()
            )
            .logout(logout -> logout
                .logoutUrl("/logout")
                .logoutSuccessUrl("/")
                .permitAll()
            );

        return http.build();
    }
}
```

**Spring Security 6.x 新写法 vs 旧写法：**

```java
// 6.x 推荐写法（Lambda DSL）
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/admin/**").hasRole("ADMIN")
    .anyRequest().authenticated()
);

// 旧写法（6.x 仍支持，但不推荐）
http.authorizeHttpRequests()
    .requestMatchers("/admin/**").hasRole("ADMIN")
    .anyRequest().authenticated();
```

## 3.2 添加密码编码器

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```

## 3.3 编写自定义登录页

```html
<!-- src/main/resources/static/login.html -->
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>登录</title>
</head>
<body>
    <h2>用户登录</h2>
    <form action="/login" method="post">
        <input type="text" name="username" placeholder="用户名" required>
        <input type="password" name="password" placeholder="密码" required>
        <button type="submit">登录</button>
    </form>
    <!-- 如果启用 CSRF，必须携带 token -->
    <!-- <input type="hidden" th:name="${_csrf.parameterName}" th:value="${_csrf.token}"> -->
</body>
</html>
```

## 3.4 项目完整结构

```
src/
├── main/
│   ├── java/com/example/
│   │   ├── SecurityApplication.java          // 启动类
│   │   ├── config/
│   │   │   └── SecurityConfig.java           // 安全配置
│   │   └── controller/
│   │       └── HelloController.java          // 测试接口
│   └── resources/
│       ├── application.yml                   // 配置文件
│       └── static/
│           └── login.html                    // 自定义登录页
└── test/
    └── java/com/example/
        └── SecurityApplicationTests.java
```
