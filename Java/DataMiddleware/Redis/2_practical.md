# 1. 短信登录

## 1.1 导入黑马点评项目

- 模块：用户、商户、优惠券、探店笔记、关注、附近商铺。
- 技术栈：Spring Boot + MyBatis Plus + Redis + MySQL + Nginx + 前端 H5。
- 关键表：`tb_user`、`tb_shop`、`tb_voucher`、`tb_follow`、`tb_blog`。

### 1.1.1 项目完整依赖 (pom.xml)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.7.12</version>
    </parent>
    <groupId>com.hmdp</groupId>
    <artifactId>hm-dianping</artifactId>
    <version>1.0.0</version>

    <properties>
        <java.version>8</java.version>
        <mybatis-plus.version>3.5.3.1</mybatis-plus.version>
        <hutool.version>5.8.18</hutool.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>mybatis-plus-boot-starter</artifactId>
            <version>${mybatis-plus.version}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        <dependency>
            <groupId>org.redisson</groupId>
            <artifactId>redisson-spring-boot-starter</artifactId>
            <version>3.23.4</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.session</groupId>
            <artifactId>spring-session-data-redis</artifactId>
        </dependency>
        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
        </dependency>
        <dependency>
            <groupId>cn.hutool</groupId>
            <artifactId>hutool-all</artifactId>
            <version>${hutool.version}</version>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-pool2</artifactId>
        </dependency>
        <dependency>
            <groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
            <version>32.0.1-jre</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

### 1.1.2 项目结构

```
src/main/java/com/hmdp/
├── HmDianPingApplication.java          # 启动类
├── config/
│   ├── MvcConfig.java                  # MVC拦截器注册配置
│   ├── RedisConfig.java                # Redis序列化配置
│   ├── RedissonConfig.java             # Redisson客户端配置
│   ├── ThreadPoolConfig.java           # 线程池配置
│   └── WebExceptionAdvice.java         # 全局异常处理
├── controller/
│   ├── UserController.java             # 用户登录
│   ├── ShopController.java             # 商户查询
│   ├── VoucherController.java          # 优惠券管理
│   ├── VoucherOrderController.java     # 秒杀下单
│   ├── BlogController.java             # 探店笔记
│   ├── FollowController.java           # 好友关注
│   └── UploadController.java           # 文件上传
├── dto/
│   ├── LoginFormDTO.java               # 登录表单
│   ├── Result.java                     # 统一返回
│   ├── UserDTO.java                    # 用户脱敏DTO
│   └── ScrollResult.java               # 滚动分页结果
├── entity/
│   ├── User.java / Shop.java / Voucher.java
│   ├── SeckillVoucher.java / VoucherOrder.java
│   ├── Blog.java / Follow.java
├── mapper/
│   ├── UserMapper.java / ShopMapper.java
│   └── VoucherOrderMapper.java
├── service/
│   ├── IUserService.java / impl/UserServiceImpl.java
│   ├── IShopService.java / impl/ShopServiceImpl.java
│   ├── IVoucherOrderService.java / impl/VoucherOrderServiceImpl.java
│   ├── IBlogService.java / impl/BlogServiceImpl.java
│   └── IFollowService.java / impl/FollowServiceImpl.java
├── utils/
│   ├── CacheClient.java                # 缓存工具类
│   ├── RedisIdWorker.java              # 全局ID生成器
│   ├── SimpleRedisLock.java            # 简易Redis锁
│   ├── UserHolder.java                 # ThreadLocal工具
│   ├── RegexUtils.java                 # 正则校验
│   └── RedisConstants.java             # Redis常量
└── interceptor/
    ├── RefreshTokenInterceptor.java    # Token刷新拦截器
    └── LoginInterceptor.java           # 登录拦截器
```

### 1.1.3 核心数据表DDL

```sql
-- 用户表
CREATE TABLE tb_user (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    phone VARCHAR(11) NOT NULL UNIQUE,
    nick_name VARCHAR(32) DEFAULT '',
    icon VARCHAR(255) DEFAULT '',
    create_time DATETIME DEFAULT CURRENT_TIMESTAMP,
    update_time DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_phone (phone)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 商户表
CREATE TABLE tb_shop (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(128) NOT NULL,
    type_id BIGINT NOT NULL,
    images VARCHAR(1024) DEFAULT '',
    address VARCHAR(255) DEFAULT '',
    x DOUBLE DEFAULT 0 COMMENT '经度',
    y DOUBLE DEFAULT 0 COMMENT '纬度',
    avg_price BIGINT DEFAULT 0,
    sold INT DEFAULT 0,
    score INT DEFAULT 0,
    create_time DATETIME DEFAULT CURRENT_TIMESTAMP,
    update_time DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_type_id (type_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 秒杀券表
CREATE TABLE tb_seckill_voucher (
    voucher_id BIGINT PRIMARY KEY,
    stock INT NOT NULL,
    begin_time DATETIME NOT NULL,
    end_time DATETIME NOT NULL,
    create_time DATETIME DEFAULT CURRENT_TIMESTAMP,
    update_time DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 订单表（含一人一单唯一索引）
CREATE TABLE tb_voucher_order (
    id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    voucher_id BIGINT NOT NULL,
    status TINYINT DEFAULT 1,
    create_time DATETIME DEFAULT CURRENT_TIMESTAMP,
    pay_time DATETIME DEFAULT NULL,
    UNIQUE KEY uk_user_voucher (user_id, voucher_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 探店笔记表
CREATE TABLE tb_blog (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    shop_id BIGINT DEFAULT 0,
    user_id BIGINT NOT NULL,
    title VARCHAR(255) NOT NULL,
    images VARCHAR(2048) DEFAULT '',
    content TEXT,
    liked INT DEFAULT 0,
    create_time DATETIME DEFAULT CURRENT_TIMESTAMP,
    update_time DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_user_id (user_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 关注表
CREATE TABLE tb_follow (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    follow_user_id BIGINT NOT NULL,
    create_time DATETIME DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_user_follow (user_id, follow_user_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## 1.2 基于session实现短信登陆的流程

- 发送验证码：校验手机号 -> 生成 6 位验证码 -> 保存到 session。
- 登录：校验手机号与验证码 -> 查询/创建用户 -> 保存用户到 session。
- 登录态：浏览器携带 JSESSIONID，服务端从 session 取用户。

### 1.2.1 Session登录流程时序图

```
用户端               Nginx              SpringBoot           Session/Tomcat       MySQL
 |                    |                     |                      |                  |
 |--(1) POST /user/code                    |                      |                  |
 |---(phone=138xxx)--->|                     |                      |                  |
 |                    |--(2) 转发---------->|                      |                  |
 |                    |                     |--(3) 校验手机号      |                  |
 |                    |                     |--(4) 生成6位验证码    |                  |
 |                    |                     |--(5) setAttribute--->|                  |
 |                    |                     |<-(6) 返回成功--------|                  |
 |<-(7) {"success":true}-------------------|                      |                  |
 |                    |                     |                      |                  |
 |--(8) POST /user/login                   |                      |                  |
 |-(phone,code)------>|                     |                      |                  |
 |                    |--(9) 转发----------->|                      |                  |
 |                    |                     |--(10) getAttribute---|                  |
 |                    |                     |<-(11) 返回code-------|                  |
 |                    |                     |--(12) 校验code       |                  |
 |                    |                     |--(13) 查用户by phone----------------->|
 |                    |                     |<-(14) 返回用户/空----------------------|
 |                    |                     |--(15) 不存在则insert-------------------->|
 |                    |                     |--(16) setAttribute-->|                  |
 |<-(17) {"data":UserDTO}-----------------|                      |                  |
```

### 1.2.2 UserController完整代码

```java
package com.hmdp.controller;

import cn.hutool.core.bean.BeanUtil;
import cn.hutool.core.util.RandomUtil;
import com.hmdp.dto.LoginFormDTO;
import com.hmdp.dto.Result;
import com.hmdp.dto.UserDTO;
import com.hmdp.entity.User;
import com.hmdp.service.IUserService;
import com.hmdp.utils.RegexUtils;
import com.hmdp.utils.UserHolder;
import lombok.extern.slf4j.Slf4j;
import org.springframework.web.bind.annotation.*;

import javax.annotation.Resource;
import javax.servlet.http.HttpSession;

@Slf4j
@RestController
@RequestMapping("/user")
public class UserController {

    @Resource
    private IUserService userService;

    @PostMapping("/code")
    public Result sendCode(@RequestParam("phone") String phone, HttpSession session) {
        if (RegexUtils.isPhoneInvalid(phone)) {
            return Result.fail("手机号格式不正确");
        }
        String code = RandomUtil.randomNumbers(6);
        session.setAttribute("login:code:" + phone, code);
        log.info("发送验证码，phone={}, code={}", phone, code);
        return Result.ok();
    }

    @PostMapping("/login")
    public Result login(@RequestBody LoginFormDTO loginForm, HttpSession session) {
        String phone = loginForm.getPhone();
        if (RegexUtils.isPhoneInvalid(phone)) {
            return Result.fail("手机号格式不正确");
        }
        String cacheCode = (String) session.getAttribute("login:code:" + phone);
        if (cacheCode == null || !cacheCode.equals(loginForm.getCode())) {
            return Result.fail("验证码错误");
        }
        User user = userService.lambdaQuery().eq(User::getPhone, phone).one();
        if (user == null) {
            user = new User();
            user.setPhone(phone);
            user.setNickName("user_" + RandomUtil.randomString(8));
            userService.save(user);
        }
        UserDTO userDTO = BeanUtil.copyProperties(user, UserDTO.class);
        session.setAttribute("user", userDTO);
        session.removeAttribute("login:code:" + phone);
        return Result.ok(userDTO);
    }

    @GetMapping("/me")
    public Result me() {
        UserDTO user = UserHolder.getUser();
        if (user == null) { return Result.fail("未登录"); }
        return Result.ok(user);
    }
}
```

## 1.3 实现发送短信验证码的功能

核心逻辑：
```java
String code = RandomUtil.randomNumbers(6);
session.setAttribute("login:code:" + phone, code);
log.debug("sms code = {}", code);
```

- 生产环境通常调用短信服务，这里使用日志模拟。
- 需要设置验证码有效期（session 默认超时）。

### 1.3.1 SmsService完整的验证码服务

```java
package com.hmdp.service;

import cn.hutool.core.util.RandomUtil;
import cn.hutool.core.util.StrUtil;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;
import javax.annotation.Resource;
import java.util.concurrent.TimeUnit;

@Slf4j
@Service
public class SmsService {

    @Resource
    private StringRedisTemplate stringRedisTemplate;

    private static final String CODE_PREFIX = "sms:code:";
    private static final long CODE_EXPIRE_SECONDS = 300L;

    public boolean sendCode(String phone) {
        String code = RandomUtil.randomNumbers(6);
        String codeKey = CODE_PREFIX + phone;
        stringRedisTemplate.opsForValue().set(codeKey, code, CODE_EXPIRE_SECONDS, TimeUnit.SECONDS);
        log.info("短信验证码: phone={}, code={}", phone, code);
        return true;
    }

    public boolean validateCode(String phone, String code) {
        if (StrUtil.isBlank(code)) return false;
        String cachedCode = stringRedisTemplate.opsForValue().get(CODE_PREFIX + phone);
        if (cachedCode == null) return false;
        if (!cachedCode.equals(code)) return false;
        stringRedisTemplate.delete(CODE_PREFIX + phone); // 防止重复使用
        return true;
    }
}
```

### 1.3.2 RegexUtils正则校验工具

```java
package com.hmdp.utils;

import cn.hutool.core.util.StrUtil;

public class RegexUtils {
    public static boolean isPhoneInvalid(String phone) {
        if (StrUtil.isBlank(phone)) return true;
        return !phone.matches("^1[3-9]\\d{9}$");
    }
}
```

### 1.3.3 LoginFormDTO 与 Result

```java
// LoginFormDTO.java
package com.hmdp.dto;
import lombok.Data;
@Data
public class LoginFormDTO {
    private String phone;
    private String code;
    private String password;
}

// Result.java
package com.hmdp.dto;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
import java.util.List;
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Result {
    private Boolean success;
    private String errorMsg;
    private Object data;
    private Long total;
    public static Result ok() { return new Result(true, null, null, null); }
    public static Result ok(Object data) { return new Result(true, null, data, null); }
    public static Result ok(List<?> data, Long total) { return new Result(true, null, data, total); }
    public static Result fail(String errorMsg) { return new Result(false, errorMsg, null, null); }
}
```

## 1.4 实现短信发送验证码登陆和注册的功能

登录流程：
```java
String cacheCode = (String) session.getAttribute("login:code:" + phone);
if (!code.equals(cacheCode)) { return Result.fail("验证码错误"); }

User user = userService.lambdaQuery().eq(User::getPhone, phone).one();
if (user == null) {
		user = new User();
		user.setPhone(phone);
		user.setNickName("user_" + RandomUtil.randomString(8));
		userService.save(user);
}
session.setAttribute("user", user);
```

- 注册逻辑与登录合并：手机号不存在则创建用户。
- 记得清理验证码或设置过期。

### 1.4.1 User实体与UserDTO

```java
// User.java (entity)
package com.hmdp.entity;
import com.baomidou.mybatisplus.annotation.IdType;
import com.baomidou.mybatisplus.annotation.TableId;
import com.baomidou.mybatisplus.annotation.TableName;
import lombok.Data;
import java.time.LocalDateTime;
@Data
@TableName("tb_user")
public class User {
    @TableId(type = IdType.AUTO)
    private Long id;
    private String phone;
    private String password;
    private String nickName;
    private String icon;
    private LocalDateTime createTime;
    private LocalDateTime updateTime;
}

// UserDTO.java (脱敏: 仅暴露id/nickName/icon)
package com.hmdp.dto;
import lombok.Data;
@Data
public class UserDTO {
    private Long id;
    private String nickName;
    private String icon;
}
```

## 1.5 实现登陆校验拦截器

- 自定义拦截器 `LoginInterceptor`：
	- 未登录返回 401 或 `Result.fail("未登录")`。
	- 登录用户写入 `UserHolder`（ThreadLocal）。

```java
public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
		User user = (User) request.getSession().getAttribute("user");
		if (user == null) { response.setStatus(401); return false; }
		UserHolder.saveUser(user);
		return true;
}
```

### 1.5.1 拦截器注册MvcConfig

```java
package com.hmdp.config;

import com.hmdp.interceptor.LoginInterceptor;
import com.hmdp.interceptor.RefreshTokenInterceptor;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;
import javax.annotation.Resource;

@Configuration
public class MvcConfig implements WebMvcConfigurer {

    @Resource
    private StringRedisTemplate stringRedisTemplate;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        // Token刷新拦截器(order=0先执行, 拦截绝大多数请求)
        registry.addInterceptor(new RefreshTokenInterceptor(stringRedisTemplate))
                .addPathPatterns("/**")
                .excludePathPatterns(
                        "/user/code", "/user/login",
                        "/shop/**", "/shop-type/**",
                        "/voucher/**", "/upload/**"
                )
                .order(0);

        // 登录拦截器(order=1, 只拦截需要登录的接口)
        registry.addInterceptor(new LoginInterceptor())
                .addPathPatterns(
                        "/user/me", "/user/info/**",
                        "/voucher-order/**", "/blog/**",
                        "/follow/**"
                )
                .excludePathPatterns("/blog/hot")
                .order(1);
    }
}
```

### 1.5.2 LoginInterceptor完整实现

```java
package com.hmdp.interceptor;

import com.hmdp.dto.UserDTO;
import com.hmdp.utils.UserHolder;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

@Slf4j
@Component
public class LoginInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        UserDTO user = UserHolder.getUser();
        if (user == null) {
            response.setStatus(401);
            log.warn("未登录访问: {}", request.getRequestURI());
            return false;
        }
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                                Object handler, Exception ex) {
        UserHolder.removeUser(); // 清理ThreadLocal，防止内存泄漏
    }
}
```

### 1.5.3 UserHolder完整实现

```java
package com.hmdp.utils;

import com.hmdp.dto.UserDTO;

/**
 * ThreadLocal用户持有者，每个请求线程持有独立用户信息副本
 */
public class UserHolder {
    private static final ThreadLocal<UserDTO> tl = new ThreadLocal<>();

    public static void saveUser(UserDTO user) { tl.set(user); }

    public static UserDTO getUser() { return tl.get(); }

    public static Long getUserId() {
        UserDTO user = tl.get();
        return user != null ? user.getId() : null;
    }

    public static void removeUser() { tl.remove(); }
}
```

## 1.6 隐藏用户敏感信息

- 定义 `UserDTO` 仅保留 `id/nickName/icon`。
- session 或返回给前端使用 `UserDTO`。

```java
UserDTO dto = BeanUtil.copyProperties(user, UserDTO.class);
session.setAttribute("user", dto);
```

## 1.7 session共享的问题分析

- 多节点部署会话不共享。
- 解决方案：
	- 粘性会话（Nginx ip_hash），但容灾差。
	- Session 复制（成本高）。
	- Redis 统一存储（推荐）。（使用spring-session-data-redis）

### 1.7.1 三种Session共享方案对比

| 方案 | 原理 | 优点 | 缺点 | 适用 |
|------|------|------|------|------|
| 粘性会话(ip_hash) | 同一IP路由到同一节点 | 实现简单 | 节点宕机丢失；负载不均 | 临时方案 |
| Session复制 | 节点间广播变更 | 无外部依赖 | 网络开销大；内存膨胀 | 小型集群 |
| Redis集中存储 | Session统一存Redis | 高可用；共享方便 | 依赖Redis；网络延迟 | 生产推荐 |

### 1.7.2 Spring Session + Redis配置

```yaml
# application.yml
spring:
  session:
    store-type: redis
    timeout: 30m
    redis:
      namespace: spring:session
      flush-mode: on_save
  redis:
    host: localhost
    port: 6379
    lettuce:
      pool:
        max-active: 8
        max-idle: 8
        min-idle: 0
```

```java
@Configuration
@EnableRedisHttpSession(maxInactiveIntervalInSeconds = 1800)
public class SessionConfig {
    // Spring Session 自动接管 Session 的 Redis 读写
}
```

## 1.8 Redis代替session的业务流程

- 登录成功生成 `token` 返回前端。
- `token` 作为 key，用户信息作为 hash 存 Redis。
- 前端后续请求在 header 携带 `authorization`。
- 刷新登录态：每次请求续期 TTL。

### 1.8.1 Token登录流程详解

```
用户端                         后端                          Redis
 |                              |                              |
 |--(1) POST /user/code        |                              |
 |   phone=138xxx              |                              |
 |----------------------------->|                              |
 |                              |--(2) 生成code+保存----------->|
 |<-(3) {"success":true}-------|                              |
 |                              |                              |
 |--(4) POST /user/login       |                              |
 |   {phone,code}              |                              |
 |----------------------------->|                              |
 |                              |--(5) 校验code--------------->|
 |                              |<-(6) 校验通过               |
 |                              |--(7) 查/创建用户(MySQL)      |
 |                              |--(8) 生成UUID作为token       |
 |                              |--(9) UserDTO存为Hash-------->|
 |                              |      KEY=login:token:{uuid}  |
 |                              |      TTL=30min               |
 |<-(10) {"data":{"token":...}}|                              |
 |                              |                              |
 |--(11) GET /user/me          |                              |
 |   Header:authorization      |                              |
 |----------------------------->|                              |
 |                              |--(12) 解析token             |
 |                              |--(13) 读Redis--------------->|
 |                              |<-(14) 返回UserDTO           |
 |                              |--(15) 刷新TTL=30min--------->|
 |<-(16) {"data":UserDTO}------|                              |
```

### 1.8.2 Redis序列化配置

```java
package com.hmdp.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializer;

@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        GenericJackson2JsonRedisSerializer jacksonSerializer =
                new GenericJackson2JsonRedisSerializer();
        template.setKeySerializer(RedisSerializer.string());
        template.setHashKeySerializer(RedisSerializer.string());
        template.setValueSerializer(jacksonSerializer);
        template.setHashValueSerializer(jacksonSerializer);
        template.afterPropertiesSet();
        return template;
    }
}
```

## 1.9 基于Redis实现短信登陆

保存用户信息到 Redis：
```java
String token = UUID.randomUUID().toString(true);
UserDTO dto = BeanUtil.copyProperties(user, UserDTO.class);
Map<String, Object> map = BeanUtil.beanToMap(dto, new HashMap<>(),
		CopyOptions.create().setIgnoreNullValue(true).setFieldValueEditor((f, v) -> v.toString()));

String tokenKey = "login:token:" + token;
stringRedisTemplate.opsForHash().putAll(tokenKey, map);
stringRedisTemplate.expire(tokenKey, Duration.ofMinutes(30));
```

读取登录用户（拦截器）：
```java
String token = request.getHeader("authorization");
Map<Object, Object> userMap = stringRedisTemplate.opsForHash().entries("login:token:" + token);
UserDTO user = BeanUtil.fillBeanWithMap(userMap, new UserDTO(), false);
```

### 1.9.1 UserServiceImpl完整Redis登录实现

```java
package com.hmdp.service.impl;

import cn.hutool.core.bean.BeanUtil;
import cn.hutool.core.bean.copier.CopyOptions;
import cn.hutool.core.lang.UUID;
import cn.hutool.core.util.RandomUtil;
import com.baomidou.mybatisplus.extension.service.impl.ServiceImpl;
import com.hmdp.dto.LoginFormDTO;
import com.hmdp.dto.Result;
import com.hmdp.dto.UserDTO;
import com.hmdp.entity.User;
import com.hmdp.mapper.UserMapper;
import com.hmdp.service.IUserService;
import com.hmdp.utils.RegexUtils;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;
import javax.annotation.Resource;
import java.time.Duration;
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.TimeUnit;

@Slf4j
@Service
public class UserServiceImpl extends ServiceImpl<UserMapper, User> implements IUserService {

    @Resource
    private StringRedisTemplate stringRedisTemplate;

    private static final String CODE_PREFIX = "login:code:";
    private static final String TOKEN_PREFIX = "login:token:";
    private static final long CODE_TTL = 5L;
    private static final long TOKEN_TTL = 30L;

    @Override
    public Result sendCode(String phone) {
        if (RegexUtils.isPhoneInvalid(phone)) return Result.fail("手机号格式错误");
        String code = RandomUtil.randomNumbers(6);
        stringRedisTemplate.opsForValue().set(
                CODE_PREFIX + phone, code, CODE_TTL, TimeUnit.MINUTES);
        log.info("发送验证码: phone={}, code={}", phone, code);
        return Result.ok();
    }

    @Override
    public Result login(LoginFormDTO loginForm) {
        String phone = loginForm.getPhone();
        if (RegexUtils.isPhoneInvalid(phone)) return Result.fail("手机号格式错误");
        // 从Redis校验验证码
        String cacheCode = stringRedisTemplate.opsForValue().get(CODE_PREFIX + phone);
        if (cacheCode == null || !cacheCode.equals(loginForm.getCode())) {
            return Result.fail("验证码错误或已过期");
        }
        // 查/创建用户
        User user = lambdaQuery().eq(User::getPhone, phone).one();
        if (user == null) {
            user = new User();
            user.setPhone(phone);
            user.setNickName("user_" + RandomUtil.randomString(8));
            save(user);
        }
        // 保存到Redis(Hash结构)，返回token
        String token = UUID.randomUUID().toString(true);
        UserDTO userDTO = BeanUtil.copyProperties(user, UserDTO.class);
        Map<String, Object> userMap = BeanUtil.beanToMap(userDTO, new HashMap<>(),
                CopyOptions.create()
                        .setIgnoreNullValue(true)
                        .setFieldValueEditor((fn, fv) -> fv.toString()));
        String tokenKey = TOKEN_PREFIX + token;
        stringRedisTemplate.opsForHash().putAll(tokenKey, userMap);
        stringRedisTemplate.expire(tokenKey, Duration.ofMinutes(TOKEN_TTL));
        stringRedisTemplate.delete(CODE_PREFIX + phone);
        // 返回token
        Map<String, String> res = new HashMap<>();
        res.put("token", token);
        return Result.ok(res);
    }
}
```

## 1.10 解决状态登陆刷新的问题

- 增加 `RefreshTokenInterceptor`：
	- 只要携带 `token`，就从 Redis 刷新 TTL。
	- 若存在用户信息则写入 `UserHolder`。
- `LoginInterceptor` 只拦截需要登录的接口。
- 拦截器顺序：刷新拦截器在前，登录拦截器在后。

### 1.10.1 RefreshTokenInterceptor完整实现

```java
package com.hmdp.interceptor;

import cn.hutool.core.bean.BeanUtil;
import cn.hutool.core.util.StrUtil;
import com.hmdp.dto.UserDTO;
import com.hmdp.utils.UserHolder;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.web.servlet.HandlerInterceptor;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.util.Map;
import java.util.concurrent.TimeUnit;

public class RefreshTokenInterceptor implements HandlerInterceptor {

    private final StringRedisTemplate stringRedisTemplate;

    public RefreshTokenInterceptor(StringRedisTemplate stringRedisTemplate) {
        this.stringRedisTemplate = stringRedisTemplate;
    }

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        // 1. 从请求头获取token
        String token = request.getHeader("authorization");
        if (StrUtil.isBlank(token)) return true; // 无token直接放行
        // 2. 从Redis获取用户
        String tokenKey = "login:token:" + token;
        Map<Object, Object> userMap = stringRedisTemplate.opsForHash().entries(tokenKey);
        if (userMap.isEmpty()) return true;
        // 3. Map转UserDTO存入ThreadLocal
        UserDTO userDTO = BeanUtil.fillBeanWithMap(userMap, new UserDTO(), false);
        UserHolder.saveUser(userDTO);
        // 4. 刷新token有效期(每次请求续期)
        stringRedisTemplate.expire(tokenKey, 30, TimeUnit.MINUTES);
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                                Object handler, Exception ex) {
        UserHolder.removeUser(); // 防内存泄漏
    }
}
```

### 1.10.2 RedisConstants常量类

```java
package com.hmdp.utils;

public class RedisConstants {
    public static final String LOGIN_CODE_KEY = "login:code:";
    public static final Long LOGIN_CODE_TTL = 5L;
    public static final String LOGIN_TOKEN_KEY = "login:token:";
    public static final Long LOGIN_TOKEN_TTL = 30L;

    public static final String CACHE_SHOP_KEY = "cache:shop:";
    public static final String CACHE_SHOP_TYPE_KEY = "cache:shop-type:";
    public static final Long CACHE_SHOP_TTL = 30L;
    public static final Long CACHE_NULL_TTL = 2L;

    public static final String LOCK_SHOP_KEY = "lock:shop:";
    public static final Long LOCK_SHOP_TTL = 10L;

    public static final String SECKILL_STOCK_KEY = "seckill:stock:";
    public static final String SECKILL_USER_KEY = "seckill:user:";
    public static final String SECKILL_ORDER_KEY = "seckill:order:";
    public static final String SECKILL_STREAM_KEY = "stream.orders";

    public static final String BLOG_LIKED_KEY = "blog:liked:";
    public static final String FEED_KEY = "feed:";
    public static final String FOLLOW_KEY = "follow:";
    public static final String SHOP_GEO_KEY = "shop:geo:";
}
```

## 1.11 Redis实现登录限流

### 1.11.1 短信验证码频控（INCR+EXPIRE）

```java
package com.hmdp.service;

import cn.hutool.core.util.RandomUtil;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.core.script.DefaultRedisScript;
import org.springframework.stereotype.Service;
import javax.annotation.Resource;
import java.util.Collections;
import java.util.concurrent.TimeUnit;

@Slf4j
@Service
public class RateLimiterService {

    @Resource
    private StringRedisTemplate stringRedisTemplate;

    /** 同一手机号1分钟内最多3次 */
    public boolean allowSendSms(String phone) {
        String rateKey = "rate:sms:" + phone;
        Long count = stringRedisTemplate.opsForValue().increment(rateKey);
        if (count != null && count == 1) {
            stringRedisTemplate.expire(rateKey, 60, TimeUnit.SECONDS);
        }
        if (count != null && count > 3) {
            log.warn("短信频率超限: phone={}", phone);
            return false;
        }
        return true;
    }

    /** 同一账号10分钟内最多5次登录失败 */
    public boolean allowLoginAttempt(String phone) {
        String failKey = "login:fail:" + phone;
        Long count = stringRedisTemplate.opsForValue().increment(failKey);
        if (count != null && count == 1) {
            stringRedisTemplate.expire(failKey, 600, TimeUnit.SECONDS);
        }
        return count == null || count <= 5;
    }

    /** 登录成功后重置失败计数 */
    public void resetLoginAttempt(String phone) {
        stringRedisTemplate.delete("login:fail:" + phone);
    }

    /** 获取剩余重试次数 */
    public long getRemainingAttempts(String phone) {
        String countStr = stringRedisTemplate.opsForValue().get("login:fail:" + phone);
        long count = countStr != null ? Long.parseLong(countStr) : 0;
        return Math.max(0, 5 - count);
    }
}
```

### 1.11.2 Lua脚本原子化INCR+EXPIRE

```lua
-- rate_limit.lua
-- KEYS[1]: 限流key  ARGV[1]: 阈值  ARGV[2]: 过期时间(秒)
-- 返回: 1=允许, 0=拒绝
local count = redis.call('INCR', KEYS[1])
if count == 1 then
    redis.call('EXPIRE', KEYS[1], ARGV[2])
end
if count > tonumber(ARGV[1]) then
    return 0
end
return 1
```

```java
public boolean allowSendSmsLua(String phone) {
    String script =
        "local count = redis.call('INCR', KEYS[1]) " +
        "if count == 1 then redis.call('EXPIRE', KEYS[1], ARGV[2]) end " +
        "if count > tonumber(ARGV[1]) then return 0 else return 1 end";
    DefaultRedisScript<Long> redisScript = new DefaultRedisScript<>();
    redisScript.setScriptText(script);
    redisScript.setResultType(Long.class);
    Long result = stringRedisTemplate.execute(redisScript,
            Collections.singletonList("rate:sms:" + phone), "3", "60");
    return result != null && result == 1L;
}
```

# 2. 商户查询缓存

## 2.1 什么是缓存

- 用于减少 DB 访问，提高读性能。
- 典型模式：Cache Aside（旁路缓存）。

### 2.1.1 Cache Aside模式详解

```
         [ 应用层 ]         [ Redis ]         [ MySQL ]
              |                  |                 |
         【读请求】              |                 |
              +-1.查缓存------->|                 |
              |<-2.miss---------|                 |
              +-3.查DB-------------------------->|
              |<-4.返回数据-----------------------|
              +-5.写回缓存---->|                 |
              +-6.返回前端      |                 |
              |                  |                 |
         【写请求】              |                 |
              +-1.更新DB------------------------->|
              |<-2.完成---------------------------|
              +-3.删除缓存---->|                 |

核心原则:
  读: 先查缓存, miss则查DB并回写
  写: 先更新DB, 再删除缓存(非更新)
  为什么先DB后缓存? 防止读请求在DB更新前将旧数据写回缓存
```

### 2.1.2 四种缓存更新策略对比

| 策略 | 原理 | 一致性 | 复杂度 | 适用 |
|------|------|--------|--------|------|
| Cache-Aside | 应用负责读写, 写时更新DB+删缓存 | 最终一致 | 低 | 读多写少 |
| Read-Through | 缓存层代理读, miss自动加载 | 最终一致 | 中 | 有缓存抽象层 |
| Write-Through | 缓存层代理写, 同步写DB | 强一致 | 中 | 写少, 要一致 |
| Write-Behind | 异步批量写DB | 弱一致 | 高 | 写多, 容忍不一致 |

## 2.2 添加商户缓存

查询流程：
```java
String key = "cache:shop:" + id;
String json = stringRedisTemplate.opsForValue().get(key);
if (StrUtil.isNotBlank(json)) {
		return JSONUtil.toBean(json, Shop.class);
}
Shop shop = getById(id);
stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(shop), 30, TimeUnit.MINUTES);
```

### 2.2.1 ShopController + ShopService完整实现

```java
// ShopController.java
package com.hmdp.controller;
import com.hmdp.dto.Result;
import com.hmdp.entity.Shop;
import com.hmdp.service.IShopService;
import org.springframework.web.bind.annotation.*;
import javax.annotation.Resource;

@RestController
@RequestMapping("/shop")
public class ShopController {
    @Resource
    private IShopService shopService;

    @GetMapping("/{id}")
    public Result queryShopById(@PathVariable("id") Long id) {
        return shopService.queryById(id);
    }

    @PostMapping
    public Result save(@RequestBody Shop shop) {
        shopService.save(shop);
        return Result.ok(shop.getId());
    }

    @PutMapping
    public Result update(@RequestBody Shop shop) {
        return shopService.update(shop);
    }
}

// ShopServiceImpl.java
package com.hmdp.service.impl;
import cn.hutool.json.JSONUtil;
import cn.hutool.core.util.StrUtil;
import com.baomidou.mybatisplus.extension.service.impl.ServiceImpl;
import com.hmdp.dto.Result;
import com.hmdp.entity.Shop;
import com.hmdp.mapper.ShopMapper;
import com.hmdp.service.IShopService;
import com.hmdp.utils.CacheClient;
import com.hmdp.utils.RedisConstants;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import javax.annotation.Resource;
import java.util.concurrent.TimeUnit;

@Service
public class ShopServiceImpl extends ServiceImpl<ShopMapper, Shop> implements IShopService {

    @Resource
    private StringRedisTemplate stringRedisTemplate;

    @Resource
    private CacheClient cacheClient;

    @Override
    public Result queryById(Long id) {
        // 使用CacheClient的穿透保护查询
        Shop shop = cacheClient.queryWithPassThrough(
                RedisConstants.CACHE_SHOP_KEY, id, Shop.class,
                this::getById, RedisConstants.CACHE_SHOP_TTL, TimeUnit.MINUTES);
        if (shop == null) return Result.fail("店铺不存在");
        return Result.ok(shop);
    }

    @Override
    @Transactional
    public Result update(Shop shop) {
        Long id = shop.getId();
        if (id == null) return Result.fail("id不能为空");
        updateById(shop);  // 先更新DB
        stringRedisTemplate.delete(RedisConstants.CACHE_SHOP_KEY + id); // 再删缓存
        return Result.ok();
    }
}
```

## 2.3 缓存练习题分析

- 关注缓存穿透、缓存雪崩、缓存击穿。
- 结合业务和访问量选择策略。

## 2.4 缓存更新策略

- 最常用：先更新数据库，再删除缓存。
- 可配合消息队列保证最终一致。

### 2.4.1 先删缓存再更新DB vs 先更新DB再删缓存

```
【先删缓存再更新DB(延时双删)】
  T1: 删除缓存
  T2: 更新DB
  T3: Thread.sleep(500ms)
  T4: 再次删除缓存
  问题: T1-T2间隙有读请求把旧数据写回缓存
  解决: T4的二次删除保证最终一致

【先更新DB再删缓存(推荐)】
  T1: 更新DB
  T2: 删除缓存
  问题: T1-T2间隙读到缓存中的旧数据
  概率: 极低(读操作必须在写删缓存之前完成)
  优点: 实现简单, 应用最广
```

### 2.4.2 延时双删实现

```java
@Override
@Transactional
public Result updateWithDelayDoubleDelete(Shop shop) {
    Long id = shop.getId();
    stringRedisTemplate.delete("cache:shop:" + id); // 1.先删
    updateById(shop);                                  // 2.更新DB
    CompletableFuture.runAsync(() -> {                 // 3.延时再删
        try { Thread.sleep(500); } catch (Exception ignored) {}
        stringRedisTemplate.delete("cache:shop:" + id);
    });
    return Result.ok();
}
```

## 2.5 实现商铺缓存与数据库的双写一致

更新逻辑：
```java
updateById(shop);
stringRedisTemplate.delete("cache:shop:" + shop.getId());
```

- 删除比更新更安全，避免并发读写产生脏数据。

## 2.6 缓存穿透问题及解决思路

问题：请求的数据既不在缓存中，也不在数据库中，导致每次请求都会直接打到数据库。

- 空值缓存（短 TTL）。
- 布隆过滤器（海量 key）。
- 参数校验（如 id 合法性）。

空值缓存示例：
```java
if (shop == null) {
		stringRedisTemplate.opsForValue().set(key, "", 2, TimeUnit.MINUTES);
		return null;
}
```

### 2.6.1 Guava布隆过滤器

```java
package com.hmdp.config;
import com.google.common.hash.BloomFilter;
import com.google.common.hash.Funnels;
import com.hmdp.entity.Shop;
import com.hmdp.service.IShopService;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import javax.annotation.Resource;
import java.nio.charset.StandardCharsets;
import java.util.List;

@Slf4j
@Configuration
public class BloomFilterConfig {
    @Resource
    private IShopService shopService;
    private static final int EXPECTED_INSERTIONS = 1_000_000;
    private static final double FPP = 0.0001; // 误判率0.01%

    @Bean
    public BloomFilter<String> shopBloomFilter() {
        BloomFilter<String> filter = BloomFilter.create(
                Funnels.stringFunnel(StandardCharsets.UTF_8), EXPECTED_INSERTIONS, FPP);
        List<Shop> shops = shopService.list();
        for (Shop shop : shops) {
            filter.put("shop:" + shop.getId());
        }
        log.info("布隆过滤器初始化: {}条", shops.size());
        return filter;
    }
}

// 使用布隆过滤器查询
public Shop queryWithBloomFilter(Long id) {
    if (!shopBloomFilter.mightContain("shop:" + id)) return null; // 一定不存在
    String key = "cache:shop:" + id;
    String json = stringRedisTemplate.opsForValue().get(key);
    if (StrUtil.isNotBlank(json)) return JSONUtil.toBean(json, Shop.class);
    Shop shop = getById(id);
    if (shop == null) {
        stringRedisTemplate.opsForValue().set(key, "", 2, TimeUnit.MINUTES);
        return null;
    }
    stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(shop), 30, TimeUnit.MINUTES);
    return shop;
}
```

### 2.6.2 Redisson布隆过滤器

```java
@Bean
public RBloomFilter<Long> shopRedissonBloomFilter(RedissonClient redissonClient) {
    RBloomFilter<Long> bloomFilter = redissonClient.getBloomFilter("bloom:shop");
    bloomFilter.tryInit(1_000_000L, 0.0001);
    return bloomFilter;
}
```

## 2.7 缓存雪崩问题及解决思路

问题：大量 key 同时过期或 Redis 宕机。

- 解决：
  - TTL 随机化。
  - 重要数据预热。
  - 多级缓存或降级。

### 2.7.1 TTL随机化

```java
package com.hmdp.utils;
import java.util.Random;
import java.util.concurrent.TimeUnit;

public class TtlRandomizer {
    private static final Random RANDOM = new Random();

    /** 基础时间的正负20%范围内随机, 防止雪崩 */
    public static long randomize(long baseTimeout, TimeUnit unit) {
        long baseSeconds = unit.toSeconds(baseTimeout);
        long min = (long)(baseSeconds * 0.8);
        long max = (long)(baseSeconds * 1.2);
        return min + (long)(RANDOM.nextDouble() * (max - min));
    }
}
// 使用:
// long ttl = TtlRandomizer.randomize(30, TimeUnit.MINUTES);
// stringRedisTemplate.opsForValue().set(key, value, ttl, TimeUnit.SECONDS);
```

### 2.7.2 缓存预热服务 (@PostConstruct)

```java
package com.hmdp.service;
import com.hmdp.entity.Shop;
import com.hmdp.utils.RedisConstants;
import com.hmdp.utils.TtlRandomizer;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;
import javax.annotation.PostConstruct;
import javax.annotation.Resource;
import java.util.List;
import java.util.concurrent.TimeUnit;
import static cn.hutool.json.JSONUtil.toJsonStr;

@Slf4j
@Service
public class CacheWarmUpService {
    @Resource
    private StringRedisTemplate stringRedisTemplate;
    @Resource
    private com.hmdp.mapper.ShopMapper shopMapper;

    @PostConstruct
    public void warmUpHotData() {
        long start = System.currentTimeMillis();
        // 预热热门商品 Top 1000
        List<Shop> hotShops = shopMapper.selectList(
                new com.baomidou.mybatisplus.core.conditions.query.LambdaQueryWrapper<Shop>()
                        .orderByDesc(Shop::getScore).last("LIMIT 1000"));
        for (Shop shop : hotShops) {
            String key = RedisConstants.CACHE_SHOP_KEY + shop.getId();
            long ttl = TtlRandomizer.randomize(30, TimeUnit.MINUTES);
            stringRedisTemplate.opsForValue().set(key, toJsonStr(shop), ttl, TimeUnit.SECONDS);
        }
        log.info("缓存预热完成: {}条, 耗时{}ms", hotShops.size(), System.currentTimeMillis() - start);
    }
}
```

### 2.7.3 多级缓存降级方案

```java
// L1: Caffeine本地缓存, L2: Redis, L3: MySQL
// 当Redis不可用时自动降级到Caffeine或直接查DB
import com.github.benmanes.caffeine.cache.Cache;
import com.github.benmanes.caffeine.cache.Caffeine;

@Service
public class MultiLevelCacheService {
    private final Cache<String, String> localCache = Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(5, TimeUnit.MINUTES)
            .build();
    @Resource private StringRedisTemplate stringRedisTemplate;

    public Shop query(Long id) {
        String key = "cache:shop:" + id;
        String json = localCache.getIfPresent(key);
        if (json != null) return JSONUtil.toBean(json, Shop.class);
        try {
            json = stringRedisTemplate.opsForValue().get(key);
            if (json != null) { localCache.put(key, json); return JSONUtil.toBean(json, Shop.class); }
        } catch (Exception e) { log.error("Redis不可用,降级查DB"); }
        Shop shop = getById(id);
        if (shop != null) {
            json = JSONUtil.toJsonStr(shop);
            localCache.put(key, json);
            try { stringRedisTemplate.opsForValue().set(key, json, 30, TimeUnit.MINUTES); }
            catch (Exception ignored) {}
        }
        return shop;
    }
}
```

## 2.8 缓存击穿问题及解决方案

问题：热点 key 失效，瞬时高并发打到 DB。

- 解决：互斥锁、逻辑过期。

### 2.8.1 利用互斥锁解决缓存击穿问题

```java
String lockKey = "lock:shop:" + id;
boolean isLock = stringRedisTemplate.opsForValue().setIfAbsent(lockKey, "1", 10, TimeUnit.SECONDS);
if (!isLock) { Thread.sleep(50); return queryById(id); }
try {
		// double check
		String json = stringRedisTemplate.opsForValue().get(key);
		if (StrUtil.isNotBlank(json)) { return JSONUtil.toBean(json, Shop.class); }
		Shop shop = getById(id);
		stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(shop), 30, TimeUnit.MINUTES);
		return shop;
} finally {
		stringRedisTemplate.delete(lockKey);
}
```

### 2.8.2 利用逻辑过期解决缓存击穿问题

- 缓存中存 `data + expireTime`，永不过期。
- 过期后由异步线程重建。

```java
RedisData redisData = new RedisData();
redisData.setData(shop);
redisData.setExpireTime(LocalDateTime.now().plusMinutes(30));
stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(redisData));
```

### 2.8.3 互斥锁 vs 逻辑过期 对比

| 维度 | 互斥锁方案 | 逻辑过期方案 |
|------|-----------|-------------|
| 原理 | miss加锁, 获锁线程重建 | 永不过期, 过期后异步重建 |
| 优点 | 简单; 一致性高 | 高可用; 不阻塞读 |
| 缺点 | 可能死锁; 读阻塞 | 复杂; 可能返回旧数据 |
| 一致性 | 强一致 | 最终一致 |
| 性能(高并发) | 一般 | 优秀 |
| 适用 | 一致性要求高 | 热点数据, 可接受旧数据 |

### 2.8.4 线程池配置(异步重建)

```java
package com.hmdp.config;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import java.util.concurrent.*;

@Configuration
public class ThreadPoolConfig {
    @Bean("cacheRebuildExecutor")
    public ThreadPoolExecutor cacheRebuildExecutor() {
        return new ThreadPoolExecutor(
                4, 8, 60, TimeUnit.SECONDS,
                new LinkedBlockingQueue<>(200),
                r -> new Thread(r, "cache-rebuild-" + Thread.currentThread().getId()),
                new ThreadPoolExecutor.CallerRunsPolicy()
        );
    }
}
```

### 2.8.5 逻辑过期数据包装+完整查询

```java
// RedisData.java - 缓存数据包装类
package com.hmdp.dto;
import lombok.Data;
import java.time.LocalDateTime;

@Data
public class RedisData {
    private Object data;
    private LocalDateTime expireTime;
}

// 逻辑过期查询
public Shop queryWithLogicalExpire(Long id) {
    String key = "cache:shop:" + id;
    String json = stringRedisTemplate.opsForValue().get(key);
    if (StrUtil.isBlank(json)) return null;

    RedisData redisData = JSONUtil.toBean(json, RedisData.class);
    Shop shop = JSONUtil.toBean((cn.hutool.json.JSONObject)redisData.getData(), Shop.class);
    LocalDateTime expireTime = redisData.getExpireTime();

    if (expireTime.isAfter(LocalDateTime.now())) return shop; // 未过期

    // 已过期, 尝试获取互斥锁, 异步重建
    String lockKey = "lock:shop:" + id;
    if (tryLock(lockKey)) {
        threadPool.execute(() -> {
            try {
                Shop newShop = getById(id);
                RedisData newRedisData = new RedisData();
                newRedisData.setData(newShop);
                newRedisData.setExpireTime(LocalDateTime.now().plusMinutes(30));
                stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(newRedisData));
            } finally {
                unlock(lockKey);
            }
        });
    }
    return shop; // 返回旧数据保证可用性
}
```

## 2.9 封装Redistribution工具类

- 工具类 `CacheClient` 统一缓存操作：
	- `set()`、`setWithLogicalExpire()`。
	- `queryWithPassThrough()` 解决穿透。
	- `queryWithMutex()` 解决击穿。
- 内部使用线程池异步重建。

### 2.9.1 完整CacheClient工具类（4种策略）

```java
package com.hmdp.utils;

import cn.hutool.core.util.StrUtil;
import cn.hutool.json.JSONObject;
import cn.hutool.json.JSONUtil;
import com.hmdp.dto.RedisData;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Component;
import javax.annotation.Resource;
import java.time.LocalDateTime;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.TimeUnit;
import java.util.function.Function;

@Slf4j
@Component
public class CacheClient {

    @Resource
    private StringRedisTemplate stringRedisTemplate;
    @Resource(name = "cacheRebuildExecutor")
    private ExecutorService cacheRebuildExecutor;

    /** [策略一] 穿透保护(空值缓存) - 适合防恶意请求 */
    public <R, ID> R queryWithPassThrough(
            String keyPrefix, ID id, Class<R> type,
            Function<ID, R> dbFallback, Long time, TimeUnit unit) {
        String key = keyPrefix + id;
        String json = stringRedisTemplate.opsForValue().get(key);
        if (StrUtil.isNotBlank(json)) return JSONUtil.toBean(json, type);
        if (json != null) return null; // 命中空值缓存
        R r = dbFallback.apply(id);
        if (r == null) {
            stringRedisTemplate.opsForValue().set(key, "", RedisConstants.CACHE_NULL_TTL, TimeUnit.MINUTES);
            return null;
        }
        long ttl = TtlRandomizer.randomize(time, unit);
        stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(r), ttl, TimeUnit.SECONDS);
        return r;
    }

    /** [策略二] 互斥锁防击穿 - 适合一致性要求高 */
    public <R, ID> R queryWithMutex(
            String keyPrefix, ID id, Class<R> type,
            Function<ID, R> dbFallback, Long time, TimeUnit unit) {
        String key = keyPrefix + id;
        String json = stringRedisTemplate.opsForValue().get(key);
        if (StrUtil.isNotBlank(json)) return JSONUtil.toBean(json, type);
        if (json != null) return null;
        String lockKey = "lock:cache:" + key;
        R r = null;
        try {
            while (!tryLock(lockKey)) {
                Thread.sleep(50);
                json = stringRedisTemplate.opsForValue().get(key);
                if (StrUtil.isNotBlank(json)) return JSONUtil.toBean(json, type);
            }
            json = stringRedisTemplate.opsForValue().get(key); // 双检
            if (StrUtil.isNotBlank(json)) return JSONUtil.toBean(json, type);
            r = dbFallback.apply(id);
            if (r == null) {
                stringRedisTemplate.opsForValue().set(key, "", RedisConstants.CACHE_NULL_TTL, TimeUnit.MINUTES);
                return null;
            }
            long ttl = TtlRandomizer.randomize(time, unit);
            stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(r), ttl, TimeUnit.SECONDS);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally { unlock(lockKey); }
        return r;
    }

    /** [策略三] 逻辑过期防击穿 - 适合热点数据高并发 */
    public <R, ID> R queryWithLogicalExpire(
            String keyPrefix, ID id, Class<R> type,
            Function<ID, R> dbFallback, Long time, TimeUnit unit) {
        String key = keyPrefix + id;
        String json = stringRedisTemplate.opsForValue().get(key);
        if (StrUtil.isBlank(json)) {
            R r = dbFallback.apply(id);
            if (r != null) setWithLogicalExpire(keyPrefix, id, r, time, unit);
            return r;
        }
        RedisData redisData = JSONUtil.toBean(json, RedisData.class);
        R r = JSONUtil.toBean((JSONObject) redisData.getData(), type);
        if (redisData.getExpireTime().isAfter(LocalDateTime.now())) return r;
        String lockKey = "lock:cache:" + key;
        if (tryLock(lockKey)) {
            cacheRebuildExecutor.execute(() -> {
                try {
                    R newData = dbFallback.apply(id);
                    if (newData != null) setWithLogicalExpire(keyPrefix, id, newData, time, unit);
                } finally { unlock(lockKey); }
            });
        }
        return r;
    }

    /** [策略四] 写入逻辑过期数据 */
    public <T> void setWithLogicalExpire(String keyPrefix, Long id, T value, Long time, TimeUnit unit) {
        RedisData redisData = new RedisData();
        redisData.setData(value);
        redisData.setExpireTime(LocalDateTime.now().plusSeconds(unit.toSeconds(time)));
        stringRedisTemplate.opsForValue().set(keyPrefix + id, JSONUtil.toJsonStr(redisData));
    }

    // 锁工具
    private boolean tryLock(String key) {
        return Boolean.TRUE.equals(
                stringRedisTemplate.opsForValue().setIfAbsent(key, "1", 10, TimeUnit.SECONDS));
    }
    private void unlock(String key) { stringRedisTemplate.delete(key); }
}
```

## 2.10 缓存总结

- 先用穿透方案兜底，再处理击穿和雪崩。
- 更新策略以"更新 DB + 删除缓存"为主。
- 热点数据可采用逻辑过期 + 异步重建。

## 2.11 缓存性能测试

### 2.11.1 简单并发压测代码

```java
@SpringBootTest
public class CachePerformanceTest {
    @Resource private CacheClient cacheClient;

    @Test
    public void comparePerformance() throws Exception {
        int threadCount = 100;
        int reqPerThread = 100;
        CountDownLatch latch = new CountDownLatch(threadCount);
        AtomicInteger success = new AtomicInteger();
        ExecutorService executor = Executors.newFixedThreadPool(threadCount);
        long start = System.currentTimeMillis();

        for (int i = 0; i < threadCount; i++) {
            final long shopId = i % 10 + 1L;
            executor.submit(() -> {
                try {
                    for (int j = 0; j < reqPerThread; j++) {
                        Shop shop = cacheClient.queryWithPassThrough(
                                "cache:shop:", shopId, Shop.class,
                                id -> null, 30L, TimeUnit.MINUTES);
                        if (shop != null) success.incrementAndGet();
                    }
                } finally { latch.countDown(); }
            });
        }
        latch.await(60, TimeUnit.SECONDS);
        executor.shutdown();
        long cost = System.currentTimeMillis() - start;
        int total = threadCount * reqPerThread;
        System.out.println("总请求: " + total + ", 成功: " + success.get()
                + ", 耗时: " + cost + "ms, QPS: " + (total * 1000L / cost));
    }
}
```

### 2.11.2 压测结果参考

```
┌─────────────────────────┬──────────┬──────────┬──────────┐
│ 场景                    │ QPS      │ Avg RT   │ Error%   │
├─────────────────────────┼──────────┼──────────┼──────────┤
│ 直查MySQL               │ ~500     │ ~2000ms  │ 10%      │
│ Redis缓存(无保护)        │ ~5000    │ ~200ms   │ 20%      │
│ +穿透保护                │ ~8000    │ ~120ms   │ 0%       │
│ +互斥锁防击穿            │ ~7000    │ ~140ms   │ 0%       │
│ +逻辑过期防击穿          │ ~10000   │ ~100ms   │ 0%       │
└─────────────────────────┴──────────┴──────────┴──────────┘
```

### 2.11.3 JMeter测试计划描述

```
JMeter: 商户查询压测
├── Thread Group: 1000线程, Ramp-Up 10s, 持续300s
├── CSV: shop_ids.csv(50%热点, 30%冷门, 20%不存在)
├── HTTP Request: GET /shop/${shop_id}
├── Summary Report (关注 Throughput, Avg, Error%)
└── Backend Listener -> InfluxDB + Grafana 监控
```

# 3. 优惠券秒杀

## 3.1 全局唯一ID

- 订单号需要全局唯一、趋势递增、可分库分表。
- 方案：Redis 自增 + 时间戳。

### 3.1.1 分布式ID方案对比

| 方案 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| UUID | 本地128位 | 极简单 | 无序, 索引差 |
| 雪花算法 | 机器ID+时间戳+序列号 | 趋势递增, 本地生成 | 依赖时钟 |
| Redis自增 | INCRBY | 简单, 递增 | 依赖Redis |
| 号段模式 | DB分配号段 | 可容灾 | DB瓶颈 |
| 美团Leaf | 雪花+号段双模式 | 高可用, 监控完善 | 运维成本高 |

## 3.2 Redis实现全局唯一ID

```java
public long nextId(String keyPrefix) {
		long timestamp = LocalDateTime.now().toEpochSecond(ZoneOffset.UTC) - BEGIN_TIMESTAMP;
		String date = LocalDate.now().format(DateTimeFormatter.ofPattern("yyyy:MM:dd"));
		long count = stringRedisTemplate.opsForValue().increment("icr:" + keyPrefix + ":" + date);
		return (timestamp << 32) | count;
}
```

### 3.2.1 完整RedisIdWorker实现

```java
package com.hmdp.utils;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Component;
import javax.annotation.Resource;
import java.time.LocalDateTime;
import java.time.ZoneOffset;
import java.time.format.DateTimeFormatter;

/**
 * ID结构: 64位 = 31位时间戳(秒) + 32位序列号 + 1位符号位(0)
 * 每天重置序列号, 理论上每秒可生成2^32个ID
 */
@Component
public class RedisIdWorker {
    @Resource
    private StringRedisTemplate stringRedisTemplate;
    private static final long BEGIN_TIMESTAMP = 1672531200L; // 2023-01-01 00:00:00 UTC
    private static final int COUNT_BITS = 32;

    public long nextId(String keyPrefix) {
        long now = LocalDateTime.now().toEpochSecond(ZoneOffset.UTC);
        long timestamp = now - BEGIN_TIMESTAMP;
        String date = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy:MM:dd"));
        long count = stringRedisTemplate.opsForValue().increment("icr:" + keyPrefix + ":" + date);
        return (timestamp << COUNT_BITS) | count;
    }
}
```

## 3.3 添加优惠券

- `tb_voucher` 保存优惠券基础信息。
- `tb_seckill_voucher` 保存秒杀库存、开始结束时间。
- 秒杀库存同步到 Redis。

### 3.3.1 VoucherController + VoucherService

```java
// VoucherController.java
@RestController
@RequestMapping("/voucher")
public class VoucherController {
    @Resource
    private IVoucherService voucherService;
    @Resource
    private IVoucherOrderService voucherOrderService;

    @PostMapping
    public Result addVoucher(@RequestBody Voucher voucher) {
        voucherService.save(voucher);
        return Result.ok(voucher.getId());
    }

    @PostMapping("/seckill")
    public Result addSeckillVoucher(@RequestBody Voucher voucher) {
        voucherService.addSeckillVoucher(voucher);
        return Result.ok(voucher.getId());
    }

    @GetMapping("/list/{shopId}")
    public Result queryVoucherOfShop(@PathVariable Long shopId) {
        return voucherService.queryVoucherOfShop(shopId);
    }
}

// VoucherServiceImpl 添加秒杀券
@Transactional
public void addSeckillVoucher(Voucher voucher) {
    save(voucher);
    SeckillVoucher sv = new SeckillVoucher();
    sv.setVoucherId(voucher.getId());
    sv.setStock(voucher.getStock());
    sv.setBeginTime(voucher.getBeginTime());
    sv.setEndTime(voucher.getEndTime());
    seckillVoucherService.save(sv);
    // 库存同步到Redis
    stringRedisTemplate.opsForValue().set(
            RedisConstants.SECKILL_STOCK_KEY + voucher.getId(),
            String.valueOf(voucher.getStock()));
}
```

## 3.4 实现秒杀下单

- 校验时间与库存。
- 扣减库存、创建订单。
- 返回订单 id。

### 3.4.1 VoucherOrderController + VoucherOrderService

```java
// VoucherOrderController.java
@RestController
@RequestMapping("/voucher-order")
public class VoucherOrderController {
    @Resource
    private IVoucherOrderService voucherOrderService;

    @PostMapping("/seckill/{voucherId}")
    public Result seckillVoucher(@PathVariable Long voucherId) {
        return voucherOrderService.seckillVoucher(voucherId);
    }
}

// VoucherOrderServiceImpl.java (同步版-基础实现)
@Slf4j
@Service
public class VoucherOrderServiceImpl
        extends ServiceImpl<VoucherOrderMapper, VoucherOrder>
        implements IVoucherOrderService {

    @Resource private ISeckillVoucherService seckillVoucherService;
    @Resource private RedisIdWorker redisIdWorker;

    @Override
    @Transactional
    public Result seckillVoucher(Long voucherId) {
        // 1.查询秒杀券
        SeckillVoucher sv = seckillVoucherService.getById(voucherId);
        if (sv == null) return Result.fail("秒杀券不存在");
        // 2.校验时间
        if (LocalDateTime.now().isBefore(sv.getBeginTime())) return Result.fail("尚未开始");
        if (LocalDateTime.now().isAfter(sv.getEndTime())) return Result.fail("已结束");
        // 3.校验库存
        if (sv.getStock() < 1) return Result.fail("库存不足");
        // 4.一人一单
        Long userId = UserHolder.getUserId();
        int cnt = lambdaQuery()
                .eq(VoucherOrder::getUserId, userId)
                .eq(VoucherOrder::getVoucherId, voucherId)
                .count();
        if  (cnt > 0) return Result.fail("已购买过");
        // 5.扣减库存(CAS乐观锁)
        boolean success = seckillVoucherService.lambdaUpdate()
                .setSql("stock = stock - 1")
                .eq(SeckillVoucher::getVoucherId, voucherId)
                .gt(SeckillVoucher::getStock, 0)
                .update();
        if (!success) return Result.fail("库存不足");
        // 6.创建订单
        VoucherOrder order = new VoucherOrder();
        order.setId(redisIdWorker.nextId("order"));
        order.setUserId(userId);
        order.setVoucherId(voucherId);
        save(order);
        return Result.ok(order.getId());
    }
}
```

## 3.5 库存超卖问题分析

- 高并发下多线程读取库存导致重复扣减。
- 需要原子扣减或锁。

### 3.5.1 超卖问题时序分析

```
库存剩余1件, t1和t2并发:

  t1                             t2
  ├─ 读库存 stock=1              |
  |                              ├─ 读库存 stock=1
  ├─ 判断 stock>0 -> true        |
  |                              ├─ 判断 stock>0 -> true
  ├─ 扣减 stock=0                |
  |                              ├─ 扣减 stock=-1  ←超卖!
  ├─ 创建订单                     |  
  |                              ├─ 创建订单  ←多了一单

原因: 读取->判断->扣减 不是原子操作
```

## 3.6 乐观锁解决超卖

SQL 乐观锁：
```sql
update tb_seckill_voucher
set stock = stock - 1
where voucher_id = ? and stock > 0;
```

- 受影响行数为 0 则表示库存不足。

### 3.6.1 乐观锁 vs 悲观锁 vs Redis原子操作 对比

| 方案 | 原理 | 并发性能 | 一致 | 适用 |
|------|------|----------|------|------|
| 悲观锁(FOR UPDATE) | 锁行串行 | 差 | 强 | 写多 |
| 乐观锁(stock>0) | 条件UPDATE | 中 | 最终 | 读多写少 |
| 乐观锁(版本号) | version匹配 | 差于CAS | 强 | 严格控制 |
| Redis DECR | 原子递减 | 高 | 最终 | 高并发秒杀 |
| Lua脚本 | 多操作原子 | 高 | 最终 | 复杂校验 |

### 3.6.2 Redis原子扣减

```java
public boolean deductStockRedis(Long voucherId) {
    String key = "seckill:stock:" + voucherId;
    Long stock = stringRedisTemplate.opsForValue().decrement(key);
    return stock != null && stock >= 0; // stock<0 时恢复+1并返回false
}
```

## 3.7 实现一人一单功能

- 先查询是否已有订单；存在则拒绝。
- 或建立唯一索引：`(user_id, voucher_id)`。

### 3.7.1 Redis SETNX一人一单 + Lua原子校验

```java
// 方式1: SETNX简单校验
public boolean checkOnePersonOneOrder(Long userId, Long voucherId) {
    String key = "seckill:order:" + userId + ":" + voucherId;
    return Boolean.TRUE.equals(
            stringRedisTemplate.opsForValue().setIfAbsent(key, "1", 24, TimeUnit.HOURS));
}

// 方式2: Lua脚本原子化 库存校验+一人一单+扣减
public Long checkAndDeduct(Long voucherId) {
    Long userId = UserHolder.getUserId();
    String script =
        "local stock = tonumber(redis.call('get', KEYS[1])) " +
        "if stock <= 0 then return 1 end " +
        "if redis.call('sismember', KEYS[2], ARGV[1]) == 1 then return 2 end " +
        "redis.call('decrby', KEYS[1], 1) " +
        "redis.call('sadd', KEYS[2], ARGV[1]) " +
        "return 0";  // 0=成功, 1=库存不足, 2=重复购买
    DefaultRedisScript<Long> redisScript = new DefaultRedisScript<>();
    redisScript.setScriptText(script);
    redisScript.setResultType(Long.class);
    return stringRedisTemplate.execute(redisScript,
            Arrays.asList(
                    "seckill:stock:" + voucherId,
                    "seckill:user:" + voucherId),
            String.valueOf(userId));
}
```

### 3.7.2 数据库唯一索引DDL

```sql
ALTER TABLE tb_voucher_order ADD UNIQUE INDEX uk_user_voucher (user_id, voucher_id);

-- 代码层面兜底
try { save(order); }
catch (DuplicateKeyException e) { return Result.fail("您已抢购过该优惠券"); }
```

## 3.8 集群下的线程并发安全问题

- 本地锁在多节点无效。
- 需要分布式锁或基于 Redis 的原子校验。

### 3.8.1 集群本地锁失效图解

```
         [ Nginx ]
       +------+------+
       |             |
  [节点1 JVM]   [节点2 JVM]
  | 锁A (本地) |  | 锁B (本地) |  ← 两个JVM互不感知!
  +------+------+
         |
      [ MySQL ]  ← 两线程同时操作同一行
```

### 3.8.2 解决方案

```java
// 方案1: Redis分布式锁
// RLock lock = redissonClient.getLock("lock:seckill:" + voucherId);

// 方案2: DB唯一索引兜底
// ALTER TABLE tb_voucher_order ADD UNIQUE INDEX uk_user_voucher(user_id, voucher_id);

// 方案3: Redis SETNX原子操作
// redisTemplate.opsForValue().setIfAbsent(key, "1", Duration.ofHours(1));
```

## 3.9 秒杀压力测试

### 3.9.1 预期QPS分析

```
┌───────────────────────────┬────────────┬────────────┐
│ 方案                      │ 预估QPS    │ 瓶颈       │
├───────────────────────────┼────────────┼────────────┤
│ 纯MySQL(无缓存)            │ ~500       │ DB连接数   │
│ Redis库存校验+同步DB       │ ~2000      │ DB写压力   │
│ Redis库存+Lua原子操作      │ ~8000      │ Tomcat线程  │
│ 全异步(Stream队列)         │ ~20000     │ Redis QPS  │
│ 全异步+连接池优化          │ ~50000+    │ 带宽       │
└───────────────────────────┴────────────┴────────────┘
```

### 3.9.2 秒杀压测结果记录模板

```
环境: 4C8G, Redis单机, MySQL单机, 并发2000, 库存100
┌────────────────┬──────────┬──────────┬──────────┬──────────┐
│ 方案           │ QPS      │ Avg(ms)  │ Error%   │ 超卖数   │
├────────────────┼──────────┼──────────┼──────────┼──────────┤
│ 无锁           │ 3500     │ 580      │ 0%       │ 45       │
│ synchronized   │ 800      │ 2500     │ 5%       │ 0(单机)  │
│ 乐观锁(CAS)    │ 1500     │ 1300     │ 30%*     │ 0        │
│ Redis分布式锁  │ 1200     │ 1700     │ 10%      │ 0        │
│ Lua脚本原子    │ 8000     │ 250      │ 0%       │ 0        │
│ 异步Stream     │ 15000    │ 130      │ 0%       │ 0        │
└────────────────┴──────────┴──────────┴──────────┴──────────┘
*30%错误率为乐观锁冲突导致的"库存不足", 非系统错误
```

# 4. 分布式锁

## 4.1 基本原理和不同实现方式对比

- 基于数据库：实现简单但性能差。
- 基于 Redis：性能高，注意误删与原子性。
- 基于 Zookeeper：强一致，成本高。

### 4.1.1 分布式锁方案详细对比

| 维度 | Redis | Zookeeper | 数据库 | Etcd |
|------|-------|-----------|--------|------|
| 原理 | SETNX+过期 | 临时顺序节点+Watch | 唯一索引+FOR UPDATE | Lease+事务 |
| 性能 | >10000 QPS | ~1000 QPS | ~500 QPS | ~5000 QPS |
| 一致性 | AP(最终) | CP(强一致) | 强一致 | CP |
| 复杂度 | 低 | 中 | 低 | 中 |
| 可靠性 | 中(需续期) | 高(自动释放) | 中 | 高 |
| 锁释放 | 手动+过期 | 连接断开自动 | 手动 | Lease到期自动 |
| 可重入 | Lua/Redisson | Curator | 额外字段 | 自行实现 |
| 适用 | 高并发,容忍短暂不一致 | 强一致要求 | 已有DB | 云原生/K8s |

### 4.1.2 Redis分布式锁演进路线

```
V1: SETNX lock 1               -> 死锁风险(无过期)
V2: SETNX + EXPIRE             -> 非原子(SETNX后宕机)
V3: SET lock 1 NX EX 10        -> 可能误删(A的锁B删)，A业务执行完毕后，锁早已过期，B拿到了，A业务执行完毕后删除B的锁
V4: SET lock uuid NX EX 10     -> 删除前用UUID标识区分是否是自己的锁。但是判断删除非原子，所以可能会导致：判断标识成功->锁过期->被人获取锁->误删别人锁
V5: Lua原子判断+删除            -> 基本可用, 业务超时锁过期，也就是业务执行期间锁就已经过期
V6: Redisson(WatchDog自动续期)  -> 生产推荐
V7: Redlock红锁(多Redis实例)    -> 更高可靠性
```

## 4.2 Redis的分布式锁实现思路

- `SETNX` + 过期时间。
- 锁值设置为唯一标识，防止误删。

## 4.3 实现Redis分布式锁版本

```java
Boolean ok = stringRedisTemplate.opsForValue()
		.setIfAbsent("lock:order:" + id, threadId, 10, TimeUnit.SECONDS);
```

### 4.3.1 SimpleRedisLock + Lua解锁完整实现

```java
// 锁接口
public interface RedisLock {
    boolean tryLock(long timeoutSec);
    void unlock();
}

// 简易Redis分布式锁
package com.hmdp.utils;
import cn.hutool.core.lang.UUID;
import org.springframework.core.io.ClassPathResource;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.core.script.DefaultRedisScript;
import java.util.Collections;
import java.util.concurrent.TimeUnit;

public class SimpleRedisLock implements RedisLock {
    private final StringRedisTemplate stringRedisTemplate;
    private final String name;
    private final String uuid; // = UUID + threadId, 唯一标识

    private static final DefaultRedisScript<Long> UNLOCK_SCRIPT;
    static {
        UNLOCK_SCRIPT = new DefaultRedisScript<>();
        UNLOCK_SCRIPT.setLocation(new ClassPathResource("unlock.lua"));
        UNLOCK_SCRIPT.setResultType(Long.class);
    }

    public SimpleRedisLock(String name, StringRedisTemplate stringRedisTemplate) {
        this.stringRedisTemplate = stringRedisTemplate;
        this.name = name;
        this.uuid = UUID.randomUUID().toString(true) + "-" + Thread.currentThread().getId();
    }

    @Override
    public boolean tryLock(long timeoutSec) {
        return Boolean.TRUE.equals(
                stringRedisTemplate.opsForValue()
                        .setIfAbsent("lock:" + name, uuid, timeoutSec, TimeUnit.SECONDS));
    }

    @Override
    public void unlock() {
        stringRedisTemplate.execute(UNLOCK_SCRIPT,
                Collections.singletonList("lock:" + name), uuid);
    }
}
```

unlock.lua (resources目录):
```lua
if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
end
return 0
```

## 4.4 Redis分布式锁误删问题

- A 线程锁过期，B 拿到锁。
- A 执行完毕删除锁，误删 B 的锁。

### 4.4.1 误删问题时序

```
A线程: ┌加锁(锁值=A)───业务执行────────────锁过期────解锁(误删B的锁!)──┐
                                                          ↓
B线程:                                        ┌加锁(锁值=B)─执行业务─→
    问题: A解锁时锁已被B持有, A误删B的锁 → C可以立即获取锁 → B与C产生并发安全问题
```

## 4.5 解决Redis分布式锁误删问题

- 解锁前比对锁值，仅删除自己持有的锁。

## 4.6 分布式锁的原子性问题

- 比较与删除需要原子执行。
- 通过 Lua 脚本保证原子性。

## 4.7 Lua脚本解决多条命令原子性问题

```lua
if redis.call('get', KEYS[1]) == ARGV[1] then
		return redis.call('del', KEYS[1])
end
return 0
```

## 4.8 Java调用lua脚本改造分布式锁

```java
DefaultRedisScript<Long> script = new DefaultRedisScript<>();
script.setScriptText(luaScript);
script.setResultType(Long.class);
Long result = stringRedisTemplate.execute(script, Collections.singletonList(key), threadId);
```

## 4.9 Redisson功能介绍

- 可重入锁、公平锁、读写锁、联锁。
- WatchDog 自动续期。

### 4.9.1 RedissonConfig完整Spring Boot配置（自定义配置可以这样，一般不需要自己配置）

```java
package com.hmdp.config;
import org.redisson.Redisson;
import org.redisson.api.RedissonClient;
import org.redisson.config.Config;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RedissonConfig {
    @Value("${spring.redis.host:localhost}") private String host;
    @Value("${spring.redis.port:6379}") private int port;
    @Value("${spring.redis.password:}") private String password;

    @Bean(destroyMethod = "shutdown")
    public RedissonClient redissonClient() {
        Config config = new Config();
        config.useSingleServer()
                .setAddress("redis://" + host + ":" + port)
                .setConnectionPoolSize(64)
                .setConnectionMinimumIdleSize(24)
                .setConnectTimeout(10000)
                .setTimeout(3000)
                .setRetryAttempts(3);
        if (password != null && !password.isEmpty()) {
            config.useSingleServer().setPassword(password);
        }
        return Redisson.create(config);
    }
}

```
### 4.9.2 redisson在springboot中的依赖

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-spring-boot-starter</artifactId>
    <version>3.52.0</version>
</dependency>
```

### 4.9.3 redisson在普通项目中的依赖
```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson</artifactId>
</dependency>
```
## 4.10 Redisson快速入门

```java
RLock lock = redissonClient.getLock("lock:order:" + id);
boolean isLock = lock.tryLock(1, 10, TimeUnit.SECONDS);
try {
		// 业务逻辑
} finally {
		lock.unlock();
}
```

### 4.10.1 Redisson可重入锁(ReentrantLock)示例

- 可重入锁的意义：防止“自己锁自己”；支持递归调用时可以拿到同一把锁

```java
@Service
public class ReentrantLockDemo {
    private final RedissonClient redissonClient;

    public void outerMethod(Long id) {
        RLock lock = redissonClient.getLock("lock:reentrant:" + id);
        lock.lock();
        try {
            System.out.println("外层加锁");
            innerMethod(id); // 同一线程可以再次加锁(可重入)
        } finally {
            lock.unlock();
        }
    }

    private void innerMethod(Long id) {
        RLock lock = redissonClient.getLock("lock:reentrant:" + id);
        lock.lock(); // 重入计数+1
        try { /* 内层业务 */ }
        finally { lock.unlock(); } // 计数-1
    }
}
```

### 4.10.2 Redisson公平锁(FairLock)

```java
// 公平锁: 按请求先后顺序获取锁, 防止线程饥饿
RLock fairLock = redissonClient.getFairLock("lock:fair:order:" + orderId);
boolean locked = fairLock.tryLock(5, 30, TimeUnit.SECONDS);
try {
    if (locked) { /* 业务 */ }
} finally {
    if (fairLock.isHeldByCurrentThread()) fairLock.unlock();
}
```

### 4.10.3 Redisson读写锁(ReadWriteLock)

```java
// 读锁共享(多个读并发), 写锁独占(读写互斥)
RReadWriteLock rwLock = redissonClient.getReadWriteLock("rw:shop:" + shopId);

// 读操作: 使用读锁(允许多线程同时持有)
public Shop readShop(Long shopId) {
    RLock readLock = rwLock.readLock();
    readLock.lock();
    try { return getById(shopId); }
    finally { readLock.unlock(); }
}

// 写操作: 使用写锁(独占)
public void updateShop(Shop shop) {
    RLock writeLock = rwLock.writeLock();
    writeLock.lock();
    try { updateById(shop); }
    finally { writeLock.unlock(); }
}
```

### 4.10.4 Redisson信号量(Semaphore)

```java
// 信号量: 控制同时访问资源的线程数(如限流)
RSemaphore semaphore = redissonClient.getSemaphore("sem:api:limit");
semaphore.trySetPermits(10); // 初始化10个许可

public void handleRequest() {
    try {
        semaphore.acquire(); // 获取许可, 没有则阻塞
        // 业务逻辑
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    } finally {
        semaphore.release(); // 释放许可
    }
}

// 带超时的获取
public boolean handleWithTimeout() {
    try {
        boolean acquired = semaphore.tryAcquire(3, TimeUnit.SECONDS);
        if (!acquired) return false; // 超时拒绝
        try { /* 业务 */ }
        finally { semaphore.release(); }
        return true;
    } catch (InterruptedException e) {
        return false;
    }
}
```

### 4.10.5 Redisson闭锁(CountDownLatch)

```java
// 分布式CountDownLatch: 等待多个服务节点就绪
RCountDownLatch latch = redissonClient.getCountDownLatch("latch:service:startup");
latch.trySetCount(3); // 等待3个节点就绪

// 各节点就绪后调用countDown
latch.countDown();

// 主节点等待所有就绪
try {
    latch.await(30, TimeUnit.SECONDS);
    // 所有节点就绪, 开始处理
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

## 4.11 Redisson的可重入锁原理

- 基于 Hash 存储：`field=threadId`，`value=重入次数`。
- 同线程重复加锁只递增计数。

### 4.11.1 可重入锁内部存储与Lua原理

```
Redis存储结构(Hash):
  KEY: lock:order:1
  +-- field: "uuid:thread-1"
  |   +-- value: 2            <- 重入次数

加锁Lua核心:
  1. EXISTS lock → 不存在 → HSET field 1; PEXPIRE 30000
     └─ 存在 → HEXISTS field → 存在 → HINCRBY field 1
              └─ 不存在 → return ttl(其他线程持有)

解锁Lua核心:
  1. HINCRBY field -1
  2. counter>0 → PEXPIRE 30000; return 0
  3. counter=0 → DEL lock; PUBLISH release_channel; return 1
```

### 4.11.2 WatchDog续期原理

```
Redisson WatchDog:
  - leaseTime=-1时启用(默认)
  - 默认30秒过期, 每10秒续期到30秒
  - 解锁或连接断开时取消续期任务

时间线: 加锁(30s) → 10s续期 → 20s续期 → 30s续期 → 解锁
       |30s       |30s       |30s       |30s       |
```

## 4.13 Redisson的multiLock原理

- 多把锁一起获取，全部成功才继续。
- 适合跨多个资源的原子操作。

### 4.13.1 MultiLock联锁示例

```java
// 同时锁定多个资源, 全部成功才继续
RLock lock1 = redissonClient.getLock("lock:order:" + orderId);
RLock lock2 = redissonClient.getLock("lock:voucher:" + voucherId);
RLock multiLock = redissonClient.getMultiLock(lock1, lock2);

if (multiLock.tryLock(5, 30, TimeUnit.SECONDS)) {
    try {
        // 扣库存 + 创建订单
    } finally {
        multiLock.unlock();
    }
}
```

## 4.14 Redlock红锁算法

### 4.14.1 Redlock原理

```
Redlock: 在N个独立的Redis节点上获取锁(N一般为奇数,如5)
  1. 获取当前时间戳
  2. 依次向N个Redis节点请求锁(SET NX PX)
  3. 获取锁的时间 = 当前时间 - 步骤1的时间
  4. 若在大多数(N/2+1)节点获取成功 && 获取耗时<锁TTL, 则成功
  5. 成功: 锁真正有效时间 = TTL - 获取耗时
  6. 失败: 向所有节点发送解锁命令

关键原则:
  - 各节点独立部署, 无主从复制关系(避免脑裂)
  - 时钟偏移不能太大
```

### 4.14.2 Redisson Redlock实现

```java
package com.hmdp.config;
import org.redisson.Redisson;
import org.redisson.api.RedissonClient;
import org.redisson.config.Config;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RedissonMultiConfig {

    // 3个独立的Redis实例
    @Bean(name = "redissonClient1")
    public RedissonClient client1() {
        Config config = new Config();
        config.useSingleServer().setAddress("redis://192.168.1.1:6379");
        return Redisson.create(config);
    }

    @Bean(name = "redissonClient2")
    public RedissonClient client2() {
        Config config = new Config();
        config.useSingleServer().setAddress("redis://192.168.1.2:6379");
        return Redisson.create(config);
    }

    @Bean(name = "redissonClient3")
    public RedissonClient client3() {
        Config config = new Config();
        config.useSingleServer().setAddress("redis://192.168.1.3:6379");
        return Redisson.create(config);
    }
}

// Redlock使用
@Service
public class RedlockService {
    @Resource private RedissonClient client1;
    @Resource private RedissonClient client2;
    @Resource private RedissonClient client3;

    public void redlockDemo() {
        RLock lock1 = client1.getLock("lock:order:1");
        RLock lock2 = client2.getLock("lock:order:1");
        RLock lock3 = client3.getLock("lock:order:1");

        RLock redLock = client1.getRedLock(lock1, lock2, lock3);
        try {
            if (redLock.tryLock(500, 30000, TimeUnit.MILLISECONDS)) {
                // 在大多数(2/3)节点获取锁成功
                try { /* 业务 */ } finally { redLock.unlock(); }
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

## 4.15 分布式锁实战总结

### 4.15.1 所有锁方案对比

| 锁方案 | 优点 | 缺点 | 适用场景 |
|--------|------|------|----------|
| synchronized | 简单, JVM内置 | 单机, 不可重试 | 本地演示 |
| ReentrantLock | 可重入, 可超时 | 单机 | 复杂的本地同步 |
| 数据库悲观锁 | 严格一致性, 无需新组件 | 性能差, 可能死锁 | 已有DB, 并发低 |
| Redis SETNX | 高性能, 易用 | 可能误删, 需续期 | 高并发, 强可用 |
| Redisson | WatchDog, 可重入, 丰富锁类型 | 依赖Redisson | 生产环境首选 |
| Zookeeper | 强一致, 自动释放 | 性能低, 运维重 | 强一致性要求 |
| Redlock | 高可靠, 防脑裂 | 多实例成本, 实现复杂 | 金融级可靠性 |

### 4.15.2 常见坑与解决方案

| 坑 | 问题 | 解决方案 |
|----|------|----------|
| 死锁 | 无过期时间, 持有线程崩溃 | 必须设置过期时间 |
| 误删 | A过期B获取后A删了B的锁 | value存唯一标识, Lua原子比对删除 |
| 非原子解锁 | GET+DEL两步非原子 | Lua脚本原子化 |
| 锁过期 | 业务超时, 锁过期被他人获取 | WatchDog自动续期; 或自定义续期逻辑 |
| 脑裂 | 主从切换锁丢失 | Redlock; 或设置合理的容错策略 |
| 性能 | 加锁本身有开销 | 减小锁粒度; 读写分离锁 |
| 循环依赖 | A锁→等B锁, B锁→等A锁 | 统一加锁顺序; tryLock超时处理 |

# 5. 秒杀优化

## 5.1 异步秒杀思路

- 下单流程拆分：资格校验 -> 发送消息 -> 异步创建订单。
- 目标：削峰、保护数据库。

### 5.1.1 异步秒杀架构图

```
[用户请求] -> [Nginx负载均衡] -> [SpringBoot]
                                      |
                              +(资格校验Lua脚本) -> Redis
                              |   - 库存检查
                              |   - 一人一单
                              |   - 扣减库存
                              |
                              +(返回订单ID给用户)
                              |
                              +(XADD写入订单消息) -> Redis Stream
                                                       |
                                                  [消费者组]
                                                       |
                                                    [MySQL]
                                                 异步创建订单
```

### 5.1.2 Lua脚本 资格校验+发送消息

```lua
-- seckill.lua
-- KEYS[1]: 库存key KEYS[2]: 用户set KEYS[3]: Stream key
-- ARGV[1]: userId ARGV[2]: voucherId ARGV[3]: orderId
local stock = tonumber(redis.call('get', KEYS[1]))
if stock <= 0 then return 1 end
if redis.call('sismember', KEYS[2], ARGV[1]) == 1 then return 2 end
redis.call('incrby', KEYS[1], -1)
redis.call('sadd', KEYS[2], ARGV[1])
-- 发送订单消息到Stream
redis.call('xadd', KEYS[3], '*', 'userId', ARGV[1], 'voucherId', ARGV[2], 'orderId', ARGV[3])
return 0
```

Java调用:
```java
public Result seckillVoucherAsync(Long voucherId) {
    Long userId = UserHolder.getUserId();
    long orderId = redisIdWorker.nextId("order");
    String script = "local stock=..."; // 上方Lua脚本
    DefaultRedisScript<Long> redisScript = new DefaultRedisScript<>();
    redisScript.setScriptText(script);
    redisScript.setResultType(Long.class);
    Long result = stringRedisTemplate.execute(redisScript,
            Arrays.asList(
                    "seckill:stock:" + voucherId,
                    "seckill:user:" + voucherId,
                    "stream.orders"),
            String.valueOf(userId), String.valueOf(voucherId), String.valueOf(orderId));
    if (result != null && result == 0) return Result.ok(orderId);
    if (result != null && result == 1) return Result.fail("库存不足");
    return Result.fail("已购买过");
}
```

## 5.3 基于阻塞队列实现异步下单

- 使用 `ArrayBlockingQueue` 作为内存队列。
- 消费线程从队列取订单并写库。
- 后续替换为 Redis Stream 更可靠。

### 5.3.1 Redis Stream消费者（完整实现）

```java
package com.hmdp.service;
import cn.hutool.json.JSONUtil;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.connection.stream.*;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;
import javax.annotation.PostConstruct;
import javax.annotation.Resource;
import java.time.Duration;
import java.util.List;
import java.util.Map;

@Slf4j
@Service
public class VoucherOrderStreamConsumer {
    @Resource private StringRedisTemplate stringRedisTemplate;
    @Resource private IVoucherOrderService voucherOrderService;
    private static final String STREAM_KEY = "stream.orders";
    private static final String GROUP = "order-consumers";
    private static final String CONSUMER = "consumer-" + System.currentTimeMillis();

    @PostConstruct
    public void init() {
        try {
            stringRedisTemplate.opsForStream().createGroup(STREAM_KEY, GROUP);
        } catch (Exception e) {
            log.info("消费者组已存在, 继续监听");
        }
        new Thread(this::listenLoop, "stream-consumer").start();
    }

    private void listenLoop() {
        while (true) {
            try {
                List<MapRecord<String, Object, Object>> records =
                        stringRedisTemplate.opsForStream().read(
                                Consumer.from(GROUP, CONSUMER),
                                StreamReadOptions.empty().count(1).block(Duration.ofSeconds(2)),
                                StreamOffset.create(STREAM_KEY, ReadOffset.lastConsumed()));
                if (records == null || records.isEmpty()) continue;
                for (MapRecord<String, Object, Object> record : records) {
                    handleMessage(record);
                    // ACK确认
                    stringRedisTemplate.opsForStream().acknowledge(STREAM_KEY, GROUP,
                            record.getId().getValue());
                }
            } catch (Exception e) {
                log.error("Stream消费异常", e);
                handlePendingMessages(); // 处理未确认消息
            }
        }
    }

    private void handleMessage(MapRecord<String, Object, Object> record) {
        Map<String, String> msg = (Map<String, String>)(Map)record.getValue();
        Long userId = Long.valueOf(msg.get("userId"));
        Long voucherId = Long.valueOf(msg.get("voucherId"));
        Long orderId = Long.valueOf(msg.get("orderId"));
        try {
            voucherOrderService.createVoucherOrderAsync(orderId, userId, voucherId);
        } catch (Exception e) {
            log.error("创建订单失败: orderId={}", orderId, e);
        }
    }

    /** 处理Pending List中的补偿消息 */
    private void handlePendingMessages() {
        try {
            List<MapRecord<String, Object, Object>> pending =
                    stringRedisTemplate.opsForStream().read(
                            Consumer.from(GROUP, CONSUMER),
                            StreamReadOptions.empty().count(5),
                            StreamOffset.create(STREAM_KEY, ReadOffset.from("0")));
            if (pending != null) {
                for (MapRecord<String, Object, Object> record : pending) {
                    handleMessage(record);
                    stringRedisTemplate.opsForStream().acknowledge(STREAM_KEY, GROUP,
                            record.getId().getValue());
                }
            }
        } catch (Exception e) {
            log.error("处理Pending消息失败", e);
        }
    }
}
```

# 6. Redis消息队列

## 6.1 认识消息队列

- 解耦、削峰、异步、可重试。
- Redis 可用 List、PubSub、Stream。

### 6.1.1 Redis消息队列 vs 专业MQ对比

| 维度 | List(BRPOP) | PubSub | Stream | RabbitMQ | Kafka |
|------|-------------|--------|--------|----------|-------|
| 持久化 | RDB/AOF | 无持久化 | 支持 | 支持 | 支持 |
| 消费确认 | 无(取出即删) | 无 | ACK+Pending | ACK | Offset |
| 消费者组 | 无 | 无 | 支持 | 支持 | 支持 |
| 消息回溯 | 不支持 | 不支持 | 支持(按ID) | 不支持(默认) | 支持 |
| 吞吐量 | 高 | 高 | 高 | 中 | 极高 |
| 可靠性 | 低 | 极低 | 中 | 高 | 高 |
| 适用场景 | 简单队列 | 实时通知 | 轻量可靠 | 企业级 | 大数据 |

### 6.1.2 消息队列选型建议

```
需要极简 + Redis已有     → Redis Stream
需要强可靠 + 企业级       → RabbitMQ
需要大数据 + 高吞吐       → Kafka
需要云原生 + 低延迟       → RocketMQ
```

## 6.2 基于List实现消息队列

- 生产：`LPUSH queue msg`
- 消费：`BRPOP queue 0`
- 简单可靠，但缺少消费确认。

```java
// List生产者
stringRedisTemplate.opsForList().leftPush("queue:order", JSONUtil.toJsonStr(orderMsg));

// List消费者(阻塞等待)
while (true) {
    String msg = stringRedisTemplate.opsForList().rightPop("queue:order", 0, TimeUnit.SECONDS);
    if (msg != null) handleMessage(msg);
}
```

## 6.3 PubSub实现消息队列

- 发布：`PUBLISH channel msg`
- 订阅：`SUBSCRIBE channel`
- 实时性强，但不可靠（无持久化）。

```java
// PubSub发布者
stringRedisTemplate.convertAndSend("channel:order", JSONUtil.toJsonStr(orderMsg));

// PubSub订阅者
@Bean
public ChannelTopic orderTopic() { return new ChannelTopic("channel:order"); }

@Component
public class OrderSubscriber implements MessageListener {
    @Override
    public void onMessage(Message message, byte[] pattern) {
        handleMessage(new String(message.getBody()));
    }
}
```

## 6.4 Stream的单消费模式

- 生产：`XADD stream * field value`
- 消费：`XREAD BLOCK 0 STREAMS stream >`
- 支持持久化与消费位点。

```java
// Stream生产者 (Java)
Map<String, String> msg = new HashMap<>();
msg.put("userId", "1");
msg.put("voucherId", "10");
stringRedisTemplate.opsForStream().add("stream.orders", msg);

// Stream消费者 (阻塞读取)
List<MapRecord<String, Object, Object>> records = stringRedisTemplate.opsForStream()
        .read(StreamOffset.create("stream.orders", ReadOffset.from("0")),
              StreamReadOptions.empty().count(5).block(Duration.ofSeconds(2)));
```

## 6.5 Stream的消费者组模式

- 创建组：`XGROUP CREATE stream g1 0 MKSTREAM`
- 消费：`XREADGROUP GROUP g1 c1 BLOCK 2000 STREAMS stream >`
- 支持 ACK 与 Pending List。

```java
// 创建消费者组
stringRedisTemplate.opsForStream().createGroup("stream.orders", "g1");

// 消费者组消费
List<MapRecord<String, Object, Object>> records = stringRedisTemplate.opsForStream()
        .read(Consumer.from("g1", "c1"),
              StreamReadOptions.empty().count(1).block(Duration.ofSeconds(2)),
              StreamOffset.create("stream.orders", ReadOffset.lastConsumed()));

// ACK确认
for (MapRecord<String, Object, Object> record : records) {
    try {
        handleMessage(record);
        stringRedisTemplate.opsForStream()
                .acknowledge("stream.orders", "g1", record.getId().getValue());
    } catch (Exception e) {
        // 处理失败, 消息进入Pending List等待重试
    }
}
```

### 6.5.1 消息幂等性处理

```java
/** 消费幂等: 使用Redis SETNX防止重复消费 */
public void handleMessageWithIdempotence(MapRecord<String, Object, Object> record) {
    String msgId = record.getId().getValue();
    String idempotentKey = "stream:consumed:" + msgId;
    // 只有首次消费设置成功, 后续返回false跳过
    Boolean consumed = stringRedisTemplate.opsForValue()
            .setIfAbsent(idempotentKey, "1", 24, TimeUnit.HOURS);
    if (!Boolean.TRUE.equals(consumed)) {
        log.warn("重复消息,跳过: msgId={}", msgId);
        return;
    }
    // 执行业务
    doBusinessLogic(record);
}
```

## 6.6 基于Stream消息队列实现异步秒杀

- Lua 脚本中 `XADD` 写入订单消息。
- 消费者组异步处理并创建订单。
- 处理 Pending List 的补偿逻辑。

## 6.7 Redis Stream实战完整案例

### 6.7.1 订单处理完整Pipeline (Stream)

```
[资格校验Lua] -> XADD stream.orders -> [消费者组监听]
                                          |
                                     +-> 消费者1: 创建订单到MySQL
                                     |    + ACK确认
                                     +-> 消费者2: 创建订单到MySQL
                                     |    + ACK确认
                                     |
                                     +-> Pending List监控
                                          (定时处理未确认消息)
```

### 6.7.2 完整订单消费者（含死信队列）

```java
@Slf4j
@Service
public class OrderStreamService {
    @Resource private StringRedisTemplate stringRedisTemplate;
    @Resource private IVoucherOrderService orderService;
    private static final String STREAM_KEY = "stream.orders";
    private static final String DEAD_LETTER_KEY = "stream.orders.dlq";
    private static final String GROUP = "order-group";
    private static final int MAX_RETRY = 3;

    @PostConstruct
    public void start() {
        try { stringRedisTemplate.opsForStream().createGroup(STREAM_KEY, GROUP); }
        catch (Exception ignored) {}
        new Thread(this::consumeLoop).start();
        // 定时任务: 处理Pending超过阈值的消息 -> 移入死信队列
        // @Scheduled(fixedDelay = 30000)
        // public void processPending() { ... }
    }

    private void consumeLoop() {
        String consumer = "consumer-" + System.currentTimeMillis();
        while (true) {
            try {
                List<MapRecord<String, Object, Object>> records = stringRedisTemplate
                        .opsForStream().read(
                                Consumer.from(GROUP, consumer),
                                StreamReadOptions.empty().count(5).block(Duration.ofSeconds(10)),
                                StreamOffset.create(STREAM_KEY, ReadOffset.lastConsumed()));
                if (records == null || records.isEmpty()) continue;
                for (MapRecord<String, Object, Object> record : records) {
                    processOrder(record, consumer);
                }
            } catch (Exception e) {
                log.error("消费异常", e);
            }
        }
    }

    private void processOrder(MapRecord<String, Object, Object> record, String consumer) {
        String msgId = record.getId().getValue();
        Map<String, String> msg = (Map<String, String>)(Map)record.getValue();
        try {
            // 幂等检查
            if (isConsumed(msgId)) {
                stringRedisTemplate.opsForStream().acknowledge(STREAM_KEY, GROUP, msgId);
                return;
            }
            // 执行订单创建
            Long userId = Long.valueOf(msg.get("userId"));
            Long voucherId = Long.valueOf(msg.get("voucherId"));
            Long orderId = Long.valueOf(msg.get("orderId"));
            orderService.createVoucherOrderAsync(orderId, userId, voucherId);
            // ACK确认
            stringRedisTemplate.opsForStream().acknowledge(STREAM_KEY, GROUP, msgId);
            markConsumed(msgId);
        } catch (Exception e) {
            log.error("订单处理失败: msgId={}", msgId, e);
            // 超过最大重试次数 -> 移入死信队列
            if (getRetryCount(msgId) >= MAX_RETRY) {
                moveToDeadLetter(record);
                stringRedisTemplate.opsForStream().acknowledge(STREAM_KEY, GROUP, msgId);
            } else {
                incrRetryCount(msgId);
                // 不ACK, 等待下次重试
            }
        }
    }

    private boolean isConsumed(String msgId) {
        return Boolean.TRUE.equals(
                stringRedisTemplate.hasKey("stream:consumed:" + msgId));
    }
    private void markConsumed(String msgId) {
        stringRedisTemplate.opsForValue()
                .set("stream:consumed:" + msgId, "1", 24, TimeUnit.HOURS);
    }
    private int getRetryCount(String msgId) {
        String val = stringRedisTemplate.opsForValue().get("stream:retry:" + msgId);
        return val == null ? 0 : Integer.parseInt(val);
    }
    private void incrRetryCount(String msgId) {
        stringRedisTemplate.opsForValue().increment("stream:retry:" + msgId, 1);
    }
    private void moveToDeadLetter(MapRecord<String, Object, Object> record) {
        stringRedisTemplate.opsForStream().add(DEAD_LETTER_KEY, record.getValue());
        log.warn("消息移入死信队列: {}", record.getId());
    }
}
```

# 7. 达人探店

## 7.1 发布探店笔记

- 保存 `tb_blog`，关联作者 id。
- 发布后将 blogId 推送到粉丝收件箱（ZSET）。

### 7.1.1 BlogController + BlogService完整实现

```java
// BlogController.java
@RestController
@RequestMapping("/blog")
public class BlogController {
    @Resource private IBlogService blogService;

    @PostMapping
    public Result saveBlog(@RequestBody Blog blog) {
        return blogService.saveBlog(blog);
    }

    @GetMapping("/{id}")
    public Result queryBlogById(@PathVariable Long id) {
        return blogService.queryBlogById(id);
    }

    @PutMapping("/like/{id}")
    public Result likeBlog(@PathVariable Long id) {
        return blogService.likeBlog(id);
    }

    @GetMapping("/likes/{id}")
    public Result queryBlogLikes(@PathVariable Long id) {
        return blogService.queryBlogLikes(id);
    }

    @GetMapping("/of/user")
    public Result queryBlogByUserId(
            @RequestParam Long id,
            @RequestParam(defaultValue = "0") Integer current) {
        return blogService.queryBlogByUserId(id, current);
    }

    @GetMapping("/hot")
    public Result queryHotBlog(@RequestParam(defaultValue = "0") Integer current) {
        return blogService.queryHotBlog(current);
    }

    @GetMapping("/of/follow")
    public Result queryBlogOfFollow(
            @RequestParam("lastId") Long max,
            @RequestParam(defaultValue = "0") Integer offset) {
        return blogService.queryBlogOfFollow(max, offset);
    }
}

// BlogServiceImpl 核心方法
@Service
public class BlogServiceImpl extends ServiceImpl<BlogMapper, Blog>
        implements IBlogService {

    @Resource private IUserService userService;
    @Resource private IFollowService followService;
    @Resource private StringRedisTemplate stringRedisTemplate;

    @Override
    @Transactional
    public Result saveBlog(Blog blog) {
        // 1. 获取登录用户
        UserDTO user = UserHolder.getUser();
        blog.setUserId(user.getId());
        // 2. 保存探店笔记
        boolean success = save(blog);
        if (!success) return Result.fail("发布失败");
        // 3. 推送到所有粉丝的收件箱 (推模式 Feed流)
        List<Follow> follows = followService.lambdaQuery()
                .eq(Follow::getFollowUserId, user.getId()).list();
        for (Follow follow : follows) {
            String key = RedisConstants.FEED_KEY + follow.getUserId();
            stringRedisTemplate.opsForZSet()
                    .add(key, blog.getId().toString(), System.currentTimeMillis());
        }
        return Result.ok(blog.getId());
    }

    @Override
    public Result likeBlog(Long id) {
        Long userId = UserHolder.getUserId();
        String key = RedisConstants.BLOG_LIKED_KEY + id;
        // 判断是否已点赞
        Double score = stringRedisTemplate.opsForZSet().score(key, String.valueOf(userId));
        if (score != null) {
            // 已点赞 -> 取消点赞
            boolean success = lambdaUpdate()
                    .setSql("liked = liked - 1").eq(Blog::getId, id)
                    .gt(Blog::getLiked, 0).update();
            if (success) stringRedisTemplate.opsForZSet().remove(key, String.valueOf(userId));
        } else {
            // 未点赞 -> 点赞
            lambdaUpdate().setSql("liked = liked + 1").eq(Blog::getId, id).update();
            stringRedisTemplate.opsForZSet()
                    .add(key, String.valueOf(userId), System.currentTimeMillis());
        }
        return Result.ok();
    }

    @Override
    public Result queryBlogLikes(Long id) {
        String key = RedisConstants.BLOG_LIKED_KEY + id;
        // 查询点赞排行榜 Top5
        Set<String> top5 = stringRedisTemplate.opsForZSet()
                .reverseRange(key, 0, 4);
        if (top5 == null || top5.isEmpty()) return Result.ok(Collections.emptyList());
        List<Long> ids = top5.stream().map(Long::valueOf).collect(Collectors.toList());
        List<UserDTO> users = userService.lambdaQuery()
                .in(User::getId, ids)
                .list()
                .stream()
                .map(u -> BeanUtil.copyProperties(u, UserDTO.class))
                .collect(Collectors.toList());
        return Result.ok(users);
    }
}
```

## 7.2 查看探店笔记

- 关联查询用户昵称和头像。
- 返回是否点赞：通过 `ZSET` 判断。

## 7.3 点赞功能

- `ZADD blog:liked:{blogId} score=userId`，score 用时间戳。
- 取消点赞：`ZREM`。
- 数据库点赞数自增/自减。

## 7.4 点赞排行榜

- `ZREVRANGE blog:liked:{blogId} 0 4 WITHSCORES`。
- 取出用户信息展示。

# 8. 好友关注

## 8.1 关注和取关

- 关注写库 `tb_follow`。
- Redis 使用 `Set` 存关注关系：`follow:{userId}`。

### 8.1.1 FollowController + FollowService完整实现

```java
// FollowController.java
@RestController
@RequestMapping("/follow")
public class FollowController {
    @Resource private IFollowService followService;

    @PutMapping("/{id}/{isFollow}")
    public Result follow(@PathVariable Long id, @PathVariable Boolean isFollow) {
        return followService.follow(id, isFollow);
    }

    @GetMapping("/or/not/{id}")
    public Result isFollow(@PathVariable Long id) {
        return followService.isFollow(id);
    }

    @GetMapping("/common/{id}")
    public Result commonFollow(@PathVariable Long id) {
        return followService.commonFollow(id);
    }
}

// FollowServiceImpl.java
@Service
public class FollowServiceImpl extends ServiceImpl<FollowMapper, Follow>
        implements IFollowService {

    @Resource private StringRedisTemplate stringRedisTemplate;
    @Resource private IUserService userService;

    @Override
    @Transactional
    public Result follow(Long followUserId, Boolean isFollow) {
        Long userId = UserHolder.getUserId();
        if (isFollow) {
            Follow follow = new Follow();
            follow.setUserId(userId);
            follow.setFollowUserId(followUserId);
            if (save(follow)) {
                stringRedisTemplate.opsForSet()
                        .add(RedisConstants.FOLLOW_KEY + userId, String.valueOf(followUserId));
            }
        } else {
            lambdaUpdate()
                    .eq(Follow::getUserId, userId)
                    .eq(Follow::getFollowUserId, followUserId).remove();
            stringRedisTemplate.opsForSet()
                    .remove(RedisConstants.FOLLOW_KEY + userId, String.valueOf(followUserId));
        }
        return Result.ok();
    }

    @Override
    public Result isFollow(Long followUserId) {
        Long userId = UserHolder.getUserId();
        Integer count = lambdaQuery()
                .eq(Follow::getUserId, userId)
                .eq(Follow::getFollowUserId, followUserId).count();
        return Result.ok(count > 0);
    }

    @Override
    public Result commonFollow(Long id) {
        Long userId = UserHolder.getUserId();
        Set<String> common = stringRedisTemplate.opsForSet()
                .intersect(
                        RedisConstants.FOLLOW_KEY + userId,
                        RedisConstants.FOLLOW_KEY + id);
        if (common == null || common.isEmpty()) return Result.ok(Collections.emptyList());
        List<Long> ids = common.stream().map(Long::valueOf).collect(Collectors.toList());
        List<UserDTO> users = userService.lambdaQuery()
                .in(User::getId, ids)
                .list().stream()
                .map(u -> BeanUtil.copyProperties(u, UserDTO.class))
                .collect(Collectors.toList());
        return Result.ok(users);
    }
}
```

## 8.2 共同关注

- `SINTER follow:user1 follow:user2`。
- 再查询用户信息展示。

## 8.3 Feed流实现方案分析

- 拉模式：查询关注列表再分页。
- 推模式：发布时推送到粉丝收件箱（ZSET）。

## 8.4 推送到粉丝收件箱

- 发布 blog 后：
```java
ZSetOperations<String, String> zset = stringRedisTemplate.opsForZSet();
zset.add("feed:" + fanId, blogId.toString(), System.currentTimeMillis());
```

## 8.5 滚动分页查询收件箱的思路

- 以 `score`（时间戳）为游标。
- 下一页查询 `score < lastScore`。

## 8.6 实现滚动分页查询

```java
Set<ZSetOperations.TypedTuple<String>> set = zset.reverseRangeByScoreWithScores(
		"feed:" + userId, 0, maxTime, offset, 2);
```

# 9. 附近商铺

## 9.1 GEO数据结构的基本用法

- `GEOADD` 添加坐标。
- `GEOSEARCH` 按半径或范围搜索。

## 9.2 导入店铺数据到GEO

```java
String key = "shop:geo:" + typeId;
redisTemplate.opsForGeo().add(key, new Point(x, y), shopId.toString());
```

## 9.3 实现附近商户功能

```java
GeoResults<RedisGeoCommands.GeoLocation<String>> result =
		redisTemplate.opsForGeo().search(key,
				GeoReference.fromCoordinate(x, y), new Distance(5, Metrics.KILOMETERS),
				RedisGeoCommands.GeoSearchCommandArgs.newGeoSearchArgs().includeDistance().limit(10));
```

# 10. 用户签到

## 10.1 BitMap功能演示

- `SETBIT sign:2025:05:12 0 1`。
- 用位图节省空间。

## 10.2 实现签到功能

```java
int day = LocalDate.now().getDayOfMonth();
String key = "sign:" + userId + ":" + yyyyMM;
stringRedisTemplate.opsForValue().setBit(key, day - 1, true);
```

## 10.3 统计连续签到

```java
List<Long> result = stringRedisTemplate.opsForValue().bitField(key,
		BitFieldSubCommands.create().get(BitFieldSubCommands.BitFieldType.unsigned(day)).valueAt(0));
```

- 从低位连续统计 1 的个数。

# 11. UV统计

## 11.1 HyperLogLog的用法

- `PFADD uv:2025-05-12 user1 user2 ...`
- `PFCOUNT uv:2025-05-12`

## 11.2 测试百万数据的统计

- HyperLogLog 误差约 0.81%，占用 12KB 左右。
- 适合大规模 UV 统计。

### 11.2.1 Java实现UV统计

```java
@Service
public class UvStatisticsService {
    @Resource private StringRedisTemplate stringRedisTemplate;

    /** 记录用户访问 */
    public void recordVisit(Long userId) {
        String key = "uv:" + LocalDate.now().format(DateTimeFormatter.ofPattern("yyyyMMdd"));
        stringRedisTemplate.opsForHyperLogLog().add(key, String.valueOf(userId));
    }

    /** 统计今日UV */
    public long getTodayUv() {
        String key = "uv:" + LocalDate.now().format(DateTimeFormatter.ofPattern("yyyyMMdd"));
        return Optional.ofNullable(stringRedisTemplate.opsForHyperLogLog().size(key)).orElse(0L);
    }

    /** 统计指定日期范围UV（合并） */
    public long getRangeUv(LocalDate start, LocalDate end) {
        List<String> keys = new ArrayList<>();
        LocalDate cur = start;
        while (!cur.isAfter(end)) {
            keys.add("uv:" + cur.format(DateTimeFormatter.ofPattern("yyyyMMdd")));
            cur = cur.plusDays(1);
        }
        String unionKey = "uv:union:" + UUID.randomUUID().toString(true);
        stringRedisTemplate.opsForHyperLogLog().union(unionKey,
                keys.toArray(new String[0]));
        Long result = stringRedisTemplate.opsForHyperLogLog().size(unionKey);
        stringRedisTemplate.delete(unionKey);
        return result != null ? result : 0L;
    }
}
```

# 12. Redis事务实战

## 12.1 MULTI/EXEC 基础事务

```java
// Redis事务: 开启MULTI后所有命令入队, EXEC时一次性执行
// 注意: Redis事务不支持回滚!

stringRedisTemplate.execute(new SessionCallback<Object>() {
    @Override
    public <K, V> Object execute(RedisOperations<K, V> operations) {
        operations.multi(); // 开启事务
        operations.opsForValue().set("key1", "val1");
        operations.opsForValue().set("key2", "val2");
        operations.opsForValue().increment("counter");
        return operations.exec(); // 执行事务 -> [true, true, 1]
    }
});
```

## 12.2 WATCH乐观锁实现

```java
/**
 * WATCH + MULTI + EXEC = Redis乐观锁
 * WATCH监视key, 在EXEC前若key被其他客户端修改, 则EXEC返回null(事务失败)
 */
public boolean transfer(String from, String to, int amount) {
    // 1. WATCH账户余额
    stringRedisTemplate.watch(from);
    int balance = Integer.parseInt(
            Optional.ofNullable(stringRedisTemplate.opsForValue().get(from)).orElse("0"));
    if (balance < amount) {
        stringRedisTemplate.unwatch();
        return false;
    }
    // 2. 开启事务
    List<Object> results = stringRedisTemplate.execute(new SessionCallback<List<Object>>() {
        @Override
        public List<Object> execute(RedisOperations operations) {
            operations.multi();
            operations.opsForValue().decrement(from, amount);
            operations.opsForValue().increment(to, amount);
            return operations.exec(); // null = 事务被WATCH取消
        }
    });
    return results != null && !results.isEmpty();
}
```

```shell
# Redis CLI示例
> WATCH mykey
> MULTI
> SET mykey newvalue
> EXEC
(nil)  # 事务失败(mykey在WATCH后被修改)
```

## 12.3 Lua脚本 vs 事务

| 特性 | MULTI/EXEC事务 | Lua脚本 |
|------|---------------|---------|
| 原子性 | 支持(命令批量执行) | 支持(整个脚本原子) |
| 条件判断 | 需要WATCH | 脚本内直接判断(if/else) |
| 回滚 | 不支持 | 不支持 |
| 性能 | 一般 | 高(减少网络往返) |
| 复杂度 | 低 | 中 |
| 推荐场景 | 简单批量操作 | 复杂原子逻辑(分布式锁/限流) |

```lua
-- Lua脚本: 原子转账 (优于WATCH+MULTI)
-- KEYS[1]=from, KEYS[2]=to, ARGV[1]=amount
local balance = tonumber(redis.call('GET', KEYS[1])) or 0
if balance < tonumber(ARGV[1]) then return 0 end
redis.call('DECRBY', KEYS[1], ARGV[1])
redis.call('INCRBY', KEYS[2], ARGV[1])
return 1
```

# 13. Pipeline批量操作

## 13.1 Pipeline性能对比

```java
@Test
public void comparePipelinePerformance() {
    int batchSize = 10000;

    // 方式1: 逐条操作 (10000次网络往返)
    long start1 = System.currentTimeMillis();
    for (int i = 0; i < batchSize; i++) {
        stringRedisTemplate.opsForValue().set("test:noPipeline:" + i, "v" + i);
    }
    long cost1 = System.currentTimeMillis() - start1;
    System.out.println("逐条SET " + batchSize + "次, 耗时: " + cost1 + "ms");

    // 方式2: Pipeline批量操作 (1次网络往返)
    long start2 = System.currentTimeMillis();
    stringRedisTemplate.executePipelined(new SessionCallback<Object>() {
        @Override
        public <K, V> Object execute(RedisOperations<K, V> operations) {
            for (int i = 0; i < batchSize; i++) {
                operations.opsForValue().set("test:pipeline:" + i, "v" + i);
            }
            return null;
        }
    });
    long cost2 = System.currentTimeMillis() - start2;
    System.out.println("Pipeline SET " + batchSize + "次, 耗时: " + cost2 + "ms");
    System.out.println("提升倍数: " + (cost1 * 1.0 / cost2));
}

// 预期输出(参考):
// 逐条SET 10000次, 耗时: 8500ms
// Pipeline SET 10000次, 耗时: 320ms
// 提升倍数: ~26倍
```

## 13.2 批量查询与缓存预热

```java
/** Pipeline批量查询(解决缓存miss多次查Redis的问题) */
public Map<Long, Shop> batchQueryShops(List<Long> ids) {
    Map<Long, Shop> result = new HashMap<>();
    // 1. 先Pipeline批量查缓存
    List<Object> cached = stringRedisTemplate.executePipelined(
            new SessionCallback<Object>() {
                @Override
                public <K, V> Object execute(RedisOperations<K, V> ops) {
                    for (Long id : ids) {
                        ops.opsForValue().get("cache:shop:" + id);
                    }
                    return null;
                }
            });
    // 2. 区分命中与未命中
    List<Long> missedIds = new ArrayList<>();
    for (int i = 0; i < ids.size(); i++) {
        String json = (String) cached.get(i);
        if (StrUtil.isNotBlank(json)) {
            result.put(ids.get(i), JSONUtil.toBean(json, Shop.class));
        } else {
            missedIds.add(ids.get(i));
        }
    }
    // 3. 查DB并回写缓存
    if (!missedIds.isEmpty()) {
        List<Shop> shops = shopMapper.selectBatchIds(missedIds);
        for (Shop shop : shops) {
            result.put(shop.getId(), shop);
            stringRedisTemplate.opsForValue().set(
                    "cache:shop:" + shop.getId(),
                    JSONUtil.toJsonStr(shop), 30, TimeUnit.MINUTES);
        }
    }
    return result;
}
```

# 14. Redis实现限流算法

## 14.1 固定窗口计数器

```java
/** 固定时间窗口: 简单但有边界突刺问题 */
public boolean fixedWindow(String key, int limit, int windowSec) {
    Long count = stringRedisTemplate.opsForValue().increment(key);
    if (count != null && count == 1) {
        stringRedisTemplate.expire(key, windowSec, TimeUnit.SECONDS);
    }
    return count != null && count <= limit;
}

// 问题: 窗口边界 (如窗口最后1秒与下个窗口第1秒) 会有2倍请求通过
```

## 14.2 滑动窗口 (ZSET实现)

```java
/** 滑动窗口: 基于ZSET, 精确控制任意时间窗口内的请求数 */
public boolean slidingWindow(String key, int limit, int windowSec) {
    long now = System.currentTimeMillis();
    long windowStart = now - windowSec * 1000L;
    // 删除过期记录
    stringRedisTemplate.opsForZSet().removeRangeByScore(key, 0, windowStart);
    // 计数当前窗口请求数
    Long count = stringRedisTemplate.opsForZSet().zCard(key);
    if (count != null && count >= limit) return false;
    // 添加当前请求
    stringRedisTemplate.opsForZSet().add(key, String.valueOf(now), now);
    stringRedisTemplate.expire(key, windowSec * 2, TimeUnit.SECONDS);
    return true;
}
```

## 14.3 令牌桶算法 (Lua)

```lua
-- token_bucket.lua
-- KEYS[1]: 令牌桶key  KEYS[2]: 上次填充时间key
-- ARGV[1]: 容量(maxTokens)  ARGV[2]: 生成速率(tokens/sec)
-- ARGV[3]: 本次请求数量(通常为1)
-- 返回: 1=允许, 0=拒绝

local tokens = tonumber(redis.call('GET', KEYS[1])) or tonumber(ARGV[1])
local lastTime = tonumber(redis.call('GET', KEYS[2])) or 0
local now = redis.call('TIME')[1]
local elapsed = now - lastTime
local generated = elapsed * tonumber(ARGV[2])
tokens = math.min(tonumber(ARGV[1]), tokens + generated)
redis.call('SET', KEYS[2], now)

if tokens >= tonumber(ARGV[3]) then
    redis.call('SET', KEYS[1], tokens - tonumber(ARGV[3]))
    return 1
else
    redis.call('SET', KEYS[1], tokens)
    return 0
end
```

Java调用:
```java
public boolean tokenBucket(String key, int maxTokens, int ratePerSec) {
    String script = "-- token_bucket.lua  ..."; // 上方Lua脚本
    DefaultRedisScript<Long> redisScript = new DefaultRedisScript<>();
    redisScript.setScriptText(script);
    redisScript.setResultType(Long.class);
    Long result = stringRedisTemplate.execute(redisScript,
            Arrays.asList("tb:token:" + key, "tb:time:" + key),
            String.valueOf(maxTokens), String.valueOf(ratePerSec), "1");
    return result != null && result == 1L;
}
```

## 14.4 漏桶算法 (Lua)

```lua
-- leaky_bucket.lua
-- KEYS[1]: 桶key  KEYS[2]: 上次漏水时间key
-- ARGV[1]: 桶容量  ARGV[2]: 漏水速率(req/sec)
-- 返回: 剩余容量, 若<0则拒绝

local water = tonumber(redis.call('GET', KEYS[1])) or 0
local lastTime = tonumber(redis.call('GET', KEYS[2])) or 0
local now = redis.call('TIME')[1]
local elapsed = now - lastTime
-- 漏水
water = math.max(0, water - elapsed * tonumber(ARGV[2]))
redis.call('SET', KEYS[2], now)

if water < tonumber(ARGV[1]) then
    redis.call('SET', KEYS[1], water + 1)
    return 1
else
    redis.call('SET', KEYS[1], water)
    return 0
end
```

## 14.5 四种限流算法对比

| 算法 | 原理 | 优点 | 缺点 | 适用 |
|------|------|------|------|------|
| 固定窗口 | 时间窗口内计数 | 实现简单 | 边界突刺 | 低精度场景 |
| 滑动窗口 | ZSET按时间戳滑动 | 精确控制 | ZSET内存稍大 | 需要精确限流 |
| 令牌桶 | 固定速率生成令牌 | 允许突发;平滑 | 实现稍复杂 | API限流 |
| 漏桶 | 固定速率流出 | 流速绝对恒定 | 无法应对突发 | 严格控制速率 |

# 15. Redis实现分布式Session

## 15.1 Spring Session + Redis集成

```xml
<!-- 依赖 -->
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  session:
    store-type: redis
    timeout: 30m
    redis:
      namespace: spring:session
      flush-mode: on_save
  redis:
    host: localhost
    port: 6379
```

```java
@Configuration
@EnableRedisHttpSession(maxInactiveIntervalInSeconds = 1800)
public class SessionConfig {
    // Spring Session 自动管理:
    // - GET /user/me 时自动从Redis读取Session
    // - 修改Session时自动同步到Redis
    // - Session过期后自动清理
}
```

## 15.2 多服务实例Session共享架构

```
                [ Nginx/负载均衡 ]
                  /      |      \
         [App-1]  [App-2]  [App-3]
             \       |      /
              [ Redis Session ]
                   |
               [ MySQL ]
```

```java
// 任意实例写入Session, 所有实例都能读到
// App-1: session.setAttribute("user", userDTO);
// App-2: UserDTO user = (UserDTO) session.getAttribute("user"); // 读取成功

@RestController
public class SessionTestController {

    @GetMapping("/session/set")
    public String setSession(HttpSession session) {
        session.setAttribute("host", System.getenv("HOSTNAME"));
        return "Session stored in Redis, host=" + System.getenv("HOSTNAME");
    }

    @GetMapping("/session/get")
    public String getSession(HttpSession session) {
        Object host = session.getAttribute("host");
        return "Read from Redis, original host=" + host
                + ", current host=" + System.getenv("HOSTNAME");
    }
}
```

## 15.3 自定义Session的Cookie配置

```java
@Configuration
public class SessionCookieConfig {

    @Bean
    public CookieSerializer cookieSerializer() {
        DefaultCookieSerializer serializer = new DefaultCookieSerializer();
        serializer.setCookieName("SESSION_ID");
        serializer.setCookiePath("/");
        serializer.setDomainNamePattern("^.+?.(\\w+\\.[a-z]+)$"); // 支持跨子域
        serializer.setUseHttpOnlyCookie(true);
        serializer.setUseSecureCookie(false); // HTTPS环境设为true
        serializer.setCookieMaxAge(1800);     // 30分钟
        return serializer;
    }
}
```

