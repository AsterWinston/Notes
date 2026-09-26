
# 1. 什么是 JavaEE

## 1.1 Java 三大技术平台

Java 并非只有一种版本，而是根据应用领域划分为三大技术平台：

| 平台 | 全称 | 定位 | 典型内容 |
|------|------|------|----------|
| **JavaSE** | Java Standard Edition | 标准版，Java 的基础 | 集合、IO、多线程、网络编程、JVM |
| **JavaEE** | Java Enterprise Edition | 企业版，WEB/企业级应用 | Servlet、JSP、EJB、JPA、JTA 、JDBC等规范 |
| **JavaME** | Java Micro Edition | 微型版，嵌入式设备 | 机顶盒、智能家电等设备程序 |

- JavaSE 是基础，JavaEE 构建在 JavaSE 之上。
- JavaEE 不是一套具体的软件，而是一套**规范（Specification）集合**，它规定了企业级应用应该提供哪些能力，但不给出具体实现。

## 1.2 JavaEE 的本质：规范而非实现

```
JavaEE 平台 = 一组规范（接口 + 文档 + 配置约定）
                 │
        ┌────────┼────────────┐
        ▼        ▼            ▼
    应用服务器实现        厂商独立实现      开发者使用
  (GlassFish/WildFly)   (Hibernate/Jersey)  (针对接口编程)
```

- 规范定义"做什么"（接口、注解、配置格式）。
- 实现方负责"怎么做"（具体类库、容器行为）。
- 开发者针对规范编程，使应用可以部署到任何实现了该规范的服务器上——这就是 JavaEE 的核心价值：**可移植性与解耦合**。

## 1.3 为什么叫"企业版"

面向企业级应用（如银行、电商、ERP、OA）普遍存在的共性需求：

- 高并发、分布式事务
- 消息异步通信
- 组件复用与依赖管理
- 安全认证授权
- 持久化与缓存
- 定时任务与批处理

JavaEE 把这些问题抽象为统一的标准 API，避免每个企业重复造轮子。

# 2. JavaEE 发展历史

```
Sun Microsystems 时期：
  1998  J2EE 1.2（Java 2 Platform Enterprise Edition）
  2001  J2EE 1.3（引入 EJB 2.0、J2EE 连接器）
  2003  J2EE 1.4（引入 JAX-RPC、JMS 1.1）
  2006  JavaEE 5（引入 EJB 3.0 注解、JSF、JPA）

Oracle 时期：
  2009   Oracle 收购 Sun，JavaEE 进入 Oracle 时代
  2009   JavaEE 6（引入 CDI、JAX-RS、Bean Validation）
  2013   JavaEE 7（JMS 2.0、WebSocket、Batch）
  2017   JavaEE 8（Servlet 4.0、JSON-P、JAX-RS 2.1）

Eclipse 基金会时期：
  2018   Oracle 将 JavaEE 捐赠给 Eclipse 基金会
  2019   更名 Jakarta EE，JavaEE 9 = Jakarta EE 8
  2021   Jakarta EE 9（包名 javax → jakarta）
  2022   Jakarta EE 10（全面支持 Java 17）
  2023   Jakarta EE 11（支持 Java 21，Servlet 6.1 等）
```

## 2.1 关键转折：javax → jakarta

JavaEE 被 Oracle 捐赠给 Eclipse 基金会后，由于 Oracle 保留了 `javax` 商标，所有 API 包名被迫从 `javax.*` 改为 `jakarta.*`：

| 迁移项 | JavaEE 8 | Jakarta EE 9+ |
|--------|----------|---------------|
| Servlet | `javax.servlet.Servlet` | `jakarta.servlet.Servlet` |
| JPA | `javax.persistence.Entity` | `jakarta.persistence.Entity` |
| JMS | `javax.jms.ConnectionFactory` | `jakarta.jms.ConnectionFactory` |
| 事务 | `javax.transaction.UserTransaction` | `jakarta.transaction.UserTransaction` |

- **影响**：基于 `javax` 的旧应用无法直接部署到 Jakarta EE 9+ 服务器；依赖 Servlet API 的框架（Spring 6 / Spring Boot 3）也随之升级到 `jakarta` 命名空间。
- **注意**：仅包名变化，API 语义基本不变，迁移主要是 import 替换。

## 2.2 JavaEE 版本与 Tomcat 兼容性速查

| JavaEE/Jakarta EE 版本 | Servlet 版本 | Tomcat 兼容版本 |
|------------------------|--------------|-----------------|
| JavaEE 5 | Servlet 2.5 | Tomcat 6 |
| JavaEE 6 | Servlet 3.0 | Tomcat 7 |
| JavaEE 7 | Servlet 3.1 | Tomcat 8 |
| JavaEE 8 | Servlet 4.0 | Tomcat 9 |
| Jakarta EE 9 | Servlet 5.0 | Tomcat 10 |
| Jakarta EE 10 | Servlet 6.0 | Tomcat 10.1 |

# 3. JavaEE 的 13 种核心规范

历史上常说"JavaEE 包含 13 种规范"，但这一数字随版本不断变化。以下是最常被引用的 13 种核心规范（JavaEE 8 时代）：

| # | 规范 | 作用 |
|---|------|------|
| 1 | **Servlet** | Web 请求处理基础组件 |
| 2 | **JSP** | 服务端页面技术 |
| 3 | **EL** | 表达式语言，简化页面取值 |
| 4 | **JSTL** | JSP 标准标签库 |
| 5 | **EJB** | 企业级业务组件 |
| 6 | **JPA** | Java 持久化 API（ORM 规范） |
| 7 | **JTA** | Java 事务 API |
| 8 | **JMS** | Java 消息服务 |
| 9 | **JNDI** | 命名与目录服务 |
| 10 | **JavaMail** | 邮件发送/接收 |
| 11 | **JAX-WS** | 基于 SOAP 的 Web 服务 |
| 12 | **JAX-RS** | 基于 RESTful 的 Web 服务 |
| 13 | **JAXB / JAXP** | XML 处理与绑定 |

## 3.1 各版本规范数量的演变

- JavaEE 5：约 22 个规范
- JavaEE 6：约 24 个规范
- JavaEE 7：约 26 个规范
- JavaEE 8：约 26 个规范
- Jakarta EE 10：38 个规范

# 4. Jakarta EE 规范全景

## 4.1 按领域分类

```
┌────────────────────────────────────────────────────────────┐
│                       Jakarta EE 规范全景                    │
├────────────┬───────────────────────────────────────────────┤
│ Web 层     │ Servlet · JSP · EL · JSTL · JSF · WebSocket   │
├────────────┼───────────────────────────────────────────────┤
│ 业务层     │ EJB · CDI · 依赖注入 · 拦截器 · 托管Bean · RMI │
├────────────┼───────────────────────────────────────────────┤
│ 数据层     │ JDBC · JPA · Bean Validation                  │
├────────────┼───────────────────────────────────────────────┤
│ 事务/并发  │ JTA · Jakarta Concurrency · Batch             │
├────────────┼───────────────────────────────────────────────┤
│ 集成层     │ JMS · JavaMail · JCA · JNDI                   │
├────────────┼───────────────────────────────────────────────┤
│ Web 服务   │ JAX-WS · JAX-RS · SAAJ · JAXB · JAXP          │
├────────────┼───────────────────────────────────────────────┤
│ 数据格式   │ JSON-P · JSON-B · JAF                          │
├────────────┼───────────────────────────────────────────────┤
│ 安全       │ Jakarta Security(JASPIC/JACC) · JAAS          │
└────────────┴───────────────────────────────────────────────┘
```

## 4.2 完整规范清单（Jakarta EE 10）

| 分组 | 规范名称 | 原简称 |
|------|----------|--------|
| Web | Jakarta Servlet | Servlet |
| Web | Jakarta Server Pages | JSP |
| Web | Jakarta Expression Language | EL |
| Web | Jakarta Standard Tag Library | JSTL |
| Web | Jakarta Server Faces | JSF |
| Web | Jakarta WebSocket | WebSocket |
| Web | Jakarta Pages Tag Library | JSPTL |
| 组件 | Jakarta Enterprise Beans | EJB |
| 组件 | Jakarta Contexts and Dependency Injection | CDI |
| 组件 | Jakarta Dependency Injection | DI (JSR-330) |
| 组件 | Jakarta Interceptors | Interceptors |
| 组件 | Jakarta Managed Beans | Managed Beans |
| 组件 | Jakarta Annotations | Common Annotations |
| 组件 | Jakarta RESTful Web Services | JAX-RS |
| 组件 | Jakarta XML Web Services | JAX-WS |
| 组件 | Jakarta SOAP with Attachments | SAAJ |
| 数据 | Jakarta Persistence | JPA |
| 数据 | Jakarta Bean Validation | BV |
| 数据 | Jakarta JDBC | JDBC |
| 数据 | Jakarta Connectors | JCA |
| 数据 | Jakarta Transaction | JTA |
| 数据 | Jakarta Messaging | JMS |
| 数据 | Jakarta Mail | JavaMail |
| 数据 | Jakarta Activation | JAF |
| 数据 | Jakarta XML Binding | JAXB |
| 数据 | Jakarta XML Processing | JAXP |
| 数据 | Jakarta JSON Processing | JSON-P |
| 数据 | Jakarta JSON Binding | JSON-B |
| 并发 | Jakarta Concurrency | Concurrency |
| 并发 | Jakarta Batch | Batch |
| 安全 | Jakarta Authentication | JASPIC |
| 安全 | Jakarta Authorization | JACC |
| 安全 | Jakarta Security | Security |
| 部署 | Jakarta Enterprise Deployment | Deployment |
| 部署 | Jakarta Management | JMX |
| 远程 | Jakarta RMI | RMI-IIOP |

# 5. 规范、实现与应用服务器的关系

## 5.1 一个规范对应多个实现

| 规范 | 参考实现（RI） | 其他主流实现 |
|------|---------------|--------------|
| Servlet | 内置于应用服务器 | Tomcat、Jetty、Undertow |
| JPA | EclipseLink | Hibernate、OpenJPA |
| JAX-RS | Jersey | RESTEasy、Apache CXF |
| CDI | Weld | OpenWebBeans |
| Bean Validation | Hibernate Validator | Apache BVal |
| JMS | 内置于应用服务器 | ActiveMQ Artemis、OpenMQ |
| JAX-WS | 内置于应用服务器 | Apache CXF、Metro |

## 5.2 WEB 服务器 vs 应用服务器

| 对比项 | WEB 服务器 | 应用服务器 |
|--------|-----------|-----------|
| 实现规范 | 仅 Servlet + JSP 等 Web 规范 | JavaEE/Jakarta EE 全部规范 |
| 典型产品 | Tomcat、Jetty | GlassFish、WildFly、WebLogic、WebSphere |
| 支持 EJB | 否 | 是 |
| 支持 JTA | 有限 | 完整 |
| 重量级 | 轻量、启动快 | 重量级、功能全 |

> 详细对比见 `2_application_servers.md`。

# 6. 本笔记覆盖范围

本目录按"完整覆盖 JavaEE/Jakarta EE 全部规范"组织，文件结构如下：

```
1_overview.md                   # 平台概述（本文件）
2_application_servers.md        # 应用服务器与部署环境
3_web_servlet_jsp.md            # Servlet/JSP/EL/JSTL（规范视角，衔接 JavaWeb 笔记）
4_faces_jsf.md                  # JSF
5_cdi_di.md                     # CDI + Jakarta 依赖注入
6_interceptors_managed_beans.md # 拦截器、托管Bean、公共注解
7_ejb.md                        # EJB
8_jndi.md                       # JNDI
9_jdbc_datasource.md            # JDBC + 连接池 + DataSource
10_jpa_persistence.md           # JPA（衔接 DataMiddleware/Hibernate 笔记）
11_bean_validation.md           # Bean Validation
12_jta_transactions.md          # JTA 事务
13_jms_messaging.md             # JMS 消息服务
14_javamail.md                  # JavaMail
15_xml_jaxp.md                  # JAXP（DOM/SAX/StAX）
16_jaxb.md                      # JAXB
17_json_p_b.md                  # JSON-P / JSON-B
18_jax_ws_soap.md               # JAX-WS（SOAP Web 服务）
19_jax_rs_rest.md               # JAX-RS（RESTful）
20_security.md                  # Jakarta Security
21_concurrency_batch.md         # Jakarta Concurrency + Batch
22_rmi_websocket.md             # RMI + WebSocket
23_packaging_deployment.md      # EAR/WAR 打包与部署
24_summary_spring_contrast.md   # 总结与 Spring 生态对照
```

## 6.1 与仓库其他目录的衔接

| 已有笔记目录 | 覆盖内容 | JavaEE 笔记的定位 |
|-------------|----------|------------------|
| `JavaCore/JavaWeb` | Servlet、JSP、EL、JSTL、Filter、Listener、Tomcat 实操 | 从规范/版本演进视角总览，不重复细节 |
| `DataMiddleware/Hibernate` | JPA 实现（Hibernate） | 从规范角度讲 JPA 标准 |
| `Messaging/` | Kafka、RabbitMQ、RocketMQ | 讲 JMS 标准，并与具体 MQ 对比 |
| `Spring/SpringFramework` | Spring IOC/AOP | 对照 Spring 如何取代部分 JavaEE 规范 |

# 7. 学习路线建议

```
第一步：掌握 Web 层规范（Servlet/JSP/EL/JSTL）→ 已有 JavaWeb 笔记
第二步：掌握组件模型（CDI/EJB/拦截器）
第三步：掌握数据与事务（JDBC/JPA/JTA）
第四步：掌握集成能力（JNDI/JMS/JavaMail/JCA）
第五步：掌握 Web 服务与数据格式（JAX-WS/JAX-RS/JAXB/JSON）
第六步：掌握安全、并发、批处理与部署（Security/Concurrency/Batch/EAR）
```

- 学习过程中不要陷入"每个规范都要精通"，企业实践中多数规范已被 Spring 生态替代。
- 面试与工作中真正高频的仍是 Servlet、JPA、JTA、CDI、JAX-RS 这五类。
