# 1. Spring Security 测试支持

## 1.1 依赖准备

`spring-boot-starter-test` 默认不包含安全测试模块，需要显式引入：

```xml
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-test</artifactId>
    <scope>test</scope>
</dependency>
```

## 1.2 典型作用

- 构造认证用户
- 模拟 CSRF
- 模拟 OAuth2/JWT
- 验证方法级权限

---

# 2. MockMvc 与安全

## 2.1 `@WithMockUser`

```java
@WithMockUser(username = "admin", roles = "ADMIN")
@Test
void adminOnlyApi_shouldPass() throws Exception {
    mockMvc.perform(get("/admin"))
        .andExpect(status().isOk());
}
```

## 2.2 请求后处理器

```java
mockMvc.perform(post("/orders")
        .with(user("u1").roles("USER"))
        .with(csrf())
)
.andExpect(status().isOk());
```

---

# 3. 方法级安全测试

## 3.1 `@WithUserDetails`

```java
@WithUserDetails("tom")
@Test
void methodSecurityTest() {}
```

适合验证基于数据库用户加载的权限。

## 3.2 自定义 SecurityContext

```java
@WithSecurityContext(factory = CustomSecurityContextFactory.class)
@Test
void customContextTest() {}
```

---

# 4. OAuth2 / JWT 测试

## 4.1 JWT

```java
mockMvc.perform(get("/api")
        .with(jwt().jwt(jwt -> jwt.claim("scope", "read")))
)
.andExpect(status().isOk());
```

## 4.2 OIDC 登录

```java
mockMvc.perform(get("/profile").with(oidcLogin()))
    .andExpect(status().isOk());
```

---

# 5. 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| 403 | 缺少 CSRF | 使用 `csrf()` |
| 401 | 未注入用户 | `@WithMockUser` 或 `user()` |
| 403（方法级） | 权限不足 | 检查 `roles/authorities` |
