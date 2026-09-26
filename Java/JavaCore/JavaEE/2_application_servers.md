
# 1. 应用服务器是什么

## 1.1 定义

应用服务器（Application Server）是**完整实现 JavaEE/Jakarta EE 全部规范**的服务端软件。它既是 WEB 服务器，也是 EJB 容器、CDI 容器、JPA 容器、事务管理器等。

```
┌─────────────────────────────────────────────┐
│              应用服务器                      │
│  ┌─────────┐ ┌─────────┐ ┌──────────────┐   │
│  │WEB容器   │ │EJB容器   │ │CDI容器       │   │
│  │(Servlet │ │(会话Bean │ │(依赖注入/事件)│   │
│  │ JSP... )│ │ MDB... )│ │              │   │
│  ├─────────┴─┴─────────┴─┴──────────────┤   │
│  │  事务管理器(JTA) · JNDI · JMS 等内置服务 │   │
│  └──────────────────────────────────────┘   │
```

## 1.2 应用服务器 vs WEB 服务器

| 对比项 | WEB 服务器 | 应用服务器 |
|--------|-----------|-----------|
| 实现的规范 | Servlet + JSP 等 Web 规范 | JavaEE 全部规范 |
| EJB 支持 | 无 | 完整 |
| JTA 分布式事务 | 无 | 支持 |
| JMS | 无 | 内置消息服务 |
| JNDI | 简单 | 完整命名服务 |
| 典型代表 | Tomcat、Jetty | GlassFish、WildFly、WebLogic、WebSphere |
| 典型场景 | 轻量 Web 应用、Spring Boot 内嵌 | 大型企业级应用 |

## 1.3 关键结论

- **应用服务器包含 WEB 服务器能力**：如 WildFly（JBoss）内部集成了基于 Undertow 的 Servlet 容器。
- **Tomcat 严格说是"Servlet 容器 + WEB 服务器"**，不是完整的应用服务器（不支持 EJB 等）。
- Spring Boot 项目通常用 Tomcat/Jetty/Undertow 内嵌启动，**不需要**完整应用服务器——这也是应用服务器在云原生时代逐渐式微的原因之一。

# 2. 主流应用服务器对比

| 产品 | 厂商/社区 | 是否开源 | JavaEE/Jakarta EE 支持 | 特点 |
|------|-----------|----------|------------------------|------|
| **GlassFish** | Oracle → Eclipse | 开源 | 完整（曾是参考实现 RI） | 官方参考实现，学习用首选 |
| **Payara Server** | Payara 社区 | 开源 | 完整 | GlassFish 衍生，企业增强版 |
| **WildFly** | Red Hat | 开源 | 完整 | 前身 JBoss AS，模块化架构，性能好 |
| **JBoss EAP** | Red Hat | 商业 | 完整 | WildFly 商业稳定版 |
| **WebLogic** | Oracle | 商业 | 完整 | 老牌企业级，金融/电信行业使用多 |
| **WebSphere** | IBM | 商业 | 完整 | IBM 企业生态，银行业使用多 |
| **Open Liberty** | IBM | 开源 | 完整 | 轻量、启动快、云原生友好 |
| **Apache TomEE** | Apache | 开源 | 完整 | 基于 Tomcat + OpenEJB，轻量 |
| **Payara Micro / WildFly Core** | - | 开源 | 部分 | 嵌入式/微服务形态 |

## 2.1 选型建议

```
学习/验证规范   → GlassFish、TomEE
企业开源部署    → WildFly、Payara
传统大企业      → WebLogic、WebSphere（存量系统）
云原生/微服务   → Open Liberty、Tomcat + Spring Boot（JavaEE 规范被 Spring 覆盖）
```

# 3. 参考实现（Reference Implementation, RI）

## 3.1 什么是 RI

每个 JavaEE/Jakarta EE 规范都会指定一个**参考实现**，用于验证规范的可行性，并作为其他实现的对照标准。

| 规范 | 参考实现 |
|------|----------|
| 全平台 | GlassFish（JavaEE 6~8 时代的平台 RI） |
| JPA | EclipseLink |
| JAX-RS | Jersey |
| CDI | Weld |
| Bean Validation | Hibernate Validator |
| JMS | OpenMQ（GlassFish 内置） |
| JAX-WS | Metro |

## 3.2 TCK（技术兼容性工具包）

- 应用服务器必须通过 **TCK（Technology Compatibility Kit）** 测试，才能宣称自己是"兼容的 Jakarta EE 实现"。
- TCK 保证"一份应用，到处运行"的可移植性承诺。

# 4. 应用服务器内置的核心服务

## 4.1 JNDI 命名服务

应用服务器启动时会把各种资源（数据源、JMS 连接工厂、EJB 等）注册到 JNDI 命名空间中，应用通过名称查找：

```
java:comp/env/jdbc/MyDS      # 数据源
java:global/MyApp/HelloEJB   # 全局 EJB
java:app/...                 # 应用级
java:module/...              # 模块级
```

## 4.2 数据源与连接池

应用服务器（如 GlassFish 管理控制台）可以配置 JDBC 连接池，应用通过 JNDI 获取：

```java
Context ctx = new InitialContext();
DataSource ds = (DataSource) ctx.lookup("java:comp/env/jdbc/MyDS");
Connection conn = ds.getConnection();
```

## 4.3 内置 JMS 提供者

- GlassFish 内置 **OpenMQ**。
- WildFly 内置 **ActiveMQ Artemis**。
- 也支持接入外部 MQ 作为 JMS 提供者。

## 4.4 安全管理

- 支持基于容器的声明式安全：在 `web.xml` / `ejb-jar.xml` 中声明角色与访问规则，由容器执行认证与授权。
- 支持 JDBC/LDAP 等领域的登录认证。

# 5. 部署单元与目录结构

## 5.1 三种部署单元

| 部署单元 | 后缀 | 内容 | 用途 |
|----------|------|------|------|
| WAR | `.war` | Web 应用（Servlet/JSP/HTML） | Web 应用 |
| JAR | `.jar` | EJB 模块 / 普通类库 | 业务逻辑模块 |
| EAR | `.ear` | 企业应用归档，包含多个 WAR/JAR + application.xml | 完整企业应用 |

```
myapp.ear
├── META-INF/
│   └── application.xml       # EAR 部署描述符
├── myapp-web.war             # Web 模块
├── myapp-ejb.jar             # EJB 模块
└── lib/                      # 公共类库
```

> 详细打包与部署见 `23_packaging_deployment.md`。

## 5.2 部署描述符 vs 注解

- 早期版本依赖 XML 部署描述符（`web.xml`、`ejb-jar.xml`）。
- JavaEE 6 开始推荐使用**注解**（`@WebServlet`、`@Stateless` 等），描述符成为可选。
- 注解可被描述符覆盖（描述符优先级更高），用于不改代码调整部署环境。

# 6. 嵌入式应用服务器与微服务形态

## 6.1 嵌入式场景

现代开发中，应用服务器以内嵌方式运行，与 Spring Boot 一体：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-tomcat</artifactId>
</dependency>
```

- Spring Boot 默认内嵌 Tomcat；也可切换 Jetty 或 Undertow。
- 内嵌形态下应用就是一个可执行 JAR，无需外部服务器，契合容器化/云原生。

## 6.2 WildFly 的微服务形态

- **WildFly Bootable JAR**：把 WildFly 与应用打进一个可执行 JAR。
- **Open Liberty**：提供了 `springBootApplication` 等轻量启动方式。

## 6.3 企业实践趋势

```
传统模式：  独立应用服务器 + 手工部署 EAR/WAR
现代模式：  可执行 JAR + 内嵌容器 + 容器编排（Docker/K8s）
```

- 应用服务器仍然存在，但更多以"嵌入式/微服务"形态出现，或被 Spring Boot 取代。

# 7. 实战：快速体验应用服务器

## 7.1 用 GlassFish 部署一个 WAR

1. 下载 GlassFish（如 `glassfish-7.x.zip`）并解压。
2. 启动：进入 `bin` 目录执行 `asadmin start-domain`。
3. 部署应用：`asadmin deploy myapp.war`。
4. 访问管理控制台：`http://localhost:4848`（默认管理员密码为空/`admin`）。
5. 通过管理控制台配置 JDBC 连接池、JMS 资源等。

## 7.2 用 Payara Micro 运行应用

```bash
# 无需安装，直接运行可执行 JAR
java -jar payara-micro.jar --deploy myapp.war
```

## 7.3 注意事项

- 不同服务器部署结构不同（如 JBoss 也有 `deployments/` 目录、`standalone.xml` 配置）。
- 同一 WAR 在不同应用服务器上的可移植性，依赖于是否只用了标准规范 API。

# 8. 常见问题排查清单

| 问题 | 排查方向 |
|------|----------|
| 应用无法启动 | 查看服务器日志（如 GlassFish `logs/server.log`、WildFly `standalone/log/server.log`） |
| JNDI 资源找不到 | 确认资源是否已注册、JNDI 名称拼写、`java:comp/env` 是否配置了 `resource-ref` |
| 数据源连接失败 | 检查连接池配置、驱动 JAR、数据库账号权限 |
| EJB 部署失败 | 检查 EJB JAR 是否打入 EAR、接口与实现是否匹配 |
| 端口冲突 | 修改服务器监听端口（GlassFish `asadmin set server.http-service.http-listener.http-listener-1.port=8081`） |
