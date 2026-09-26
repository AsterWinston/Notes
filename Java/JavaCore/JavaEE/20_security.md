
# 1. 概述

## 1.1 安全体系结构

JavaEE 安全由多个相关规范组成：

```
┌────────────────────────────────────────────┐
│          JavaEE 安全体系                   │
│  ┌────────────┐  ┌────────────┐            │
│  │ 认证(AuthN) │  │ 授权(AuthZ) │            │
│  │ Jakarta    │  │ Jakarta    │            │
│  │ Security   │  │ Security   │            │
│  │ (JASPIC)   │  │ (JACC)     │            │
│  └────────────┘  └────────────┘            │
│  ┌──────────────────────────────────┐      │
│  │ JAAS（Java 认证与授权服务）       │      │
│  │ LoginModule / Subject / Principal│      │
│  └──────────────────────────────────┘      │
│  ┌──────────────────────────────────┐      │
│  │ 声明式安全（web.xml / 注解）     │      │
│  └──────────────────────────────────┘      │
└────────────────────────────────────────────┘
```

| 规范 | 原简称 | 作用 |
|------|--------|------|
| Jakarta Authentication | JASPIC | 可插拔认证机制（ServerAuthModule） |
| Jakarta Authorization | JACC | 访问决策（Policy） |
| Jakarta Security | - | 统一安全 API（JavaEE 8 新增） |
| JAAS | - | 认证/授权基础设施（LoginModule） |

## 1.2 三个安全概念

| 概念 | 说明 |
|------|------|
| 身份（Principal） | 已认证用户的身份 |
| 角色（Role） | 逻辑权限分组（admin、user） |
| 领域（Realm） | 用户/角色数据来源（JDBC、LDAP、File） |

# 2. 声明式安全（Declarative Security）

## 2.1 web.xml 认证配置

```xml
<web-app>
    <security-constraint>
        <web-resource-collection>
            <web-resource-name>admin_area</web-resource-name>
            <url-pattern>/admin/*</url-pattern>
        </web-resource-collection>
        <auth-constraint>
            <role-name>admin</role-name>
        </auth-constraint>
    </security-constraint>

    <login-config>
        <auth-method>FORM</auth-method>
        <form-login-config>
            <form-login-page>/login.html</form-login-page>
            <form-error-page>/login-error.html</form-error-page>
        </form-login-config>
    </login-config>

    <security-role>
        <role-name>admin</role-name>
        <role-name>user</role-name>
    </security-role>
</web-app>
```

## 2.2 认证方式（auth-method）

| 方式 | 说明 |
|------|------|
| `BASIC` | HTTP 基础认证（Base64 明文，配合 HTTPS） |
| `DIGEST` | 摘要认证 |
| `FORM` | 自定义登录表单（主流） |
| `CLIENT-CERT` | 客户端证书（双向 TLS） |

## 2.3 授权注解

```java
@WebServlet("/admin/delete")
@RolesAllowed({"admin"})
public class DeleteServlet extends HttpServlet { }

// 或
@DenyAll        // 拒绝所有
@PermitAll      // 允许所有（已认证）
@RunAs("admin") // 以指定角色运行
```

# 3. 编程式安全（Programmatic Security）

## 3.1 Servlet 中获取身份信息

```java
public void doGet(HttpServletRequest req, HttpServletResponse resp) {
    // 认证状态
    boolean auth = req.isUserInRole("admin");
    Principal user = req.getUserPrincipal();

    // 认证（BASIC/FORM 登录）
    req.login(username, password);
    // 退出
    req.logout();

    // 会话中的身份信息
    String name = (String) req.getSession().getAttribute("username");
}
```

## 3.2 EJB 中获取身份信息

```java
@Stateless
public class AuditService {

    @Resource
    private SessionContext ctx;

    public void audit() {
        Principal principal = ctx.getCallerPrincipal();
        boolean isAdmin = ctx.isCallerInRole("admin");
        // ctx.getCallerPrincipal() 返回当前调用者
    }
}
```

# 4. Realm（领域）配置

## 4.1 JDBC Realm（从数据库认证）

以 GlassFish/WildFly 为例配置 JDBC Realm，从用户表加载认证信息：

```xml
<security-domain name="jdbc-realm" cache-type="default">
    <authentication>
        <login-module code="org.jboss.security.auth.spi.DatabaseServerLoginModule"
                      flag="required">
            <module-option name="dsJndiName"
                    value="java:jboss/datasources/SecurityDS"/>
            <module-option name="principalsQuery"
                    value="SELECT password FROM users WHERE username=? AND enabled=1"/>
            <module-option name="rolesQuery"
                    value="SELECT role FROM user_roles WHERE username=?"/>
        </login-module>
    </authentication>
</security-domain>
```

## 4.2 LDAP Realm / File Realm

- **LDAP Realm**：企业统一身份目录（Active Directory、OpenLDAP）。
- **File Realm**：`tomcat-users.xml` 等文件方式（Tomcat 默认）。

# 5. JASPIC（Jakarta Authentication）

## 5.1 什么是 JASPIC

JASPIC 定义可插拔的认证机制：应用服务器把认证交给自定义的 **ServerAuthModule（SAM）**，实现自定义登录流程（如对接第三方 SSO、自定义 Token）。

```
请求
  │
  ▼
容器调用 ServerAuthModule.validateRequest()
  │
  ├─ 认证成功 → 建立 Subject/Principal → 继续业务
  ├─ 认证失败 → 返回 401
  └─ 无凭据  → 挑战（返回登录页/401）
```

## 5.2 注册 AuthConfigProvider

- 通过 `AuthConfigFactory` 注册 SAM。
- 或用供应商提供的集成方式（如 WildFly 的 security domain 配置）。

# 6. JACC（Jakarta Authorization）

- JACC 定义标准化的**授权策略**（`Policy`）。
- 容器把 web.xml/EJB 安全约束翻译为 JACC Permission，通过 `PolicyProvider` 决策。
- 大多数场景无需直接接触 JACC，由容器/框架（如 WildFly Elytron）实现。

# 7. JAAS（Java Authentication and Authorization Service）

## 7.1 核心 API

| API | 作用 |
|-----|------|
| `Subject` | 认证主体（用户 + 多个 Principal） |
| `Principal` | 身份（如用户名） |
| `Credential` | 凭证（密码、证书等） |
| `LoginContext` | 认证入口 |
| `LoginModule` | 具体认证逻辑 |
| `CallbackHandler` | 与用户交互（取用户名/密码） |

## 7.2 自定义 LoginModule 示例

```java
public class MyLoginModule implements LoginModule {
    private Subject subject;
    private CallbackHandler callbackHandler;
    private boolean succeeded;

    @Override
    public boolean login() throws LoginException {
        NameCallback nameCb = new NameCallback("用户名");
        PasswordCallback pwdCb = new PasswordCallback("密码", false);
        try {
            callbackHandler.handle(new Callback[]{nameCb, pwdCb});
            String name = nameCb.getName();
            char[] pwd = pwdCb.getPassword();

            // 校验用户名密码（查数据库等）
            if ("admin".equals(name) && "123456".equals(new String(pwd))) {
                succeeded = true;
                return true;
            }
            return false;
        } catch (Exception e) {
            throw new LoginException("认证失败");
        }
    }

    @Override
    public boolean commit() {
        if (succeeded) {
            subject.getPrincipals().add(new UserPrincipal("admin"));
        }
        return succeeded;
    }
    // logout() / abort() 省略
}
```

## 7.3 使用 LoginContext

```java
LoginContext ctx = new LoginContext("MyApp", new CallbackHandler() {
    @Override
    public void handle(Callback[] callbacks) throws IOException, UnsupportedCallbackException {
        for (Callback cb : callbacks) {
            if (cb instanceof NameCallback n) {
                n.setName("admin");
            } else if (cb instanceof PasswordCallback p) {
                p.setPassword("123456".toCharArray());
            }
        }
    }
});

ctx.login();          // 认证
Subject subject = ctx.getSubject();
ctx.logout();         // 退出
```

# 8. JavaEE 8+ 新安全 API（Jakarta Security）

```java
// 获取当前身份（SecurityContext）
@RequestScoped
public class CurrentUser {
    @Inject
    private SecurityContext securityContext;

    public boolean isAdmin() {
        return securityContext.isCallerInRole("admin");
    }

    public Principal getPrincipal() {
        return securityContext.getCallerPrincipal();
    }
}
```

# 9. 常见漏洞防护

| 漏洞 | JavaEE 中的防护 |
|------|----------------|
| 会话固定 | 登录后 `request.changeSessionId()` / 重建会话 |
| CSRF | 校验 Token（`jakarta.servlet.http` 无内置，需框架/Filter 实现） |
| XSS | 输出编码（JSP `<c:out>`、框架转义） |
| SQL 注入 | `PreparedStatement`、JPA 参数化 |
| 敏感数据泄露 | HTTPS、密码哈希（BCrypt 等） |

> 详细 Web 安全见 `Security/` 目录与 `Interview/60_web_security.md`、`Interview/61_crypto.md`。

# 10. JavaEE 安全 vs Spring Security

| JavaEE | Spring Security |
|--------|-----------------|
| web.xml `security-constraint` | `SecurityFilterChain` 配置 |
| `@RolesAllowed` | `@PreAuthorize` |
| FORM/BASIC 认证 | 表单登录、OAuth2、JWT |
| Realm | `UserDetailsService` / `AuthenticationProvider` |
| JASPIC | 过滤器链自定义认证 |
| 安全注解（`jakarta.annotation.security`） | Spring Security 注解 |

> 现代 Spring 项目统一使用 Spring Security，JavaEE 安全机制主要在传统应用服务器生态与存量系统中出现。
