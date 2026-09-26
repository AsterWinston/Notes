
# 1. 概述

## 1.1 什么是 JNDI

JNDI（Java Naming and Directory Interface，Java 命名与目录接口）是 JavaEE 的**命名/目录服务规范**，为 Java 应用提供统一的**按名称查找对象**的机制。参考实现由各应用服务器提供。

```
应用代码
   │  lookup("java:comp/env/jdbc/MyDS")
   ▼
┌──────────────────────────────┐
│         JNDI API            │  ← 程序员只面对 JNDI 接口
├──────────────────────────────┤
│   JNDI SPI（服务提供接口）    │  ← 具体实现接入点
├───────────┬──────────────────┤
│ LDAP 实现  │  RMI 实现   │ 文件系统实现│
└───────────┴──────────────┴────────────┘
```

- 通过 JNDI，应用**不依赖具体实现**，只依赖"名字"——把资源的创建、绑定、管理交给容器。

## 1.2 JNDI 能绑定什么

| 资源类型 | 示例 |
|----------|------|
| 数据源 | `DataSource`（连接池） |
| JMS 资源 | `ConnectionFactory`、`Queue`、`Topic` |
| EJB | 会话 Bean 的远程/本地引用 |
| JavaMail | `Session` |
| 环境条目 | 配置字符串、数字（`<env-entry>`） |
| 自定义对象 | 任意可绑定对象 |

## 1.3 为什么用 JNDI

- **解耦**：代码不写死具体实现类，只写逻辑名称。
- **配置集中**：资源的创建与参数由容器/配置文件统一管理。
- **可移植**：同一个应用部署到不同服务器，只需调整服务器侧的资源绑定。

# 2. 核心 API

## 2.1 `InitialContext` 与 `lookup`

```java
Context ctx = new InitialContext();
DataSource ds = (DataSource) ctx.lookup("java:comp/env/jdbc/MyDS");
Connection conn = ds.getConnection();
```

## 2.2 常用操作

| 方法 | 作用 |
|------|------|
| `lookup(String name)` | 按名称查找对象 |
| `bind(String name, Object obj)` | 绑定对象到名称 |
| `rebind(String name, Object obj)` | 重新绑定（覆盖） |
| `unbind(String name)` | 解绑 |
| `list(String name)` | 列出子上下文 |
| `createSubcontext(String name)` | 创建子上下文 |
| `close()` | 关闭上下文 |

## 2.3 上下文对象

| 接口 | 说明 |
|------|------|
| `Context` | 命名上下文，绑定/查找对象 |
| `InitialContext` | 初始上下文入口 |
| `DirContext` | 目录上下文（LDAP，支持属性查询） |
| `EventContext` / `EventDirContext` | 命名/目录事件 |

# 3. JNDI 命名空间

## 3.1 应用服务器中的标准命名空间

| 命名空间 | 含义 | 示例 |
|----------|------|------|
| `java:comp/env` | **组件环境命名空间**（标准用法） | `java:comp/env/jdbc/MyDS` |
| `java:comp/` | 组件相关的系统引用 | `java:comp/UserTransaction` |
| `java:app/` | 应用（EAR）级命名空间 | `java:app/MyApp/OrderEJB` |
| `java:module/` | 模块级命名空间 | `java:module/OrderEJB` |
| `java:global/` | 全局命名空间（跨应用） | `java:global/MyApp/OrderEJB` |
| `java:jboss/`、`corba/` 等 | 厂商特定命名空间 | 与服务器相关 |

## 3.2 `java:comp/env` 详解

- `java:comp/env` 是**应用代码应该使用的标准命名空间**，它避免代码直接依赖服务器特定命名空间。
- 应用通过 `<resource-ref>` / `<env-entry>` 描述符把逻辑名映射到服务器实际资源。

# 4. 数据源资源引用配置

## 4.1 web.xml 中声明资源引用

```xml
<web-app>
    <resource-ref>
        <res-ref-name>jdbc/MyDS</res-ref-name>
        <res-type>javax.sql.DataSource</res-type>
        <res-auth>Container</res-auth>
    </resource-ref>

    <env-entry>
        <env-entry-name>maxPageSize</env-entry-name>
        <env-entry-type>java.lang.Integer</env-entry-type>
        <env-entry-value>20</env-entry-value>
    </env-entry>
</web-app>
```

- `res-auth=Container`：连接由容器管理（自动获取/归还，配合事务）。
- `res-auth=Application`：应用自己通过 `getConnection(user, pwd)` 管理。

## 4.2 代码中查找

```java
Context ctx = new InitialContext();
DataSource ds = (DataSource) ctx.lookup("java:comp/env/jdbc/MyDS");
```

## 4.3 服务器侧绑定

**GlassFish 管理控制台** 或 `asadmin`：

```bash
asadmin create-jdbc-connection-pool --datasourceclassname com.mysql.cj.jdbc.MysqlDataSource \
    --restype javax.sql.DataSource mysql_pool
asadmin create-jdbc-resource --connectionpoolid mysql_pool jdbc/MyDS
```

**WildFly（standalone.xml）**：

```xml
<datasource jndi-name="java:jboss/datasources/MyDS" pool-name="MyDS">
    <connection-url>jdbc:mysql://localhost:3306/db</connection-url>
    <driver>mysql</driver>
    <security><user-name>root</user-name><password>123456</password></security>
</datasource>
```

# 5. JNDI 在 JavaEE 各场景的应用

## 5.1 查找 EJB

```java
// 远程 EJB
Context ctx = new InitialContext();
OrderService ejb = (OrderService) ctx.lookup(
        "java:global/MyApp/OrderServiceBean!com.example.OrderService");
```

## 5.2 查找 JMS 资源

```java
ConnectionFactory factory =
        (ConnectionFactory) ctx.lookup("java:comp/env/jms/ConnectionFactory");
Queue queue = (Queue) ctx.lookup("java:comp/env/jms/OrderQueue");
```

## 5.3 查找 JavaMail Session

```java
Session mailSession = (Session) ctx.lookup("java:comp/env/mail/MyMailSession");
```

# 6. 使用 `@Resource` 注入替代手动 lookup

手动 `lookup` 繁琐且易错，容器组件可以直接注入：

```java
public class DataSourceClient {
    @Resource(lookup = "java:comp/env/jdbc/MyDS")
    private DataSource dataSource;
}
```

- `@Resource` 等价于自动查找并注入。
- 支持注入：`DataSource`、`ConnectionFactory`、`Queue`、`Session`、`UserTransaction`、EJB 等。

# 7. JNDI 在普通 Java 程序中的使用

非 JavaEE 环境也可以使用 JNDI（如绑定到文件系统、LDAP）：

```java
Hashtable<String, String> env = new Hashtable<>();
env.put(Context.INITIAL_CONTEXT_FACTORY, "com.sun.jndi.fscontext.RefFSContextFactory");
Context ctx = new InitialContext(env);
ctx.bind("/temp/config", "hello");
System.out.println(ctx.lookup("/temp/config"));   // 输出 hello
```

- 需要对应 SPI 实现（如 fscontext）。
- JavaEE 容器内置了完整 JNDI 实现，普通应用通常不直接使用。

# 8. 常见问题排查清单

| 问题 | 原因/排查 |
|------|----------|
| `NameNotFoundException` | JNDI 名称拼写错误，或资源未在服务器注册 |
| `ClassCastException` | 查找返回的类型与强转类型不符（如接口/实现不匹配） |
| 缺少 `java:comp/env` | 未在描述符中声明 `resource-ref`，或使用了错误命名空间 |
| 在普通 main 中 lookup 失败 | 无 JNDI 提供者环境，需设置 `INITIAL_CONTEXT_FACTORY` |
| 连接池获取不到连接 | 检查服务器侧数据源配置、驱动、账号密码 |

# 9. JNDI vs Spring

| JNDI | Spring |
|------|--------|
| 容器（服务器）提供命名服务 | Spring 容器自身管理 Bean 命名 |
| `lookup()` / `@Resource` | `@Autowired` / `@Value` |
| 服务器侧配置资源 | 应用内 `application.yml` 配置 |
| 强依赖部署环境 | 独立运行（内嵌连接池如 HikariCP） |

- Spring Boot 默认**不依赖 JNDI**，直接用 HikariCP 等内嵌连接池。
- 传统 Spring（部署到应用服务器）可用 `<jee:jndi-lookup>` 或 `@Resource` 访问服务器资源。
