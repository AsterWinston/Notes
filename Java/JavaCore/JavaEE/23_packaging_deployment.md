
# 1. 概述

JavaEE 应用的打包与部署遵循标准化的归档结构。理解归档结构是部署到应用服务器的基础。

```
三种部署单元：
  JAR   ──► EJB 模块 / 类库
  WAR   ──► Web 应用
  EAR   ──► 企业应用（组合多个模块）
```

# 2. 部署单元对比

| 归档 | 全称 | 内容 | 部署描述符 | 部署位置 |
|------|------|------|-----------|----------|
| JAR | Java Archive | class、资源、EJB | `META-INF/ejb-jar.xml`（可选） | 应用服务器模块 |
| WAR | Web Archive | Web 应用（Servlet/JSP/HTML） | `WEB-INF/web.xml`（可选） | Web 应用 |
| EAR | Enterprise Archive | 多个 WAR/JAR 模块 | `META-INF/application.xml` | 企业应用 |

# 3. WAR 结构

```
myapp.war
├── index.html
├── css/ js/ image/
├── WEB-INF/
│   ├── web.xml                  # 部署描述符（Servlet 3.0+ 可省略）
│   ├── classes/                 # 编译后的 class
│   │   └── com/example/...
│   ├── lib/                     # 依赖 JAR
│   │   └── mysql-connector-j.jar
│   └── tags/                    # 自定义标签（可选）
```

## 3.1 web.xml 核心元素

```xml
<web-app xmlns="https://jakarta.ee/xml/ns/jakartaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="https://jakarta.ee/xml/ns/jakartaee
                             https://jakarta.ee/xml/ns/jakartaee/web-app_6_0.xsd"
         version="6.0">

    <!-- Servlet 注册 -->
    <servlet>
        <servlet-name>hello</servlet-name>
        <servlet-class>com.example.HelloServlet</servlet-class>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>hello</servlet-name>
        <url-pattern>/hello</url-pattern>
    </servlet-mapping>

    <!-- 欢迎页 -->
    <welcome-file-list>
        <welcome-file>index.html</welcome-file>
    </welcome-file-list>

    <!-- 会话超时（分钟） -->
    <session-config>
        <session-timeout>30</session-timeout>
    </session-config>
</web-app>
```

- Servlet 3.0+ 用注解（`@WebServlet`）后 web.xml 可省略，但**资源引用、安全配置、欢迎页**仍需描述符。
- 注解优先级低于描述符（描述符可覆盖）。

# 4. JAR（EJB 模块）结构

```
myejb.jar
├── com/example/bean/OrderServiceBean.class
├── META-INF/
│   ├── ejb-jar.xml            # 可选（注解方式可省略）
│   └── MANIFEST.MF
```

# 5. EAR 结构

```
myapp.ear
├── META-INF/
│   ├── application.xml        # 企业应用部署描述符
│   └── MANIFEST.MF
├── myapp-web.war              # Web 模块
├── myapp-ejb.jar              # EJB 模块
├── myapp-client.jar           # 客户端模块（可选）
└── lib/                       # 公共库（JavaEE 6+ 支持）
    └── commons-*.jar
```

## 5.1 application.xml

```xml
<application xmlns="https://jakarta.ee/xml/ns/jakartaee"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="https://jakarta.ee/xml/ns/jakartaee
                                 https://jakarta.ee/xml/ns/jakartaee/application_9_0.xsd"
             version="9.0">

    <display-name>MyApp</display-name>

    <module>
        <web>
            <web-uri>myapp-web.war</web-uri>
            <context-root>/myapp</context-root>
        </web>
    </module>

    <module>
        <ejb>myapp-ejb.jar</ejb>
    </module>

    <security-role>
        <role-name>admin</role-name>
    </security-role>
</application>
```

## 5.2 EAR 的优势

- 模块间通过 JNDI 相互引用（`java:app/` 命名空间）。
- 统一安全管理、统一部署、统一生命周期。
- Web 模块可调用同一 EAR 内的 EJB 模块。

# 6. 打包工具

## 6.1 Maven 打包

```xml
<!-- WAR -->
<project>
    <packaging>war</packaging>
</project>
```

```bash
mvn clean package
```

## 6.2 三种打包方式选择

| 目标 | packaging |
|------|-----------|
| Web 应用 | `war` |
| EJB 模块 | `ejb`（或 `jar`） |
| 企业应用 | `ear` |

# 7. 部署流程

## 7.1 应用服务器部署方式

| 方式 | 说明 |
|------|------|
| 管理控制台 | GlassFish `localhost:4848`、WildFly 管理控制台 |
| 命令行 | `asadmin deploy`、`jboss-cli deploy` |
| 拷贝目录 | Tomcat `webapps/`、WildFly `standalone/deployments/` |
| 自动部署 | 监听部署目录，出现新归档自动部署 |

```bash
# GlassFish
asadmin deploy myapp.war
asadmin undeploy myapp

# WildFly
./jboss-cli.sh --connect
deploy /path/to/myapp.war
undeploy myapp.war
```

## 7.2 部署热更新

- 部分服务器支持热部署（替换 WAR）。
- 生产环境建议滚动发布，避免热更新不稳定。

# 8. 环境隔离与配置

| 机制 | 说明 |
|------|------|
| 部署描述符覆盖 | 用描述符覆盖注解默认值，适应不同环境 |
| JNDI 资源 | 数据源等由服务器配置，应用按名引用 |
| 配置文件 | `META-INF/conf` 或外部化配置 |
| 环境条目 | `<env-entry>` 定义参数化配置 |

# 9. 常见部署问题排查清单

| 问题 | 排查方向 |
|------|----------|
| 部署失败 | 查看服务器日志（`server.log`）中的部署异常 |
| `ClassNotFoundException` | 检查 `WEB-INF/lib`、EAR `lib` 是否包含依赖 |
| 依赖冲突 | 检查不同模块的 jar 版本是否冲突 |
| 上下文路径不对 | 检查 `application.xml` 的 `context-root` |
| JNDI 找不到 | 确认资源已注册、名称正确 |
| 内存溢出 | 检查服务器 JVM 参数（`-Xmx`） |
| 热部署不生效 | 清理服务器 work/tmp 缓存，重启 |

# 10. 现代部署方式对比

```
传统：EAR/WAR → 应用服务器部署目录 → 服务器托管

现代：可执行 JAR（Spring Boot）→ 内嵌容器 → 单进程
      WAR → Tomcat（旧式）
      容器化 → Docker 镜像 → K8s 编排
```

| 维度 | 传统 JavaEE | Spring Boot 可执行 JAR |
|------|-------------|------------------------|
| 部署单元 | WAR/EAR | 可执行 JAR |
| 服务器 | 独立应用服务器 | 内嵌容器 |
| 启动 | 部署到服务器 | `java -jar app.jar` |
| 扩展性 | 集群/负载均衡 | 容器编排（K8s） |
| 配置 | JNDI/服务器配置 | application.yml |
