# 1.回顾：Servlet阶段已学的Tomcat知识

```
在学Servlet的时候，你已经接触过Tomcat了。我们先快速复习一下你已经知道的东西，避免重复：

1. Tomcat是什么？
   - Apache基金会的开源Web容器 / 轻量级应用服务器
   - 实现了Servlet规范、JSP规范、EL规范、WebSocket规范
   - 昵称"catalina"（Tomcat的核心Servlet容器模块就叫Catalina）

2. Tomcat的下载和安装
   - 官网：tomcat.apache.org
   - 绿色版（zip/tar.gz），解压即用，不需要安装
   - 解压后的目录结构：
     ├── bin/      存放启动/停止脚本（startup.bat、shutdown.bat、catalina.bat）
     ├── conf/     配置文件（server.xml、web.xml、tomcat-users.xml）
     ├── lib/      Tomcat运行时需要的jar包（servlet-api.jar、jsp-api.jar等）
     ├── logs/     日志文件（catalina.out、localhost.log等）
     ├── temp/     临时文件
     ├── webapps/  部署web应用的目录（你的oa、crm项目都放这里）
     └── work/     JSP编译后的.java和.class文件，以及Session持久化文件

3. 启动Tomcat需要配置的环境变量
   - JAVA_HOME：必须指向JDK根目录（不是JRE，因为JSP编译需要javac）
   - CATALINA_HOME：Tomcat根目录（可以自动检测，建议配）
   - PATH：不需要配置，直接在bin目录下运行脚本即可

4. 启动与验证
   - 双击 startup.bat 或者在命令行下运行 catalina.bat run
   - 打开浏览器输入 http://localhost:8080，看到Tomcat小猫页面就说明成功了

5. 在IDEA中集成Tomcat
   - Settings → Application Servers → 添加Tomcat安装目录
   - Run Configuration → Tomcat Server → Local → 配置端口和部署war包

6. 你已经理解的Tomcat角色
   - Tomcat是"WEB Server"和"Servlet容器"的合体
   - 浏览器发来的HTTP请求由Tomcat接收
   - Tomcat根据web.xml或注解找到对应的Servlet
   - Tomcat通过反射创建Servlet对象，调用service方法
   - 你的Servlet只管写业务逻辑，HTTP解析和响应包装由Tomcat完成

这些是你已经掌握的。接下来我们从这些基础知识出发，由浅入深地把Tomcat彻底讲透。
```

# 2.Tomcat是什么？为什么需要它？

```
要真正理解Tomcat，首先要回答一个根本问题：为什么浏览器不能直接调用你的Java类？

一、HTTP协议的本质

浏览器和服务器之间传输的是HTTP报文，它是纯文本格式：
```

```http
GET /oa/user/list HTTP/1.1
Host: localhost:8080
User-Agent: Mozilla/5.0
Accept: text/html

```

```
服务器返回的也是纯文本：
```

```http
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 1234

<html>...</html>
```

```
二、直接调用Java类的障碍

浏览器只能发HTTP文本，不能直接调用JVM中的方法。如果每写一个web应用都要自己实现：
  - TCP连接管理
  - HTTP协议解析（请求行、请求头、请求体）
  - 多线程并发处理
  - 请求路由（哪个URL对应哪个处理类）
  - 响应封装（设置状态码、响应头）
  - Session/Cookie管理
  - 安全控制
  - ...

那你一个简单的CRUD项目还没开始写业务代码，就要先写几千行网络编程代码。
这就是"为什么需要Tomcat"的答案——它帮你把所有底层通信的脏活累活都干了。

三、Servlet容器的核心职责

Tomcat作为Servlet容器，它的核心职责是：

┌──────────────────────────────────────────────────────┐
│                   Servlet容器                         │
│                                                      │
│  1. 通信支持    —— 把HTTP请求解析成HttpServletRequest  │
│  2. 生命周期    —— 创建Servlet实例、调用init/service/  │
│                   destroy                             │
│  3. 多线程支持  —— 每个请求分配线程，Servlet可并发处理  │
│  4. 会话管理    —— Session创建、维护、销毁             │
│  5. 安全管理    —— 认证、授权、SSL                     │
│  6. JSP支持     —— 把.jsp文件翻译成Servlet并编译       │
│  7. 静态资源    —— 直接返回HTML/CSS/JS/图片等          │
│  8. 部署管理    —— war包自动解压、热部署               │
└──────────────────────────────────────────────────────┘

四、Tomcat vs 其他容器

┌──────────┬──────────────┬──────────────────────────┐
│  容器     │  定位         │  特点                     │
├──────────┼──────────────┼──────────────────────────┤
│ Tomcat   │ 轻量级        │ 最流行，Spring Boot默认    │
│          │ Servlet容器   │ Servlet/JSP/EL/WS        │
├──────────┼──────────────┼──────────────────────────┤
│ Jetty    │ 轻量级        │ 可嵌入性好，Eclipse出品    │
│          │ Servlet容器   │ 常用于分布式中间件         │
├──────────┼──────────────┼──────────────────────────┤
│ Undertow │ 高性能        │ Red Hat出品，非阻塞IO     │
│          │ Web服务器     │ Spring Boot可选替代        │
├──────────┼──────────────┼──────────────────────────┤
│ WebLogic │ 重型JavaEE    │ Oracle商业产品             │
│          │ 应用服务器    │ 支持完整JavaEE规范(EJB等)  │
├──────────┼──────────────┼──────────────────────────┤
│ WebSphere│ 重型JavaEE    │ IBM商业产品               │
│          │ 应用服务器    │ 金融/大型企业              │
└──────────┴──────────────┴──────────────────────────┘

注意：Tomcat只是一个"Servlet容器"+"HTTP服务器"，不是完整的"JavaEE应用服务器"。
它不支持EJB、JMS、JTA等重量级规范。如果你需要这些，用WebLogic/WebSphere/JBoss(WildFly)。
但对于绝大多数JavaWeb项目来说，Servlet+JSP+JDBC就足够了，Tomcat完全胜任。
```

# 3.Tomcat的由来与版本演进

```
一、历史渊源

Tomcat的故事要从Apache基金会的James Duncan Davidson说起：

1995年 ─ Sun发布Java语言
1996年 ─ Sun发布Servlet API（0.9版），Java开始支持动态Web
1998年 ─ Sun推出JSP规范
1999年 ─ James Davidson为了验证Servlet/JSP规范，写了一款参考实现
         命名为"Tomcat"（因为他养了一只叫Tom的猫）
         Sun把这个参考实现捐献给了Apache基金会
         → Apache Tomcat 3.0 正式发布（前身是Apache JServ）

2001年 ─ Tomcat 4.0：全新架构"Catalina"替换了原来的容器核心
2002年 ─ Tomcat 5.0：支持Servlet 2.4 / JSP 2.0
2006年 ─ Tomcat 6.0：支持Servlet 2.5 / JSP 2.1
2010年 ─ Tomcat 7.0：支持Servlet 3.0（注解、异步处理、文件上传Part）
2014年 ─ Tomcat 8.0：支持Servlet 3.1（NIO Connector成熟）
2017年 ─ Tomcat 9.0：支持Servlet 4.0（HTTP/2、Server Push）
2021年 ─ Tomcat 10.0：支持Jakarta Servlet 5.0（包名从javax改为jakarta）
2022年 ─ Tomcat 10.1：支持Jakarta Servlet 6.0

二、版本与Servlet规范对照表（重要！）

┌────────────┬──────────────────────┬────────────────────┬──────────────┐
│ Tomcat版本  │ Servlet规范版本       │ JSP规范版本         │ Java最低版本  │
├────────────┼──────────────────────┼────────────────────┼──────────────┤
│ 6.x        │ Servlet 2.5          │ JSP 2.1            │ Java 5       │
│ 7.x        │ Servlet 3.0          │ JSP 2.2            │ Java 6       │
│ 8.x        │ Servlet 3.1          │ JSP 2.3            │ Java 7       │
│ 9.x        │ Servlet 4.0          │ JSP 2.3            │ Java 8       │
│ 10.0.x     │ Jakarta Servlet 5.0  │ Jakarta JSP 3.0    │ Java 8+      │
│ 10.1.x     │ Jakarta Servlet 6.0  │ Jakarta JSP 3.1    │ Java 11      │
│ 11.x       │ Jakarta Servlet 6.1  │ Jakarta Pages 4.0  │ Java 17+     │
└────────────┴──────────────────────┴────────────────────┴──────────────┘

三、为什么要关注版本？

1. Servlet 3.0 是分水岭：
   - 之前只能用web.xml配置Servlet
   - 之后可以用@WebServlet注解（你学Servlet时直接用的注解就是3.0的功劳）
   - 文件上传不再需要Commons FileUpload，可以用request.getPart()
   - 支持异步处理（AsyncContext）

2. Tomcat 8.0 → 9.0：
   - 连接器默认从BIO变成NIO
   - 支持HTTP/2
   - 内存占用更低

3. Tomcat 9.0 → 10.0（最大的一次断裂）：
   - javax.servlet.* → jakarta.servlet.* 全部改名！
   - 所有import语句要改，所有Maven依赖坐标要改
   - 原因：Oracle把JavaEE捐给Eclipse基金会后，不允许继续使用javax命名空间
   - 这个改名的深远影响我们在后面专门有一节来讲

四、当前学习建议

如果你在2024年以后开始学习JavaWeb：
  - 首选Tomcat 10.1.x（Jakarta Servlet 6.0），跟Spring Boot 3.x保持一致
  - 如果公司老项目用的是Tomcat 9（javax.servlet），你也得会
  - 新手建议直接用Tomcat 10+
```

# 4.Tomcat整体架构概览

```
在深入每个细节之前，必须先在脑子里建立一个"顶层设计地图"。Tomcat的架构可以用一句话概括：

  一个Server包含多个Service，每个Service包含一个Container和多个Connector
  Container内部又分四层：Engine → Host → Context → Wrapper

一、顶层组件关系图

┌─────────────────────────────────────────────────────────┐
│                        Server（服务器）                   │
│  整个Tomcat实例，管理所有Service                          │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │                  Service（服务）                   │    │
│  │  将Connector和Container组合在一起                  │    │
│  │                                                   │    │
│  │  ┌──────────┐  ┌──────────┐  ┌───────────────┐  │    │
│  │  │Connector │  │Connector │  │   Container   │  │    │
│  │  │ (HTTP)   │  │ (AJP)    │  │   (Catalina)  │  │    │
│  │  │ 端口8080 │  │ 端口8009 │  │               │  │    │
│  │  └──────────┘  └──────────┘  │  ┌─────────┐  │  │    │
│  │                               │  │ Engine  │  │  │    │
│  │                               │  │ ┌─────┐ │  │  │    │
│  │                               │  │ │Host │ │  │  │    │
│  │                               │  │ │┌───┐│ │  │  │    │
│  │                               │  │ ││Ctx││ │  │  │    │
│  │                               │  │ ││┌─┐││ │  │  │    │
│  │                               │  │ │││W│││ │  │  │    │
│  │                               │  │ ││└─┘││ │  │  │    │
│  │                               │  │ │└───┘│ │  │  │    │
│  │                               │  │ └─────┘ │  │  │    │
│  │                               │  └─────────┘  │  │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘

二、三大子项目

Tomcat不是铁板一块，而是三个独立子项目的组合：

┌──────────────────────────────────────────────────┐
│  1. Coyote（郊狼）                                 │
│     职责：网络通信层（HTTP协议解析）                │
│     对应组件：Connector、ProtocolHandler、Endpoint │
│     你之前在Servlet阶段没怎么接触，因为它是Tomcat   │
│     内部的东西，对Servlet开发者透明                 │
├──────────────────────────────────────────────────┤
│  2. Catalina（卡特琳娜）                           │
│     职责：Servlet容器核心（管理Servlet生命周期）    │
│     对应组件：Engine、Host、Context、Wrapper       │
│     这是你写得最多的那块——你的Servlet就在Wrapper里  │
├──────────────────────────────────────────────────┤
│  3. Jasper（碧玉）                                 │
│     职责：JSP引擎（把.jsp翻译成.java再编译成.class）│
│     对应组件：JspServlet、JSP Compiler            │
│     你学的JSP、EL、JSTL最终都由Jasper处理          │
└──────────────────────────────────────────────────┘

三、Connector和Container的分工

这是理解Tomcat最核心的概念之一：

  Connector（连接器）：负责"外部通信"
    - 监听端口，接收TCP连接
    - 解析HTTP请求报文
    - 把HTTP请求包装成标准的ServletRequest对象
    - 把ServletResponse对象转换回HTTP响应报文

  Container（容器）：负责"内部处理"
    - 接收Connector传来的ServletRequest
    - 根据URL找到对应的Servlet
    - 执行Filter链
    - 调用Servlet.service()方法

你可以这样理解：
  Connector是"门卫+邮差"：收信、拆信、分类
  Container是"办公室"：处理信件内容、写回信

四、为什么这样设计？

这种"连接器-容器"分离的设计带来巨大的灵活性：
  - 可以给一个Container配多个Connector（比如同时开HTTP 8080和AJP 8009端口）
  - Connector的实现可以随意切换（BIO/NIO/APR），不影响Container
  - 理论上可以更换Container实现（虽然实际中极少这么做）
  - Coyote和Catalina是两个独立模块，各自演进互不干扰
```

# 5.Tomcat源码结构概览

```
阅读源码是深入理解Tomcat的最佳方式。这一节给你一个源码地图，让你知道从哪里入手。

一、获取源码

方式一：官网下载Source版
  https://tomcat.apache.org → Downloads → Source Distributions
  下载 apache-tomcat-10.1.x-src.zip，解压即可

方式二：GitHub（推荐）
  git clone https://github.com/apache/tomcat.git
  git checkout 10.1.x（选择你学习的版本分支）

方式三：Maven依赖源码（如果你只是想看某个类的实现）
  在IDEA中点击类名 → Download Sources → 自动下载源码jar

二、源码模块结构

tomcat/
├── bin/              启动脚本源码（.bat/.sh模板）
├── conf/             默认配置文件
├── java/             Java源码根目录
│   ├── javax/        兼容层（Tomcat内部用，非标准API）
│   └── org/apache/
│       ├── catalina/         *** Catalina（核心容器）
│       ├── coyote/           *** Coyote（连接器）
│       ├── jasper/           **  Jasper（JSP引擎）
│       ├── el/               *   EL表达式引擎
│       ├── juli/             *   日志系统
│       ├── naming/           ☆   JNDI命名服务
│       ├── tomcat/util/      *   通用工具类
│       └── tomcat/util/net/  **  NIO/SSL网络工具
├── modules/          各模块独立构建配置
├── res/              资源文件（如默认的web.xml）
├── test/             单元测试
└── webapps/          默认web应用（ROOT、manager、docs等）

三、关键包详解

1. org.apache.catalina 核心包
```

```java
// 最核心的接口和实现都在这里：
org.apache.catalina.Server       // Server接口
org.apache.catalina.Service      // Service接口
org.apache.catalina.Container    // Container接口（Engine/Host/Context/Wrapper的父接口）
org.apache.catalina.Lifecycle    // 生命周期接口（后面会重点讲）
org.apache.catalina.Valve        // 阀门接口（Pipeline机制）
org.apache.catalina.Pipeline     // 管道接口

// 标准实现（真正的代码逻辑在这里）：
org.apache.catalina.core.StandardServer
org.apache.catalina.core.StandardService
org.apache.catalina.core.StandardEngine
org.apache.catalina.core.StandardHost
org.apache.catalina.core.StandardContext    // *最复杂的类（一个webapp的全生命周期）
org.apache.catalina.core.StandardWrapper    // *封装一个Servlet

// 启动入口：
org.apache.catalina.startup.Bootstrap     // main方法在这里
org.apache.catalina.startup.Catalina      // 解析server.xml、管理生命周期
```

```java
// 2. org.apache.coyote 连接器包
org.apache.coyote.Adapter              // 适配器接口（CoyoteAdapter连接Coyote和Catalina）
org.apache.coyote.http11.Http11NioProtocol  // HTTP/1.1 + NIO协议处理器
org.apache.tomcat.util.net.NioEndpoint      // NIO端点（Acceptor + Poller + Worker）
org.apache.tomcat.util.net.SocketWrapperBase // Socket包装

// 3. org.apache.tomcat.util 工具包
org.apache.tomcat.util.threads.ThreadPoolExecutor  // 线程池
org.apache.tomcat.util.digester.Digester           // XML解析器（解析server.xml）
org.apache.tomcat.util.scan.StandardJarScanner     // Jar扫描（Servlet 3.0+的注解扫描）
```

```
四、推荐阅读源码的顺序（由浅入深）

第一步：Lifecycle → LifecycleBase（理解生命周期）
第二步：Server → Service → Connector → Container 接口定义
第三步：Bootstrap → Catalina（理解启动流程）
第四步：StandardServer → StandardService（理解实现）
第五步：NioEndpoint → Acceptor → Poller（理解网络层）
第六步：CoyoteAdapter（理解连接器→容器的桥接点）
第七步：StandardEngine → StandardHost → StandardContext → StandardWrapper
第八步：Pipeline → Valve → StandardWrapperValve（理解请求处理链）

看不懂没关系，接下来的每一节我们都会逐步剖析这些源码。你不需要一开始就读完。
```

# 6.server.xml核心配置详解（上）

```
server.xml 是 Tomcat 最重要的配置文件，位于 $CATALINA_BASE/conf/server.xml。
Tomcat 启动时做的第一件大事就是解析这个文件（用Digester框架），然后根据配置创建对应的Java对象。

一、server.xml 的全貌骨架

一个完整的 server.xml 结构如下：
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!--
  port="8005" 是关闭Tomcat的监听端口
  shutdown="SHUTDOWN" 是关闭指令（发这个字符串到8005端口，Tomcat就会关闭）
-->
<Server port="8005" shutdown="SHUTDOWN">

  <!-- 全局监听器：监听Server组件的生命周期事件 -->
  <Listener className="org.apache.catalina.startup.VersionLoggerListener" />
  <Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="on" />
  <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" />
  <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />
  <Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />

  <!-- 全局JNDI资源（数据源等） -->
  <GlobalNamingResources>
    <Resource name="UserDatabase"
              auth="Container"
              type="org.apache.catalina.UserDatabase"
              description="User database that can be updated and saved"
              factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
              pathname="conf/tomcat-users.xml" />
  </GlobalNamingResources>

  <!-- Service：一个Connector+Engine的组合 -->
  <Service name="Catalina">

    <!-- 下面会详细展开 -->

  </Service>

</Server>
```

```
二、Server 元素

<Server port="8005" shutdown="SHUTDOWN">

Server 代表整个 Tomcat 实例，一个 JVM 里只有一个 Server。
  - port：监听关闭命令的端口。收到 shutdown 指令后执行关闭流程。
    这是一个安全隐患：如果你暴露8005端口在外网，任何人发"SHUTDOWN"就能关掉你的Tomcat！
    生产环境通常把 port 改成 "-1"（禁用shutdown端口）
  - shutdown：关闭指令字符串，可以自定义（但没必要，禁用端口更直接）

三、Listener 元素

Listener 是 Tomcat 的"观察者"。Server 在生命周期的各个节点会触发事件，
Listener 收到事件后执行相应的操作。常用的 Listener：

  - VersionLoggerListener：启动时打印Tomcat版本和JVM信息
  - AprLifecycleListener：尝试加载APR本地库（如果安装了tomcat-native）
  - JreMemoryLeakPreventionListener：修复JRE内存泄漏bug（比如DriverManager注册）
  - GlobalResourcesLifecycleListener：将全局JNDI资源注册为MBean（JMX管理）
  - ThreadLocalLeakPreventionListener：重加载时清理ThreadLocal避免内存泄漏

四、GlobalNamingResources 元素

JNDI（Java Naming and Directory Interface）是Java的"全局字典"，
可以在这里配置全局的数据库连接池、邮件会话等资源，供所有webapp共享。

五、Service 元素

<Service name="Catalina">

Service 是一个"逻辑组合"——将一个 Container（处理请求）和若干个 Connector（接收请求）
捆绑在一起。一个 Server 可以包含多个 Service，每个 Service 监听不同的端口组。

通常情况下，你只有一个 Service，名字叫 "Catalina"。
```

# 7.server.xml核心配置详解（下）

```
六、Connector 元素（最重要！）

这是 server.xml 中你最常修改的配置。Connector 定义了一个对外监听端口。

一个典型的 HTTP Connector 配置：
```

```xml
<Connector port="8080"
           protocol="HTTP/1.1"
           connectionTimeout="20000"
           maxThreads="200"
           minSpareThreads="10"
           acceptCount="100"
           maxConnections="10000"
           redirectPort="8443"
           URIEncoding="UTF-8"
           compression="on"
           compressionMinSize="2048"
           noCompressionUserAgents="gozilla, traviata"
           compressibleMimeType="text/html,text/xml,text/plain,text/css,text/javascript,application/json,application/javascript"
           useBodyEncodingForURI="true" />
```

```
各参数详解：

┌──────────────────────┬────────────────────────────────────────────┐
│ 参数                  │ 说明                                        │
├──────────────────────┼────────────────────────────────────────────┤
│ port                  │ 监听端口（默认8080）                         │
│ protocol              │ 协议类型：                                  │
│                      │   "HTTP/1.1" → 自动选择（Tomcat 8+选NIO)    │
│                      │   也可以直接指定类名                          │
│                      │   如 org.apache.coyote.http11.Http11NioProtocol│
│ connectionTimeout     │ 连接超时时间（毫秒）                         │
│                      │   如果客户端在此时间内没发数据，断开连接       │
│ maxThreads            │ 最大工作线程数（默认200）                    │
│                      │   处理用户请求的线程                          │
│ minSpareThreads       │ 最小空闲线程数（默认10）                     │
│                      │   即使没请求也保持这些线程存活（避免频繁创建） │
│ acceptCount           │ 等待队列长度（默认100）                      │
│                      │   所有线程忙时，新连接进入队列等待              │
│ maxConnections        │ 最大连接数（NIO默认10000，BIO和maxThreads同）│
│                      │   超过此值后新的连接会被拒绝                   │
│ redirectPort          │ 重定向端口（SSL默认8443）                    │
│                      │   如果请求需要SSL，Tomcat会自动重定向到此端口   │
│ URIEncoding           │ URI的字符编码（强烈建议设为UTF-8！）          │
│ compression           │ 响应压缩开关（on/off/force）                 │
│ compressionMinSize    │ 压缩的最小响应体大小（字节）                  │
│ compressibleMimeType  │ 哪些MIME类型需要压缩                         │
│ useBodyEncodingForURI │ 是否用请求体的编码来解码URI（解决GET参数乱码） │
│ maxKeepAliveRequests  │ 单个长连接最大请求数（默认100，-1表示无限制） │
│ keepAliveTimeout      │ 长连接超时时间（毫秒）                        │
│ maxPostSize           │ POST请求体最大大小（字节），0表示无限制       │
│ maxParameterCount     │ 最大参数个数（默认10000）                    │
│ maxSwallowSize        │ 请求体上传失败时最多"吞掉"多少字节            │
└──────────────────────┴────────────────────────────────────────────┘

七、AJP Connector（通常被注释掉）

AJP（Apache JServ Protocol）是Tomcat和Apache HTTPD之间的二进制协议。
以前用 Apache HTTPD 做反向代理时，通过 AJP 协议把请求转发给 Tomcat。
现在基本都用 Nginx + HTTP 代理了，AJP 已经很少用。从 Tomcat 9 开始默认注释掉了。

<Connector protocol="AJP/1.3"
           port="8009"
           redirectPort="8443"
           secretRequired="false" />

八、Engine 元素

<Engine name="Catalina" defaultHost="localhost">
  - name：引擎名字（日志中会打印）
  - defaultHost：默认虚拟主机名。如果请求头中的Host不匹配任何Host，就路由到这个默认Host

九、Host 元素（虚拟主机）

<Host name="localhost"
      appBase="webapps"
      unpackWARs="true"
      autoDeploy="true">

  - name：虚拟主机的域名（localhost表示只能通过localhost访问）
  - appBase：存放web应用的目录（默认webapps）
    可以是绝对路径，也可以是相对于$CATALINA_BASE的相对路径
  - unpackWARs：是否自动解压war包（true = 解压后运行，false = 直接运行war）
    解压后运行的好处：读取静态资源更快，可以修改其中的文件热加载
  - autoDeploy：是否自动部署（true = 检测到新war包/新目录自动部署）
  - startStopThreads：启动/停止Context时的线程数

Host 内部可以配置多个 Context（本节最后讲），还可以配置 Valve、Realm 等。

十、Valve 在 server.xml 中的配置

Valve（阀门）是Tomcat中的"拦截器"，可以插入到Engine/Host/Context的管道中。

常用Valve：
```

```xml
<!-- 访问日志阀门：记录所有HTTP请求 -->
<Valve className="org.apache.catalina.valves.AccessLogValve"
       directory="logs"
       prefix="localhost_access_log"
       suffix=".txt"
       pattern="%h %l %u %t %r %s %b" />

<!-- 远程地址过滤器：只允许特定IP访问 -->
<Valve className="org.apache.catalina.valves.RemoteAddrValve"
       allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1" />

<!-- 错误报告阀门：自定义错误页面 -->
<Valve className="org.apache.catalina.valves.ErrorReportValve"
       showReport="false"
       showServerInfo="false" />
```

```
十一、一个完整的 server.xml 骨架总结

<Server>
  <Listener ... />                      # 各种监听器
  <GlobalNamingResources> ... </>        # 全局资源

  <Service name="Catalina">

    <Connector port="8080" ... />        # HTTP连接器
    <!-- <Connector port="8009" ... /> --> # AJP连接器（通常注释）

    <Engine name="Catalina" defaultHost="localhost">
      <Host name="localhost" appBase="webapps" autoDeploy="true">
        <!-- Context 通常由自动部署产生，不手写在server.xml中 -->
        <!-- <Valve ... /> -->
      </Host>
    </Engine>

  </Service>
</Server>
```

# 8.Connector连接器深度解析（上）—— 架构与协议

```
前面我们讲了Connector是"门卫+邮差"，现在来看它内部到底长什么样。

一、Connector的三层组件

Connector不是一块铁板，它内部有三个核心组件分工合作：

┌──────────────────────────────────────────────────────────┐
│                     Connector                            │
│                                                          │
│  ┌──────────────────┐                                    │
│  │ ProtocolHandler  │ -- 协议处理器（核心）               │
│  │                  │                                    │
│  │  ┌────────────┐  │                                    │
│  │  │  Endpoint  │  │ -- 网络端点（Socket通信、线程池）    │
│  │  │  (网络层)  │  │                                    │
│  │  └────────────┘  │                                    │
│  │  ┌────────────┐  │                                    │
│  │  │ Processor  │  │ -- 协议解析器（解析HTTP报文）       │
│  │  │  (应用层)  │  │                                    │
│  │  └────────────┘  │                                    │
│  └──────────────────┘                                    │
│           │                                              │
│           v                                              │
│  ┌──────────────────┐                                    │
│  │    Adapter       │ -- 适配器（Coyote -- Catalina桥梁） │
│  └──────────────────┘                                    │
│           │                                              │
│           v                                              │
│  ┌──────────────────┐                                    │
│  │    Mapper        │ -- 路由映射器（URL -- Host/Context/ │
│  │                  │    Wrapper）                       │
│  └──────────────────┘                                    │
└──────────────────────────────────────────────────────────┘

二、ProtocolHandler：协议处理器

ProtocolHandler是"老板"，它把Endpoint和Processor组合在一起。
不同的协议有不同实现：

  Http11NioProtocol    -- NioEndpoint + Http11Processor
  Http11Nio2Protocol   -- Nio2Endpoint + Http11Processor
  Http11AprProtocol    -- AprEndpoint + Http11AprProcessor

你可以在server.xml中显式指定：
```

```xml
<!-- 方式一：写简写（推荐） -->
<Connector protocol="HTTP/1.1" />

<!-- 方式二：写完整类名 -->
<Connector protocol="org.apache.coyote.http11.Http11NioProtocol" />
```

```
三、Endpoint：网络端点

Endpoint 负责最底层的网络I/O操作。它的内部结构：

┌──────────────────────────────────────────────┐
│               NioEndpoint                    │
│                                              │
│  ┌──────────┐  ┌─────────┐  ┌────────────┐  │
│  │ Acceptor │  │ Poller  │  │  Worker    │  │
│  │ (接收器) │  │ (轮询器)│  │  (工作线程) │  │
│  └──────────┘  └─────────┘  └────────────┘  │
│       │             │              │         │
│   接收新连接    检测IO就绪    执行业务处理   │
│   (阻塞)       (非阻塞)       (多线程池)    │
└──────────────────────────────────────────────┘

Acceptor（接收器）：
  - 一个独立线程，死循环调用 ServerSocket.accept()
  - 接收新的TCP连接，设置为非阻塞模式
  - 把连接注册到Poller的事件队列中
  - 如果达到 maxConnections 上限，暂停接收（LimitLatch控制）

Poller（轮询器）：
  - 通常有1-2个线程（可配：pollerThreadCount，默认 Math.min(2, Runtime.getRuntime().availableProcessors())）
  - 使用 Java NIO 的 Selector 监听已注册的 SocketChannel
  - 检测到某个Socket有数据可读，就用SocketProcessor包装后提交给Worker线程池
  - Selector.select() 的阻塞时间很短（约1秒），确保不会死等

SocketProcessor：
  - 封装了一个Socket连接的处理任务
  - 由Worker线程池执行

四、Processor：协议解析器

Processor负责把字节流解析成HTTP请求对象：

流程：SocketChannel 上的字节
  → Http11InputBuffer 读入缓冲区
  → 按照RFC 7230解析：
      1. 解析请求行（GET /path HTTP/1.1）
      2. 解析请求头（Host: xxx, Content-Type: xxx...）
      3. 根据 Content-Length / Transfer-Encoding 解析请求体
  → 构建内部的 Request 和 Response 对象
  → 调用 Adapter.service(request, response)
```

```
五、LimitLatch：连接数限制

LimitLatch 是 Endpoint 内部用来限制最大连接数的组件。
它基于AQS（AbstractQueuedSynchronizer）实现，

工作原理：
  - 初始化时设置一个计数器 = maxConnections
  - 每接收一个连接，计数器减1
  - 如果计数器为0，Acceptor线程阻塞等待
  - 每个连接关闭时，计数器加1，唤醒等待的Acceptor

这保证了Tomcat不会被海量连接撑爆。
```

# 9.Connector连接器深度解析（下）—— 内部组件

```
六、Adapter：CoyoteAdapter 桥接器

这是整个Tomcat架构中最关键的"转接头"。它在 Coyote 世界和 Catalina 世界之间
做翻译工作：

CoyoteAdapter.service(request, response) 做了这些事情：
```

```java
// 简化的 CoyoteAdapter.service() 流程
public void service(org.apache.coyote.Request req,
                    org.apache.coyote.Response res) throws Exception {

    // 1. 检查请求头中是否有 "Connection: close"
    //    如果有，响应完成后关闭连接

    // 2. 把 coyote.Request 包装成 HttpServletRequest
    //    把 coyote.Response 包装成 HttpServletResponse
    //    Request 类是 org.apache.catalina.connector.Request
    //    （注意区别：coyote.Request 是内部对象，catalina.connector.Request 是你Servlet中用的）

    // 3. 调用 postParseRequest() 方法，这个方法做了：
    //    - 解析URL中的 scheme://host:port/contextPath/servletPath/pathInfo
    //    - 设置请求的字符编码
    //    - 处理 Session ID（从Cookie或URL中提取）
    //    - 调用 Mapper.map() 进行路由

    // 4. 调用 Container 的 Pipeline 的第一个 Valve
    //    connector.getService().getContainer().getPipeline().getFirst().invoke(request, response);

    // 5. 响应结束后：
    //    - 如果是异步处理，等待异步完成
    //    - 如果是 keep-alive，不关闭连接，等待下一个请求
    //    - 否则关闭Socket连接
}
```

```
这就是为什么你在Servlet中写的是 HttpServletRequest 而不是 Coyote的Request——
CoyoteAdapter 帮你做了转换。

七、请求对象的创建和回收

Tomcat 为了性能，不会每次都 new Request/Response 对象，而是使用"对象池"：

1. 每个 Processor 内部有一个 Request + Response 对象的引用
2. 处理完一个请求后，调用 recycle() 方法重置对象状态（清空所有属性）
3. 下一次请求复用同一个对象（省去了 GC 的开销）

这就是为什么从Tomcat内部看，Request/Response对象在你Servlet执行完后立刻就被回收了——
你在异步处理中（AsyncContext）需要自己保存它们的引用，否则数据就丢了。

八、keep-alive 处理的细节

HTTP/1.1 默认开启 keep-alive（长连接）。处理流程：

1. Processor 处理完一个请求后，检查响应头中 Connection 的值
2. 如果是 "Connection: keep-alive"（1.1默认），Processor 不关闭Socket
3. Processor 返回 Poller，Poller 把这个连接重新注册到 Selector 上
4. 下一个请求到来时，Poller 再次检测到可读，再次交给 Processor 处理
5. 如果达到 maxKeepAliveRequests（默认100）或者超过 keepAliveTimeout（默认和connectionTimeout同），
   关闭连接

九、一个完整的Connector处理流程总结

客户端发起连接
  │
  v
Acceptor 接收连接，注册到Poller
  │
  v
Poller 检测到有数据可读 → 创建 SocketProcessor → 提交到 Worker 线程池
  │
  v
Worker 线程执行 SocketProcessor
  │
  v
Http11Processor.process()
  ├─ InputBuffer 读数据
  ├─ 解析HTTP报文
  ├─ 构建 coyote.Request / coyote.Response
  │
  v
CoyoteAdapter.service(coyoteReq, coyoteRes)
  ├─ 包装为 HttpServletRequest / HttpServletResponse
  ├─ postParseRequest() + Mapper 路由
  └─ Engine Pipeline 的第一个 Valve.invoke()
      │
      v
      ... 经过各级 Valve ...
      │
      v
      Servlet.service(request, response)
      │
      v
      返回响应 → Processor 写响应到 Socket
      │
      v
      根据 keep-alive 决定：继续等下一个请求 / 关闭连接
```

# 10.I/O模型对比：BIO / NIO / NIO2 / APR

```
这是一个面试高频问题，也是性能调优的基础。
把Tomcat支持的四中I/O模型彻底搞清楚。

一、BIO（Blocking IO，阻塞IO）

原理：
  - 每个连接分配一个线程，read()操作阻塞直到数据到达
  - 线程在read()没等到数据时什么都不能做，只能干等

Tomcat 中的实现：
  - org.apache.coyote.http11.Http11Protocol（Tomcat 7及之前默认）
  - 对应 Endpoint：JIoEndpoint

特点：
  - 优点：编程简单，数据处理逻辑清晰
  - 缺点：每个连接一个线程！1000个并发 = 1000个线程
    操作系统线程有上限（Linux 默认约 1000-2000个，Windows更少）
    大量线程切换消耗CPU，内存占用大（每个线程栈约1MB，1000线程 = 1GB内存）

适用场景：低并发（<200）、连接数固定

二、NIO（Non-Blocking IO，非阻塞IO）

原理：
  - 使用 Java NIO 的 Selector（多路复用器）
  - 少量线程（1-2个Poller）轮询大量连接（Selector.select()）
  - 只有真正有数据可读的连接才分配工作线程去处理

Tomcat 中的实现：
  - org.apache.coyote.http11.Http11NioProtocol（Tomcat 8+ 默认）
  - 对应 Endpoint：NioEndpoint

特点：
  - 优点：少量线程管理大量连接，线程切换开销小
    几千并发连接只需要几十个线程
  - 缺点：编程模型比BIO复杂；对于"连接数少但数据量大"的场景，优势不明显

核心原理图解：
```

```
  BIO 模型（一对一）：
   连接1 ---- 线程1（read阻塞）
   连接2 ---- 线程2（read阻塞）
   连接3 ---- 线程3（read阻塞）
   ...
   连接N ---- 线程N（read阻塞）

  NIO 模型（多路复用）：
   连接1 ----┐
   连接2 ----┤
   连接3 ----┼---- Selector（Poller轮询）---- 有数据的连接分发给Worker线程
   连接4 ----┤
   ...    │
   连接N ----┘
```

```
selector.select() 会返回所有有事件（可读/可写）的SelectionKey，
然后为每个有数据的连接创建一个任务，交给Worker线程池处理。

三、NIO2（AIO，异步IO）

原理：
  - 基于 Java 7 的 AsynchronousChannelGroup
  - 操作系统完成IO后回调通知应用程序（比NIO更彻底的非阻塞）
  - 完全由操作系统内核完成IO读写，CPU介入最少

Tomcat 中的实现：
  - org.apache.coyote.http11.Http11Nio2Protocol
  - 对应 Endpoint：Nio2Endpoint

特点：
  - 优点：理论上更高性能，操作系统做更多事
  - 缺点：Windows上AIO支持最好，Linux上实现不够成熟（底层用epoll模拟）
    实际性能对比NIO优势不明显，使用率低

适用场景：Windows服务器环境（很少见）、实验性使用

四、APR（Apache Portable Runtime，本机库）

原理：
  - 不依赖JVM的IO能力，直接调用操作系统底层C库
  - 需要安装 tomcat-native 包（.dll / .so 文件）
  - 底层使用 OpenSSL 做HTTPS（比JSSE性能好很多）

Tomcat 中的实现：
  - org.apache.coyote.http11.Http11AprProtocol
  - 对应 Endpoint：AprEndpoint

特点：
  - 优点：HTTPS场景下性能大幅领先（因为用OpenSSL替代JSSE）
    连接处理能力比NIO更强
  - 缺点：需要安装额外组件，部署复杂度增加
    windows下需安装重分发包

安装方式：
  - Linux: yum install tomcat-native  / apt install libtcnative-1
  - 或者直接在 $CATALINA_HOME/bin 下放 tcnative-1.dll / libtcnative-1.so

五、四种模型对比总结

┌──────────┬──────────┬──────────────┬──────────────┬──────────────┐
│ 模型      │ 线程数    │ 最大连接数    │ HTTPS性能     │ 推荐场景      │
├──────────┼──────────┼──────────────┼──────────────┼──────────────┤
│ BIO      │ =连接数  │ 受线程数限制  │ 一般          │ 已淘汰        │
│ NIO      │ 很少     │ 数万          │ 一般          │ * 默认首选   │
│ NIO2(AIO)│ 很少     │ 数万          │ 一般          │ 实验性        │
│ APR      │ 很少     │ 数万          │ ** 极佳      │ 高并发HTTPS  │
└──────────┴──────────┴──────────────┴──────────────┴──────────────┘

六、怎么配置切换？

在 server.xml 的 Connector 元素中修改 protocol 属性：
```

```xml
<!-- NIO（默认，通常不用显式写） -->
<Connector protocol="org.apache.coyote.http11.Http11NioProtocol" port="8080" />

<!-- NIO2 -->
<Connector protocol="org.apache.coyote.http11.Http11Nio2Protocol" port="8080" />

<!-- APR（需安装tomcat-native） -->
<Connector protocol="org.apache.coyote.http11.Http11AprProtocol" port="8080" />

<!-- 也可以写简写（但不推荐，不同版本默认不同） -->
<Connector protocol="HTTP/1.1" port="8080" />
```

```
七、Spring Boot 中的配置

Spring Boot 内嵌 Tomcat 默认使用 NIO。如果你要切换：
```

```yaml
# application.yml
server:
  tomcat:
    protocol-header: org.apache.coyote.http11.Http11Nio2Protocol  # 切换为NIO2
    max-threads: 200
    max-connections: 10000
    accept-count: 100
```

```
或通过编程方式：
```

```java
@Bean
public TomcatServletWebServerFactory tomcatFactory() {
    TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory();
    factory.setProtocol("org.apache.coyote.http11.Http11Nio2Protocol");
    return factory;
}
```

# 11.Catalina容器层次结构

```
前面我们把Connector讲透了，现在来看"集装箱"的另一半——Catalina容器。

一、四级容器模型

Catalina容器是一个嵌套结构，大容器套小容器：

┌─────────────────────────────────────────────────────────┐
│  Engine（引擎）                                          │
│  整个Servlet引擎，一个Service有一个Engine                 │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │  Host（虚拟主机）                                   │  │
│  │  对应一个域名（如 localhost、www.example.com）      │  │
│  │                                                    │  │
│  │  ┌──────────────────────────────────────┐          │  │
│  │  │  Context（上下文，即一个webapp）       │          │  │
│  │  │  对应你部署的 oa 项目、crm 项目等     │          │  │
│  │  │                                      │          │  │
│  │  │  ┌──────────────────────┐            │          │  │
│  │  │  │  Wrapper（包装器）    │            │          │  │
│  │  │  │  对应一个 Servlet    │            │          │  │
│  │  │  │  如 HelloServlet     │            │          │  │
│  │  │  └──────────────────────┘            │          │  │
│  │  └──────────────────────────────────────┘          │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘

一个请求来了以后，就是在这些容器之间"逐层深入"。

二、Engine：引擎

实现类：org.apache.catalina.core.StandardEngine

Engine 是整个Servlet处理的入口，一个Service有且只有一个Engine。
它的职责：
  - 作为容器的"根节点"
  - 根据请求的Host头选择对应的 Host 子容器
  - 如果找不到匹配的Host，使用 defaultHost 属性指定的默认Host

源码中：
  Engine 本身几乎不做什么事情，
  请求过来后，它把请求甩给 Pipeline 的第一个 Valve（StandardEngineValve）
  StandardEngineValve 做的事：根据request.getHost()选择对应的Host子容器，
  然后调用 host.getPipeline().getFirst().invoke(request, response)

三、Host：虚拟主机

实现类：org.apache.catalina.core.StandardHost

Host 代表一个"网站"。你可以在同一台机器上跑多个网站，每个绑定不同域名：
  www.a.com → appBase="/data/webapp_a"
  www.b.com → appBase="/data/webapp_b"
  都通过同一个Tomcat的同一个8080端口访问（通过HTTP头中的Host区分）

Host 的职责：
  - 管理该域名下的所有 Context（webapp）
  - 处理 ErrorPage（错误页面的 Valve）
  - 管理自动部署（autoDeploy、deployOnStartup）
  - 为每个 Context 分配 Session 管理器

四、Context：上下文（最核心！）

实现类：org.apache.catalina.core.StandardContext

Context 对应你部署的"一个web应用"（比如 oa 项目）。这是整个Tomcat源码中最复杂、
代码量最大的类之一（StandardContext.java 约 7000+行！）。

Context 的职责：
  - 管理 Servlet 实例（Wrapper 集合）
  - 管理 Filter 链（FilterDef + FilterMap）
  - 管理 Listener 集合
  - 管理欢迎文件（welcome-file-list）
  - 管理错误页面（error-page）
  - 加载和解析 web.xml
  - 扫描注解（@WebServlet、@WebFilter、@WebListener）
  - 管理 Session 管理器
  - 管理安全约束（security-constraint）
  - 管理资源文件（docBase目录下的静态文件）
  - 管理类加载器（WebappClassLoader）
  - 管理MBean注册（JMX）
  - 处理 reloadable（热加载：检测到类文件变化自动重新加载）

五、Wrapper：包装器

实现类：org.apache.catalina.core.StandardWrapper

Wrapper 是对一个 Servlet 的最内层包装。它负责：
  - 持有 Servlet 实例
  - 管理 Servlet 的生命周期（init → service → destroy）
  - 管理 load-on-startup（启动时是否提前加载）
  - 管理 Servlet 的初始化参数（init-param）
  - 调用 Filter 链
  - 单线程模型（SingleThreadModel，已废弃）
  - Servlet 安全角色（@ServletSecurity）

注意：Wrapper 本身不直接调用 Servlet。
它通过 StandardWrapperValve 来完成实际的 Servlet 调用。

六、容器接口设计

所有容器（Engine、Host、Context、Wrapper）都实现了 Container 接口：
```

```java
public interface Container extends Lifecycle {
    // 容器名
    String getName();

    // 父容器、子容器
    Container getParent();
    void addChild(Container child);
    Container[] findChildren();

    // 管道和阀门
    Pipeline getPipeline();

    // 日志
    Log getLogger();

    // 关联的Loader（类加载器）
    Loader getLoader();
    void setLoader(Loader loader);

    // 关联的Manager（Session管理器）
    Manager getManager();
    void setManager(Manager manager);

    // 关联的Realm（安全域）
    Realm getRealm();
    void setRealm(Realm realm);

    // 背景处理（周期性任务，如Session过期检查）
    void backgroundProcess();
}
```

```
这种统一的接口设计让Tomcat可以用完全相同的代码来处理不同层级的容器。
这就是"面向接口编程"的威力。
```

# 12.Pipeline-Valve管道阀门机制

```
Pipeline-Valve 是 Tomcat 内部实现请求处理的核心设计模式——责任链模式。

一、什么是Pipeline-Valve？

每个容器（Engine、Host、Context、Wrapper）都有一个 Pipeline。
Pipeline 中串联了多个 Valve（阀门），请求像水流一样依次穿过每个 Valve：

┌─────────────────────────────────────────────────┐
│                  Pipeline                        │
│                                                  │
│  请求 → [Valve 1] → [Valve 2] → [Valve 3] → ...  → [Basic Valve] → 响应
│                                                  │
└─────────────────────────────────────────────────┘

二、两个核心接口

Valve 接口：
```

```java
public interface Valve {
    // 获取下一个Valve
    Valve getNext();
    void setNext(Valve valve);

    // 执行阀门逻辑
    void invoke(Request request, Response response)
        throws IOException, ServletException;

    // 是否支持异步
    boolean isAsyncSupported();
}
```

```java
public interface Pipeline {
    Valve getBasic();       // 获取基础阀门（最后一个）
    void setBasic(Valve valve);

    void addValve(Valve valve);    // 添加自定义阀门
    Valve[] getValves();

    Valve getFirst();       // 获取第一个Valve（入口）

    boolean isAsyncSupported();
    Container getContainer();
    void setContainer(Container container);
}
```

```
三、一个请求的阀门执行顺序

以标准的本地Tomcat为例，请求从Connector进入容器后，经过的阀门链是：

CoyoteAdapter.service()
  │
  v
Engine.getPipeline().getFirst()
  ├─→ [自定义Valve 1]
  ├─→ [自定义Valve 2]
  └─→ StandardEngineValve（Basic Valve）
        │  选择对应的Host
        v
      Host.getPipeline().getFirst()
        ├─→ [AccessLogValve]（通常配置在Host级别的日志阀门）
        ├─→ [ErrorReportValve]（错误报告阀门）
        └─→ StandardHostValve（Basic Valve）
              │  选择对应的Context
              v
            Context.getPipeline().getFirst()
              ├─→ [自定义Valve]
              └─→ StandardContextValve（Basic Valve）
                    │  选择对应的Wrapper
                    v
                  Wrapper.getPipeline().getFirst()
                    └─→ StandardWrapperValve（Basic Valve）
                          │  创建Filter链 + 调用Servlet
                          v
                        FilterChain.doFilter(request, response)
                          └─→ Servlet.service(request, response)

四、Basic Valve——每个容器最核心的阀门

┌────────────────────┬──────────────────────────────────────────────┐
│ 容器                │ Basic Valve 的职责                           │
├────────────────────┼──────────────────────────────────────────────┤
│ StandardEngineValve│ 根据请求Host头选择对应的Host子容器              │
│                    │ 将请求转发给 Host.getPipeline().getFirst()    │
├────────────────────┼──────────────────────────────────────────────┤
│ StandardHostValve  │ 根据URI中的ContextPath选择对应的Context子容器   │
│                    │ 将请求转发给 Context.getPipeline().getFirst() │
├────────────────────┼──────────────────────────────────────────────┤
│ StandardContextValve│ 根据URI中的ServletPath选择对应的Wrapper子容器  │
│                    │ 将请求转发给 Wrapper.getPipeline().getFirst() │
├────────────────────┼──────────────────────────────────────────────┤
│ StandardWrapperValve│ *最关键！建立Filter链，调用Servlet.service()  │
│                    │ 处理完响应后，清理Filter链                     │
└────────────────────┴──────────────────────────────────────────────┘

五、StandardWrapperValve 源码精读

这是整个请求链路中最关键的一步，简化后的逻辑如下：
```

```java
// StandardWrapperValve.invoke() 简化的核心逻辑
public final void invoke(Request request, Response response) {

    // 1. 获取这个Wrapper对应的Servlet实例（如果还没创建就创建）
    StandardWrapper wrapper = (StandardWrapper) getContainer();
    Servlet servlet = wrapper.allocate();  // 注意：Servlet可能是单例多线程！

    // 2. 创建Filter链
    //    ApplicationFilterFactory.createFilterChain(request, wrapper, servlet)
    ApplicationFilterChain filterChain =
        ApplicationFilterFactory.createFilterChain(request, wrapper, servlet);

    // 3. 如果Filter链为空或所有Filter都通过了，执行FilterChain
    //    FilterChain.doFilter() 在执行完所有Filter后会调用 Servlet.service()
    if (filterChain != null) {
        filterChain.doFilter(request.getRequest(), response.getResponse());
    }

    // 4. 释放Servlet（单例模式下不销毁，只是归还计数器）
    wrapper.deallocate(servlet);

    // 5. 如果是一次性连接（Connection: close），设置响应完成后的清理动作
}
```

```
六、自定义 Valve 示例

你可以写自己的 Valve 插入到任意容器的 Pipeline 中：
```

```java
package com.example.valve;

import org.apache.catalina.Valve;
import org.apache.catalina.connector.Request;
import org.apache.catalina.connector.Response;
import org.apache.catalina.valves.ValveBase;

import javax.servlet.ServletException;
import java.io.IOException;

public class MyLoggingValve extends ValveBase {

    @Override
    public void invoke(Request request, Response response)
            throws IOException, ServletException {

        long start = System.currentTimeMillis();

        // 记录请求信息
        System.out.println("[MyValve] Request: " + request.getRequestURI());

        // *必须调用 getNext().invoke()，否则请求链就断了！
        getNext().invoke(request, response);

        // 请求处理完之后
        long end = System.currentTimeMillis();
        System.out.println("[MyValve] Response: " + response.getStatus()
                + " took " + (end - start) + "ms");
    }
}
```

```
在 server.xml 中配置：
```

```xml
<Host name="localhost" ...>
    <Valve className="com.example.valve.MyLoggingValve" />
</Host>
```

```
注意：
  - Valve 可以用在任何容器层级（Engine / Host / Context / Wrapper）
  - 如果 Valve 在 Host 级别，则该Host下的所有Context的所有请求都会经过它
  - 如果 Valve 在 Context 级别，则只对该webapp生效
  - 如果不调用 getNext().invoke()，请求就不会往下传递——相当于拦截了请求
```

# 13.Tomcat的类加载机制（上）—— 为什么打破双亲委派

```
类加载机制是Tomcat设计中最精妙的部分之一，也是面试必问话题。

一、JVM的"双亲委派"模型（快速复习）

先复习Java标准的类加载机制：

┌──────────────────────────────┐
│  Bootstrap ClassLoader       │ -- 加载 JRE/lib/rt.jar （JDK9+ 变为Platform）
│  （启动类加载器，C++实现）    │
├──────────────────────────────┤
│  Extension ClassLoader       │ -- 加载 JRE/lib/ext/ （JDK9+ 被废弃）
│  （扩展类加载器）             │
├──────────────────────────────┤
│  Application ClassLoader     │ -- 加载 classpath 下的类
│  （应用类加载器）             │
└──────────────────────────────┘

加载类的流程：
  1. Application ClassLoader 收到加载请求
  2. 先委托给父加载器 Extension ClassLoader
  3. Extension ClassLoader 再委托给 Bootstrap ClassLoader
  4. Bootstrap 在 rt.jar 中找，找到了→返回；没找到→
  5. Extension 在 ext/ 目录中找，找到了→返回；没找到→
  6. Application 在 classpath 中找，找到了→返回；没找到→ ClassNotFoundException

这叫"双亲委派"——先问爹，爹找不到再自己找。

二、为什么Tomcat不能直接用双亲委派？

假设你有两个web应用部署在同一个Tomcat上：
  - webapp A 依赖 my-lib-1.0.jar（里面有个 com.example.Util 类）
  - webapp B 依赖 my-lib-2.0.jar（里面也有个 com.example.Util 类，但实现不同！）

如果Tomcat用双亲委派：
  1. Tomcat的ClassLoader只加载了一个版本的com.example.Util
  2. 两个webapp共用一个类
  3. 如果两个版本的API不同，必定有一个webapp会出错！

再看一个场景：
  - webapp A 用了 Spring 5，依赖 spring-core-5.3.0.jar
  - webapp B 用了 Spring 6，依赖 spring-core-6.0.0.jar
  - 两个jar中有很多同名但不同实现的类！
  - 用双亲委派必然出问题

Tomcat必须解决的四个核心需求：

  需求1：隔离性
    webapp A 和 webapp B 各自的类互相不可见
    同名的类（不同版本）可以在各自的webapp中独立存在

  需求2：共享性
    多个webapp可能需要共享某些库（比如Tomcat的servlet-api.jar）
    不能每个webapp都加载一遍，浪费内存

  需求3：安全性
    webapp 不能访问 Tomcat 内部的类（防止恶意代码破坏Tomcat本身）
    webapp 不能覆盖 JRE 的核心类（比如不能自己写一个 java.lang.String）

  需求4：热部署
    更新一个webapp的类文件时，只需要重新加载这个webapp
    不需要重启整个Tomcat
    （销毁旧的ClassLoader → 创建新的ClassLoader → 重新加载类）

Java标准的双亲委派模型一个需求都满足不了！
所以Tomcat必须自己设计一套定制的类加载机制。
```

# 14.Tomcat的类加载机制（下）—— 类加载器层次与热部署

```
三、Tomcat的类加载器层次

Tomcat设计了一套打破双亲委派的类加载结构：

┌──────────────────────────────────────────────────────────┐
│                    Bootstrap ClassLoader                  │
│                    （JVM自带的，加载JRE核心类）              │
│                                                          │
│         ┌────────────────────┐                           │
│         │   System ClassLoader │ -- 启动脚本中指定 -classpath│
│         │   （Tomcat自身的启动类）│    catalina.jar,bootstrap.jar│
│         └─────────┬──────────┘                           │
│                   │                                      │
│         ┌─────────V──────────┐                           │
│         │   Common ClassLoader│ -- $CATALINA_BASE/lib      │
│         │   （共同类加载器）   │   *所有webapp和Tomcat共享  │
│         └────┬───────────┬───┘                           │
│              │           │                                │
│    ┌─────────V────┐ ┌───V────────────┐                   │
│    │Server ClassLoader│ Shared ClassLoader│               │
│    │（Tomcat内部用）  │  （共享webapp用） │                │
│    │ $CATALINA_HOME/* │ $CATALINA_BASE/  │               │
│    │            │ shared/lib（可选）     │               │
│    └────────────┘ └──────────────────┘                   │
│                                                          │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│   │ WebappClassLoader 1 │ WebappClassLoader 2 │...   │  │
│   │ /WEB-INF/classes    │ /WEB-INF/classes    │      │  │
│   │ /WEB-INF/lib/*.jar  │ /WEB-INF/lib/*.jar  │      │  │
│   │ *打破双亲委派       │ *打破双亲委派       │      │  │
│   └──────────────┘  └──────────────┘  └──────────────┘  │
└──────────────────────────────────────────────────────────┘

四、各个加载器的职责

┌─────────────────────┬──────────────────────────────────────────────┐
│ 类加载器              │ 加载路径与职责                                │
├─────────────────────┼──────────────────────────────────────────────┤
│ Bootstrap ClassLoader│ JRE/lib下的核心类（java.lang.*, java.util.*） │
│ System ClassLoader   │ Tomcat启动脚本classpath中的类                 │
│                      │ 主要是 bootstrap.jar 和 tomcat-juli.jar      │
├─────────────────────┼──────────────────────────────────────────────┤
│ Common ClassLoader   │ $CATALINA_BASE/lib 下的jar                   │
│                      │ 包括：servlet-api.jar, jsp-api.jar等          │
│                      │ * 这是所有webapp共享的类                      │
├─────────────────────┼──────────────────────────────────────────────┤
│ Server ClassLoader   │ Tomcat自身的类（catalina.jar等）              │
│                      │ （Tomcat 8.5+默认可见性设为false，较少用了）   │
├─────────────────────┼──────────────────────────────────────────────┤
│ Shared ClassLoader  │ $CATALINA_BASE/shared/lib（默认不存在）        │
│                      │ 如果你有所有webapp都要用的公共jar放这里        │
├─────────────────────┼──────────────────────────────────────────────┤
│ WebappClassLoader   │ /WEB-INF/classes + /WEB-INF/lib/*.jar        │
│                      │ 每个webapp有自己独立的WebappClassLoader       │
│                      │ * 加载顺序跟双亲委派是反的！                   │
└─────────────────────┴──────────────────────────────────────────────┘

五、WebappClassLoader 的加载顺序（打破双亲委派！）

这是最关键的区别——WebappClassLoader 的加载顺序和标准双亲委派完全反了：

Tomcat 的 WebappClassLoader.loadClass() 逻辑（简化版）：
```

```java
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {

    // 1. 先从自己的缓存中查找（是否已经加载过）
    Class<?> clazz = findLoadedClass(name);
    if (clazz != null) return clazz;

    // 2. *** 先自己尝试加载！（打破双亲委派的关键）
    //    先查 JRE 基础类（java.*, javax.* 等，交给System加载保证安全性）
    if (name.startsWith("java.") || name.startsWith("javax.")) {
        // JRE类：走标准双亲委派，不允许webapp自己定义
        clazz = system.loadClass(name);
    } else {
        try {
            // * 先在自己的 /WEB-INF/classes 和 /WEB-INF/lib 中找
            clazz = findClass(name);  // 自己的加载逻辑
        } catch (ClassNotFoundException e) {
            // 自己找不到，再委托给父加载器
            // （父加载器 = Shared → Common → System → Bootstrap）
            clazz = parent.loadClass(name);
        }
    }

    if (resolve) resolveClass(clazz);
    return clazz;
}
```

```
关键点：
  - 对于 JRE 核心类（java.* / javax.* 等），仍然走双亲委派（安全性）
  - 对于应用类，先自己在 WEB-INF 中加载，找不到再向上委托
  - 这就是"打破双亲委派"——优先自己找，不再先问爹！

  这样做的好处：
    webapp A 的 /WEB-INF/lib/my-lib-1.0.jar 中的 com.example.Util
    和 webapp B 的 /WEB-INF/lib/my-lib-2.0.jar 中的 com.example.Util
    完全不会冲突！各自在各自的 WebappClassLoader 中被加载。

六、热部署的原理

热部署（hot deploy）的本质是：销毁旧的ClassLoader，创建新的ClassLoader。

流程：
  1. Tomcat 检测到 webapp 的 war 文件或目录发生了变化
  2. 调用 StandardContext.reload()
  3. reload() 内部：
     a. 调用 stop()：销毁所有Servlet实例、Filter、Listener
     b. 销毁旧的 WebappClassLoader（置于可回收状态）
        - 此时旧的类对象如果没有引用，将被GC回收
     c. 创建新的 WebappClassLoader
     d. 调用 start()：重新加载所有类、初始化Servlet

  4. 整个过程不需要重启JVM，这就是热部署！

注意事项：
  - 如果有ThreadLocal引用了旧ClassLoader加载的类对象，会导致内存泄漏
  - Tomcat的 ThreadLocalLeakPreventionListener 就是用来检测和清理这种泄漏的
  - 频繁热部署会导致Metaspace碎片化（JDK8+类信息存Metaspace）

七、为什么你的Servlet不会被webapp A的Servlet干扰？

现在你应该能回答这个问题了：
  webapp A 和 webapp B 各有一个 UserServlet，
  它们由不同的 WebappClassLoader 加载，
  虽然在JVM中类名完全一样，但属于不同的"类命名空间"。
  JVM 识别一个类用的是 "全限定名 + 类加载器"，两者缺一不可。
  同一个ClassLoader不能加载两个同名的类，但不同ClassLoader可以。
```

# 15.Lifecycle接口与生命周期管理

```
一、什么是Lifecycle？

Tomcat中几乎所有重要组件（Server、Service、Connector、Container、Valve...）
都实现了 Lifecycle 接口。这个接口定义了一个组件的生命周期。

为什么需要Lifecycle？
  Tomcat的组件非常多（几十个），它们之间有严格的初始化和启动顺序：
    先初始化Server → 再初始化Service → 再初始化Connector和Container → ...
    这些组件需要像多米诺骨牌一样按顺序一个个启动。

  如果每个组件都各搞一套启动逻辑，整个Tomcat就是一盘散沙。
  Lifecycle 接口给所有组件统一了生命周期规范。

二、Lifecycle接口定义
```

```java
public interface Lifecycle {
    // 生命周期事件常量
    String BEFORE_INIT_EVENT = "before_init";
    String AFTER_INIT_EVENT = "after_init";
    String START_EVENT = "start";
    String BEFORE_START_EVENT = "before_start";
    String AFTER_START_EVENT = "after_start";
    String STOP_EVENT = "stop";
    String BEFORE_STOP_EVENT = "before_stop";
    String AFTER_STOP_EVENT = "after_stop";
    String PERIODIC_EVENT = "periodic";      // 周期性事件（如Session清理）
    String CONFIGURE_START_EVENT = "configure_start";
    String CONFIGURE_STOP_EVENT = "configure_stop";

    // 核心生命周期方法
    void init() throws LifecycleException;
    void start() throws LifecycleException;
    void stop() throws LifecycleException;
    void destroy() throws LifecycleException;

    // 获取当前生命周期状态
    LifecycleState getState();
    String getStateName();

    // 监听器管理
    void addLifecycleListener(LifecycleListener listener);
    LifecycleListener[] findLifecycleListeners();
    void removeLifecycleListener(LifecycleListener listener);
}
```

```
三、生命周期状态机

Tomcat的生命周期不是简单的"启动→运行→停止"，而是一个精密的状态机：

┌──────┐   init()   ┌─────────────┐
│ NEW  │ --------- │ INITIALIZED │
└──────┘            └──────┬──────┘
                           │ start()
          ┌────────────────┼────────────────┐
          v                v                v
   ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
   │STARTING_PREP│→│  STARTING   │→│   STARTED   │ -- 正常运行状态
   └─────────────┘ └─────────────┘ └──────┬──────┘
                                          │ stop()
          ┌───────────────────────────────┼───────────────┐
          v                               v               v
   ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
   │STOPPING_PREP│→│  STOPPING   │→│   STOPPED   │
   └─────────────┘ └─────────────┘ └──────┬──────┘
                                          │ destroy()
                                          v
                                   ┌─────────────┐
                                   │  DESTROYED  │
                                   └─────────────┘

每个状态转换时，都会触发对应的事件（LifecycleEvent），
注册的 LifecycleListener 会收到通知。

四、LifecycleBase：模板方法模式的体现

实际的实现类是 LifecycleBase（抽象类），所有Tomcat组件都继承它：
```

```java
public abstract class LifecycleBase implements Lifecycle {

    @Override
    public final synchronized void init() throws LifecycleException {
        // 1. 检查状态（必须是NEW才能init）
        if (!state.equals(LifecycleState.NEW)) {
            invalidTransition(BEFORE_INIT_EVENT);
        }

        try {
            // 2. 状态设为 INITIALIZING
            setStateInternal(LifecycleState.INITIALIZING, null, false);

            // 3. * 调用子类的初始化方法（模板方法）
            initInternal();

            // 4. 状态设为 INITIALIZED
            setStateInternal(LifecycleState.INITIALIZED, null, false);
        } catch (Throwable t) {
            handleSubClassException(t, "lifecycleBase.initFail", toString());
        }
    }

    // * 留给子类实现的模板方法
    protected abstract void initInternal() throws LifecycleException;
    protected abstract void startInternal() throws LifecycleException;
    protected abstract void stopInternal() throws LifecycleException;
    protected abstract void destroyInternal() throws LifecycleException;
}
```

```
LifecycleBase 的 start() 方法也是同样的套路：
  1. 状态检查
  2. 发射 BEFORE_START_EVENT 事件
  3. 调用子类的 startInternal()
  4. 发射 START_EVENT 事件
  5. 设置状态为 STARTED
  6. 发射 AFTER_START_EVENT 事件

五、观察者模式：LifecycleListener

任何组件都可以注册 LifecycleListener，在状态变化时执行逻辑。

比如 AprLifecycleListener 就是 Service 的一个监听器：
  - 收到 BEFORE_INIT_EVENT → 尝试初始化APR本地库
  - 收到 AFTER_START_EVENT → 记录APR是否加载成功

你也可以自定义一个监听器：
```

```java
public class MyLifecycleListener implements LifecycleListener {
    @Override
    public void lifecycleEvent(LifecycleEvent event) {
        String type = event.getType();
        Object source = event.getLifecycle();

        if (Lifecycle.AFTER_START_EVENT.equals(type)) {
            System.out.println(source + " 启动完成！");
        } else if (Lifecycle.BEFORE_STOP_EVENT.equals(type)) {
            System.out.println(source + " 即将停止...");
        }
    }
}
```

```
在 server.xml 中注册到某个组件上：
```

```xml
<Server port="8005" shutdown="SHUTDOWN">
    <Listener className="com.example.MyLifecycleListener" />
</Server>
```

```
六、lifecycle 在 Tomcat 中的实际使用场景

1. GlobalResourcesLifecycleListener：
   监听 Server 启动 → 注册全局JNDI资源到JMX

2. JreMemoryLeakPreventionListener：
   在 Server 初始化时触发 → 修复JDK的DriverManager内存泄漏bug
   （在类加载时预先加载DriverManager，防止webapp的ClassLoader造成泄漏）

3. ThreadLocalLeakPreventionListener：
   在 Context 停止时触发 → 清理可能残留的 ThreadLocal 引用

4. NamingContextListener：
   监听 Container 启动 → 创建JNDI上下文

七、背景处理：backgroundProcess()

除了标准的init/start/stop/destroy，Lifecycle还有一个"后台任务"机制：

public void backgroundProcess();

这个方法的调用周期是固定的（默认10秒），由 ContainerBackgroundProcessor 线程调用。
常用于：
  - Session过期检查和清理
  - 集群心跳
  - Context的reload检测（检查web.xml是否变化）
```

# 16.Mapper：请求路由映射机制

```
Connector把请求交给Container后，Container怎么知道该由哪个Servlet处理？
答案是：Mapper。

一、Mapper 是什么？

Mapper 是 Connector 和 Container 之间的"导航系统"。
它维护了一张巨大的路由表，记录了：
  哪个 Host → 哪些 Context → 哪些 Wrapper（Servlet）

当请求进来时，Mapper 根据 URL 的分段路径，快速定位到最终的目标 Wrapper。

二、URL 的分解

Tomcat 把一个完整的 URL 拆成几个部分：

  http://localhost:8080/oa/user/list?id=123
  ------ ------- --- -- -------- ------
  协议    主机名   端口 Context  Servlet  参数
                       Path      Path

  /oa          → Context Path（决定哪个webapp）
  /user/list   → Servlet Path（决定哪个Servlet，从Context内部开始匹配）
  如果还剩下路径 → Path Info（留给Servlet做更精细的路由，如RESTful风格）

三、Mapper 的路由过程

MapperListener 监听到容器的添加/删除事件后，会更新 Mapper 的路由表。
请求到达时，Mapper.map() 的执行过程：
```

```java
// Mapper.map() 简化逻辑
public void map(MessageBytes host, MessageBytes uri,
                MappingData mappingData) throws IOException {

    // 1. 根据 Host 头找到对应的 Host 对象（精确匹配 vs defaultHost）
    //    如果请求的Host在Mapper中没有，使用Engine的defaultHost

    // 2. 在 Host 下找到对应的 Context
    //    从URI中提取 ContextPath（最长匹配原则）
    //    例如：/oa/user/list → ContextPath = /oa

    // 3. 在 Context 下找到对应的 Wrapper
    //    去掉ContextPath后剩下的路径就是匹配Servlet的部分
    //    匹配规则（按优先级）：

    //    3a. 精确匹配（最高优先级）
    //        <url-pattern>/user/login</url-pattern>
    //        请求 /oa/user/login → 精确命中

    //    3b. 前缀匹配（通配符：/*）
    //        <url-pattern>/user/*</url-pattern>
    //        请求 /oa/user/xxx → 前缀命中
    //        匹配最长的前缀

    //    3c. 扩展名匹配
    //        <url-pattern>*.jsp</url-pattern>
    //        请求 /oa/index.jsp → 扩展名命中

    //    3d. 默认匹配（最低优先级）
    //        <url-pattern>/</url-pattern>（DefaultServlet）
    //        以上都不匹配时使用（通常是静态资源）

    mappingData.host = matchedHost;
    mappingData.context = matchedContext;
    mappingData.wrapper = matchedWrapper;
}
```

```
四、路由匹配的优先级规则

┌──────────────┬──────────────────────┬─────────────────────────┐
│ 优先级        │ url-pattern 示例      │ 匹配条件                  │
├──────────────┼──────────────────────┼─────────────────────────┤
│ 1（最高）     │ /user/login          │ 路径完全一致              │
│ 2           │ /user/*              │ 以 /user/ 开头            │
│ 3           │ *.jsp / *.do         │ 以指定扩展名结尾          │
│ 4（最低）     │ /                    │ 其他都不匹配时兜底        │
└──────────────┴──────────────────────┴─────────────────────────┘

五、MapperListener：自动更新路由表

MapperListener 实现了 ContainerListener（容器监听器）和 LifecycleListener。
当容器层次发生变化时（新增Context、删除Servlet等），它自动更新Mapper路由表。

监听的事件：
  - Host 被添加 → 在 Mapper 中注册这个 Host
  - Context 被添加 → 在对应 Host 下注册这个 Context
  - Wrapper 被添加（新Servlet部署）→ 在对应 Context 下注册这个 Wrapper
  - Wrapper 被删除 → 从 Mapper 中移除这个 Wrapper
  - Context 被停止 → 从 Mapper 中移除该 Context 下的所有路由

六、为什么需要 Mapper 而不是直接遍历容器？

性能！想象一下：
  - 5个Host
  - 每个Host有 20个Context
  - 每个Context有 50个Servlet（Wrapper）
  - 总共需要匹配 5 × 20 × 50 = 5000 次

Mapper 将 Host/Context/Wrapper 组织成多层Map结构，可以 O(log n) 找到目标，
而不是 O(n) 遍历。
```

# 17.Tomcat启动流程（上）—— Bootstrap到Server

```
Tomcat的启动看似简单（双击startup.bat），但背后的流程非常复杂。
我们从入口开始，一层层剖析。

一、启动入口：startup.bat → catalina.bat

startup.bat 最终调用的是 catalina.bat start，而 catalina.bat 的核心是：
```

```bash
# catalina.bat 简化逻辑
set CLASSPATH=%CATALINA_HOME%\bin\bootstrap.jar;%CATALINA_HOME%\bin\tomcat-juli.jar

# Bootstrap 的 main 方法
%_RUNJAVA% %JAVA_OPTS% %CATALINA_OPTS% ^
  -classpath %CLASSPATH% ^
  -Dcatalina.base=%CATALINA_BASE% ^
  -Dcatalina.home=%CATALINA_HOME% ^
  -Djava.io.tmpdir=%CATALINA_TMPDIR% ^
  org.apache.catalina.startup.Bootstrap start
```

```
注意：
  - classpath 只包含 bootstrap.jar 和 tomcat-juli.jar
  - catalina.jar 不在classpath中！（这就是后面要讲的类加载器隔离）
  - JAVA_HOME必须指向JDK（JSP编译需要javac）
  - catalina.base 和 catalina.home 可以不同（Tomcat的"实例分离"特性）

二、Bootstrap.main() —— 启动的总指挥

Bootstrap 是整个Tomcat的"启动器"，它的类图非常简单：
  - 它不在任何Tomcat的类加载器中
  - 它由 JVM 的 Application ClassLoader 加载
  - 它通过反射创建 Catalina 实例，隔离了Tomcat内部类

Bootstrap 做了什么？
```

```java
// Bootstrap.main() 简化流程
public static void main(String args[]) {
    Bootstrap bootstrap = new Bootstrap();

    // 1. 初始化 Bootstrap
    bootstrap.init();

    // 2. 根据参数执行不同的操作
    if ("start".equals(args[0])) {
        bootstrap.load(args);    // 加载配置
        bootstrap.start();       // 启动
    } else if ("stop".equals(args[0])) {
        bootstrap.stopServer(args);
    }
}
```

```
三、Bootstrap.init() —— 创建三大ClassLoader

这是Tomcat类加载机制的核心初始化：
```

```java
public void init() throws Exception {
    // 1. 创建 Common ClassLoader
    //    加载路径：$CATALINA_BASE/lib + $CATALINA_HOME/lib
    //    这个ClassLoader的父加载器：Application ClassLoader
    ClassLoader commonLoader = createClassLoader("common", null);

    // 2. 创建 Catalina ClassLoader（Server ClassLoader）
    //    加载路径：$CATALINA_HOME/server/lib（默认不存在，指向common）
    //    父加载器：Common ClassLoader
    ClassLoader catalinaLoader = createClassLoader("server", commonLoader);

    // 3. 创建 Shared ClassLoader
    //    加载路径：$CATALINA_BASE/shared/lib（默认不存在，指向common）
    //    父加载器：Common ClassLoader
    ClassLoader sharedLoader = createClassLoader("shared", commonLoader);

    // 4. * 关键操作：用 Catalina ClassLoader 加载 Catalina 类
    //    Thread.currentThread().setContextClassLoader(catalinaLoader)
    //    然后用反射创建 org.apache.catalina.startup.Catalina 对象
    Thread.currentThread().setContextClassLoader(catalinaLoader);

    Class<?> startupClass = catalinaLoader.loadClass("org.apache.catalina.startup.Catalina");
    Object startupInstance = startupClass.getConstructor().newInstance();

    // 5. 通过反射设置 Catalina 的父 ClassLoader（Shared ClassLoader）
    //    WebappClassLoader 的父加载器就是 Shared ClassLoader
    Method setParentClassLoader = startupClass.getMethod("setParentClassLoader", ClassLoader.class);
    setParentClassLoader.invoke(startupInstance, sharedLoader);

    // 6. 把 Catalina 实例保存在 Bootstrap 的 catalinaDaemon 字段中
    this.catalinaDaemon = startupInstance;
}
```

```
四、Bootstrap.load() —— 加载 server.xml

通过反射调用 Catalina.load()，它做了这些事情：
```

```java
// Catalina.load() 简化流程
public void load() {
    // 1. 用 Digester 框架解析 server.xml
    //    Digester 是 Apache 的一个 XML → Java对象 映射工具
    //    每遇到一个 <Server> 标签，就 new StandardServer()
    //    每遇到一个 <Service> 标签，就 new StandardService()
    //    每遇到一个 <Connector> 标签，就 new Connector()
    //    自动设置属性和父子关系

    // 2. 调用 Server.init()
    //    Server.init() → Service.init() → Engine.init() → Connector.init()
    //    Host 和 Context 的 init 稍后发生（在 start 阶段）
    Digester digester = createStartDigester();
    File serverXml = new File(getConfigFile(), "server.xml");
    Server server = (Server) digester.parse(serverXml.getInputStream());

    // 3. 设置 Server 的 Catalina 引用
    server.setCatalina(this);

    // 4. Server 的 init() 被调用
    //    这一步会初始化所有在 server.xml 中定义的第一层组件
    getServer().init();
}
```

```
五、Digester 是什么？

Digester 是 Apache Commons 的一个 XML 解析框架，它用"规则链"的方式把 XML 元素
映射为 Java 对象。

简化示例：
```

```java
// 类似这样定义映射规则：
Digester digester = new Digester();

// 遇到 <Server> → 创建 StandardServer 对象
digester.addObjectCreate("Server", "org.apache.catalina.core.StandardServer");
// 把 Server 的属性 port 映射到 setPort() 方法
digester.addSetProperties("Server");

// 遇到 <Server>/<Service> → 创建 StandardService
digester.addObjectCreate("Server/Service", "org.apache.catalina.core.StandardService");
digester.addSetProperties("Server/Service");
// 把 Service 加入到 Server 中（调用 addChild）
digester.addSetNext("Server/Service", "addService");

// 遇到 <Server>/<Service>/<Connector> → 创建 Connector
digester.addObjectCreate("Server/Service/Connector", "org.apache.catalina.connector.Connector");
digester.addSetProperties("Server/Service/Connector");
// 把 Connector 加入到 Service 中
digester.addSetNext("Server/Service/Connector", "addConnector");

// ... 以此类推
```

```
这样，server.xml 中的每个元素都变成了一个 Java 对象，并被正确地"挂"到父对象上。
至此，Server → Service → Connector → Engine → Host 的 Java 对象树就建立起来了。
```

# 18.Tomcat启动流程（下）—— Service到Context

```
六、Bootstrap.start() → Catalina.start() → Server.start()

Server.init() 只是创建对象、设置属性，真正让一切运行起来的是 start()。

start() 的执行链路：
  Bootstrap.start()
    → Catalina.start()
      → Server.start()
        → 对所有 Service 执行 start()

七、Service.start()
```

```java
public void startInternal() throws LifecycleException {

    // 1. * 先启动 Engine（容器）
    engine.start();
    // Engine.start() → Host.start() … 但这个阶段Host还没有子Context
    // Context 的加载是后面自动部署做的事

    // 2. 启动 Executor（线程池，如果在 server.xml 中配置了的话）
    for (Executor executor : findExecutors()) {
        executor.start();
    }

    // 3. 启动 MapperListener
    //    MapperListener 会监听容器变化，自动更新路由表
    mapperListener.start();

    // 4. * 启动 Connector（连接器）
    //    连接器开始监听端口，这是最后一步！
    //    如果先启动Connector，请求进来了但容器还没准备好就出事了
    for (Connector connector : connectors) {
        connector.start();
    }
}
```

```
注意启动顺序：
  ① 先启动容器（Engine + Host）
  ② 再启动 MapperListener
  ③ 最后启动 Connector
这样做保证了当Connector开始接收请求时，容器已经准备好处理了。

八、Connector.start() 内部

Connector 的启动主要是启动 ProtocolHandler：
  ProtocolHandler.start() →
    Endpoint.start() →
      - 创建 Acceptor 线程（开始监听端口）
      - 创建 Poller 线程（开始轮询）
      - 初始化 Worker 线程池

此时，Tomcat 已经可以接收外部请求了！

九、Host 的自动部署

Connector启动后，Host 的自动部署机制开始工作：

StandardHost.start() 内部会调用 deployApps()：
```

```java
protected void deployApps() {
    // 1. 扫描 $CATALINA_BASE/conf/[EngineName]/[HostName]/ 下的XML文件
    //    每个XML文件定义了一个Context的配置
    deployDescriptors(configBase.list());

    // 2. 扫描 $CATALINA_BASE/webapps/ 下的 war 文件
    deployWARs(appBase.list());

    // 3. 扫描 $CATALINA_BASE/webapps/ 下的目录（已解压的webapp）
    deployDirectories(appBase.list());
}
```

```
对于每个war包/目录，Tomcat会创建一个 StandardContext 对象并启动它。

十、StandardContext.startInternal() —— 整个Tomcat最复杂的方法

这是 Tomcat 源码中最复杂的方法之一（几百行代码），它做了这些事情：
```

```java
// StandardContext.startInternal() 简化流程
protected synchronized void startInternal() throws LifecycleException {

    // 1. * 创建 WebappClassLoader（为该webapp创建独立的类加载器）
    if (getLoader() == null) {
        WebappLoader webappLoader = new WebappLoader(getParentClassLoader());
        webappLoader.setDelegate(delegate);  // delegate=false 表示打破双亲委派
        setLoader(webappLoader);
    }

    // 2. 发送 "configure_start" 事件
    //    这会触发 ContextConfig 监听器

    // 3. ContextConfig 完成以下工作（这是最耗时的部分！）：
    //    a. 解析 /WEB-INF/web.xml
    //    b. 扫描 jar 包中的 /META-INF/web-fragment.xml（Servlet 3.0+ 片段）
    //       * 这是启动慢的主要原因之一！需要遍历所有jar包的META-INF目录
    //    c. 扫描注解：@WebServlet, @WebFilter, @WebListener
    //       * 需要扫描 /WEB-INF/classes 和 /WEB-INF/lib 下的所有类文件！
    //    d. 合并所有的 web.xml / web-fragment.xml / 注解配置
    //    e. 创建 Servlet/Filter/Listener 的包装对象
    //    f. 设置默认的 Servlet（DefaultServlet）和 JspServlet

    // 4. 创建 Session 管理器
    if (getManager() == null) {
        setManager(new StandardManager());
    }

    // 5. 初始化字符编码 Filter（SetCharacterEncodingFilter，如果配置的话）

    // 6. 初始化 JMX MBean

    // 7. 处理 load-on-startup 的 Servlet
    //    loadOnStartup() → 值为正数的Servlet按值从小到大依次初始化
    //    load-on-startup=1 的先创建，-1 的在第一次请求时创建

    // 8. 启动后台处理线程（定期执行 backgroundProcess）
    ContainerBackgroundProcessorMonitor.start();

    // 9. 设置状态为 STARTED
    setState(LifecycleState.STARTED);
}
```

```
十一、为什么Tomcat启动有时候很慢？

看了上面的启动流程，你应该能理解启动慢的几个原因：

1. Jar 扫描：
   /WEB-INF/lib 下有大量jar包时，Tomcat要遍历每个jar的 META-INF 目录
   查找 web-fragment.xml 和 TLD文件（JSP标签定义）
   解决办法：在 Context 中配置 JarScanner 跳过不需要扫描的jar：
```

```xml
<Context>
    <JarScanner>
        <JarScanFilter defaultPluggabilityScan="false"
                       defaultTldScan="false" />
    </JarScanner>
</Context>
```

```
2. 注解扫描：
   Servlet 3.0+ 引入了注解，Tomcat要扫描 /WEB-INF/classes 下所有.class文件
   解决办法：在 web.xml 中设置 metadata-complete="true" 跳过注解扫描：
```

```xml
<web-app metadata-complete="true" ...>
```

```
3. SecureRandom（Linux上最常见的问题）：
   JVM 启动时初始化 SecureRandom 需要收集熵值
   在Linux上，如果熵池不足，会阻塞很长时间
   解决办法：
   在 catalina.sh 中加 -Djava.security.egd=file:/dev/./urandom

4. 数据库连接池初始化：
   如果配置了 load-on-startup 的 Servlet/MemoryRealm 等，
   它们启动时可能会尝试连接数据库，如果数据库连不上就会超时等待。

5. DNS 解析：
   Tomcat 启动时可能会尝试解析本机的主机名，如果 DNS 不可达就会很慢。
   在 /etc/hosts 中确保 127.0.0.1 对应 localhost 和你的主机名。
```

# 19.一个HTTP请求的完整旅程（上）—— 从Socket到Adapter

```
把前面学的所有知识串起来，看一个HTTP请求从进来到出去的完整路径。

一、阶段一：连接建立

1. 浏览器发起 TCP 连接请求
   SYN → 服务器 8080 端口

2. Tomcat 的 NioEndpoint 中的 Acceptor 线程调用 ServerSocketChannel.accept()
   接收到这个新连接

3. Acceptor 检查 LimitLatch 计数器是否还有余量（maxConnections 限制）
   如果没超过上限 → 继续
   如果超过了 → Acceptor 阻塞等待

4. Acceptor 把 SocketChannel 设置为非阻塞模式

5. Acceptor 把 SocketChannel 注册到 Poller 的事件队列中
   Poller 内部有一个 SynchronizedQueue，Acceptor 往里面添加 PollerEvent

二、阶段二：IO检测与数据读取

6. Poller 线程从自己的事件队列中取出 PollerEvent
   把 SocketChannel 注册到 Selector 上（感兴趣的事件是 OP_READ）

7. Poller 调用 Selector.select() 检测哪些 SocketChannel 有数据可读
   这是 NIO 的核心——一个线程管理成千上万个连接

8. Selector 返回有 OP_READ 事件的 SelectionKey
   Poller 遍历这些 SelectionKey

9. 对于每一个可读的 SocketChannel：
   Poller 创建 SocketProcessor 任务
   把 SocketProcessor 提交给 Worker 线程池（Executor）

三、阶段三：HTTP协议解析

10. Worker 线程池中的一个空闲线程拿到 SocketProcessor 任务
    开始执行 SocketProcessor.run()

11. SocketProcessor 找到对应的 Http11Processor（每个连接关联一个 Processor）

12. Http11Processor 从 Socket 中读取字节数据到 InputBuffer

13. Http11Processor 按照 HTTP 协议逐字节解析：
```

```java
// 简化的HTTP解析过程
// 第一步：解析请求行
// "GET /oa/user/list?page=1 HTTP/1.1\r\n"
String method = parseMethod();        // "GET"
String uri = parseURI();             // "/oa/user/list?page=1"
String protocol = parseProtocol();   // "HTTP/1.1"

// 第二步：解析请求头
// "Host: localhost:8080\r\n"
// "User-Agent: Mozilla/5.0\r\n"
// "Accept: text/html\r\n"
// ...
while (hasMoreHeaders()) {
    String name = parseHeaderName();   // "Host"
    String value = parseHeaderValue(); // "localhost:8080"
    request.addHeader(name, value);
}

// 第三步：如果有请求体，根据 Transfer-Encoding 或 Content-Length 读取
if ("chunked".equals(request.getHeader("Transfer-Encoding"))) {
    readChunkedBody(request);
} else if (request.getContentLength() > 0) {
    readFixedLengthBody(request, request.getContentLength());
}
```

```
14. Http11Processor 构建内部对象：
    org.apache.coyote.Request    —— 存储解析结果
    org.apache.coyote.Response   —— 响应包装

四、阶段四：CoyoteAdapter 桥接

15. Http11Processor 调用 adapter.service(coyoteReq, coyoteRes)
    即 CoyoteAdapter.service()

16. CoyoteAdapter.service() 做了这些：
```

```java
// CoyoteAdapter.service() 的核心逻辑
public void service(org.apache.coyote.Request req,
                    org.apache.coyote.Response res) throws Exception {

    // 1. 把 coyote.Request/Response 包装为
    //    org.apache.catalina.connector.Request/Response（实现了HttpServletRequest/Response）
    //    这里使用了"门面模式"：request.getRequest() 获取真正的 RequestFacade

    // 2. postParseRequest() —— 关键步骤！
    //    a. 解析请求的 scheme（http/https）
    //    b. 调用 Mapper.map() 进行路由：
    //       根据Host头 → 找对应的 Host 容器
    //       根据URI中的ContextPath → 找对应的 Context 容器
    //       根据剩余路径 → 找对应的 Wrapper（Servlet）
    //       把结果存入 request.getMappingData()

    // 3. 如果找到了目标 Wrapper：
    //    调用 connector.getService().getContainer().getPipeline().getFirst().invoke(request, response);
    //    ↑ 这一行标志着请求正式进入 Catalina 容器！
}
```

```
至此，请求已经从网络字节流变成了一个结构化的 Java 对象，
并且 Tomcat 已经知道该由哪个 Servlet 来处理它了。
接下来就要进入容器的Pipeline-Valve执行链。
```

# 20.一个HTTP请求的完整旅程（下）—— 从Valve到Servlet

```
五、阶段五：容器Pipeline执行

17. Engine Pipeline 的 StandardEngineValve.invoke()
    根据请求的 Host 头，选择对应的 Host 子容器
    调用 host.getPipeline().getFirst().invoke(request, response)

18. Host Pipeline 的 StandardHostValve.invoke()
    根据请求URI的 ContextPath（如 /oa），选择对应的 Context 子容器
    调用 context.getPipeline().getFirst().invoke(request, response)

19. Context Pipeline 的 StandardContextValve.invoke()
    根据请求URI的剩余部分（如 /user/list），选择对应的 Wrapper 子容器
    调用 wrapper.getPipeline().getFirst().invoke(request, response)

六、阶段六：StandardWrapperValve —— 最关键的阀门

20. StandardWrapperValve.invoke() 做的工作：
```

```java
public final void invoke(Request request, Response response) {

    StandardWrapper wrapper = (StandardWrapper) getContainer();

    // 1. * 获取 Servlet 实例
    //    如果是第一次访问，创建Servlet实例（通过反射调用无参构造）
    //    然后调用 Servlet.init(servletConfig)
    Servlet servlet = wrapper.allocate();

    // 2. ** 创建 Filter 链
    //    把匹配当前请求路径的所有 Filter 按照：
    //      2a. url-pattern 匹配的 Filter（按 web.xml 中的声明顺序）
    //      2b. servlet-name 匹配的 Filter（按 web.xml 中的声明顺序）
    //    组织成一个 ApplicationFilterChain
    ApplicationFilterChain filterChain =
        ApplicationFilterFactory.createFilterChain(request, wrapper, servlet);

    //    createFilterChain 的内部逻辑：
    //    - 遍历 Context 中所有的 FilterMap
    //    - 如果 FilterMap 的 url-pattern 匹配当前请求路径 → 加入链
    //    - 如果 FilterMap 的 servlet-name 匹配当前 Servlet → 加入链
    //    - 最后把目标 Servlet 也挂到链的末尾

    // 3. *** 执行 Filter 链
    //    Filter链的结构：
    //    Filter1.doFilter() → Filter2.doFilter() → Filter3.doFilter()
    //    → ...
    //    → Servlet.service(request, response)
    //    → ...
    //    ← Filter3的后续代码
    //    ← Filter2的后续代码
    //    ← Filter1的后续代码
    filterChain.doFilter(request.getRequest(), response.getResponse());

    // 4. 释放 Servlet 实例
    //    对于单例的Servlet（SingleThreadModel已废弃），只是引用计数-1
    //    不销毁实例
    wrapper.deallocate(servlet);
}
```

```
七、阶段七：Filter链的执行

21. ApplicationFilterChain.doFilter()

Filter 链的执行是一个"洋葱模型"（也叫"俄罗斯套娃"）：

                    ┌─────────────────────────┐
                    │     Filter 1             │
                    │  ┌─────────────────────┐ │
                    │  │    Filter 2          │ │
                    │  │ ┌─────────────────┐ │ │
                    │  │ │   Filter 3       │ │ │
                    │  │ │ ┌─────────────┐ │ │ │
                    │  │ │ │  Servlet    │ │ │ │
                    │  │ │ │ .service()  │ │ │ │
                    │  │ │ └─────────────┘ │ │ │
                    │  │ │  Filter 3 后处理 │ │ │
                    │  │ └─────────────────┘ │ │
                    │  │   Filter 2 后处理    │ │
                    │  └─────────────────────┘ │
                    │    Filter 1 后处理        │
                    └─────────────────────────┘

每一次 filterChain.doFilter(request, response) 都会：
  1. 取出当前要执行的 Filter
  2. 调用 filter.doFilter(request, response, filterChain)
  3. Filter 内部处理完后，再次调用 filterChain.doFilter() → 进入下一个Filter
  4. 所有Filter执行完后，filterChain 内部调用 servlet.service(request, response)
  5. 然后逐层返回

八、阶段八：响应返回

22. Servlet 执行完毕，写入了响应（通过 response.getWriter() 或 response.getOutputStream()）

23. Filter 链逐层返回，各Filter中 doFilter() 之后的后处理代码执行

24. 返回到 StandardWrapperValve，再逐层返回到 StandardContextValve
    → StandardHostValve → StandardEngineValve → CoyoteAdapter

25. CoyoteAdapter 检查响应状态
    - 如果有异常 → 处理错误页面
    - 如果没有异常 → 准备写响应到客户端

26. Processor 把 Response 对象序列化为 HTTP 响应报文
    - 写状态行（HTTP/1.1 200 OK）
    - 写响应头（Content-Type、Set-Cookie等）
    - 写响应体

27. Processor 把报文写入 Socket 的输出缓冲区

28. 如果 "Connection: keep-alive"：
    - 不关闭 Socket
    - 把连接重新注册到 Poller，等待下一个请求
    - 如果达到 maxKeepAliveRequests 上限，关闭连接

29. 如果 "Connection: close"：
    - 关闭 Socket
    - Connection 数 -1（LimitLatch计数器 +1）

整个请求的生命周期完成了！

九、一张图总结全部流程

外网
 │
 V
[TCP连接到达:8080]
 │
 V
Acceptor 接收 ---- Poller 检测IO ---- Worker线程池
                                        │
                                        V
                                  Http11Processor 解析HTTP报文
                                        │
                                        V
                                   CoyoteAdapter.service()
                                   ├─ postParseRequest()
                                   ├─ Mapper.map() 路由
                                   └─ Engine Pipeline 入口
                                        │
                                        V
                                  StandardEngineValve（选Host）
                                        │
                                        V
                                  StandardHostValve（选Context）
                                        │
                                        V
                                  StandardContextValve（选Wrapper）
                                        │
                                        V
                                  StandardWrapperValve
                                   ├─ 创建/获取 Servlet
                                   ├─ 创建 Filter 链
                                   └─ FilterChain.doFilter()
                                        │
                                        V
                                   Filter1 → Filter2 → ... → Servlet.service()
                                        │
                                        V
                                  逐层返回 → Processor写响应 → Socket发送 → 客户端收到
```

# 21.Tomcat线程模型与线程池调优

```
一、Tomcat的线程池

Tomcat 有两套线程池：

1. 系统默认：java.util.concurrent.ThreadPoolExecutor
   - 创建方式在 NioEndpoint 内部

2. Tomcat定制的：org.apache.tomcat.util.threads.ThreadPoolExecutor
   - 扩展了 JDK 的 ThreadPoolExecutor
   - 增加了"先增加线程到最大，再入队"的行为（不同于JDK默认的先入队再增加线程）

两者的区别：
  JDK标准线程池的行为：
    核心线程满 → 任务入队列 → 队列满 → 创建新线程到max → 拒绝

  Tomcat线程池的行为：
    核心线程满 → 继续创建新线程到max → max满 → 任务入队列 → 队列满 → 拒绝

  Tomcat 的改法更适合Web服务器：
    Web请求不应该等待（用户在浏览器那头等着呢！）
    应该尽快创建线程处理，而不是先排队

二、核心参数详解（server.xml 中的 Connector 参数）
```

```xml
<Connector port="8080"
           maxThreads="200"
           minSpareThreads="10"
           maxConnections="10000"
           acceptCount="100"
           connectionTimeout="20000"
           maxKeepAliveRequests="100" />
```

```
┌──────────────────────┬─────────┬──────────────────────────────────────┐
│ 参数                  │ 默认值   │ 含义                                  │
├──────────────────────┼─────────┼──────────────────────────────────────┤
│ maxThreads           │ 200     │ 最大工作线程数                         │
│                      │         │ 决定能同时处理多少请求                 │
│                      │         │ BIO模式下，这也是最大连接数            │
├──────────────────────┼─────────┼──────────────────────────────────────┤
│ minSpareThreads      │ 10      │ 最小空闲线程数                         │
│                      │         │ 即使没请求也保持这些线程存活            │
│                      │         │ 避免请求来了还要临时创建线程的延迟      │
├──────────────────────┼─────────┼──────────────────────────────────────┤
│ maxConnections       │ 10000   │ 最大连接数（NIO模式下独立于maxThreads） │
│                      │         │ 超过此值，新连接被拒绝                 │
│                      │         │ BIO模式下此值与maxThreads相同          │
├──────────────────────┼─────────┼──────────────────────────────────────┤
│ acceptCount          │ 100     │ 当所有线程忙、连接数也满了时的等待队列  │
│                      │         │ 超过队列长度，新连接被拒绝             │
│                      │         │ * 这个队列在操作系统层面(TCP backlog)  │
├──────────────────────┼─────────┼──────────────────────────────────────┤
│ connectionTimeout    │ 60000   │ 连接超时（ms）                         │
│                      │         │ 客户端建立连接后，在此时长内没有发数据  │
│                      │         │ Tomcat主动断开连接                     │
├──────────────────────┼─────────┼──────────────────────────────────────┤
│ maxKeepAliveRequests │ 100     │ 一个keep-alive连接上最多处理多少请求   │
│                      │         │ 达到上限后，响应头自动加上Connection:close│
└──────────────────────┴─────────┴──────────────────────────────────────┘

三、线程数怎么设置？（经验公式）

没有一个绝对正确的公式，但有几个指导原则：

1. 根据服务器CPU核心数
   业务类型判断：
     - CPU密集型（加密、压缩、复杂计算）→ 线程数 ≈ CPU核心数 + 1
     - IO密集型（数据库操作、网络请求）→ 线程数 ≈ CPU核心数 × 2 或更多

2. 根据数据库连接池大小
   如果每个线程都要请求数据库，线程数应该接近或略大于数据库连接池的大小
   比如连接池配了50个连接，maxThreads 设300也没用——250个线程在排队等数据库连接

3. 根据系统资源限制
   每个线程有自己的栈空间（-Xss，默认1MB）
   200个线程 ≈ 200MB 栈空间 + 堆内存
   系统总内存必须撑得住

4. 参考值：
   4核8G  → maxThreads=200 较合适
   8核16G → maxThreads=500
   16核32G → maxThreads=800-1000

5. 压测来验证！
   理论都是参考，最终要用 JMeter / ab / wrk 压测，找到最优值

四、Spring Boot 中配置线程池
```

```yaml
# application.yml
server:
  tomcat:
    threads:
      max: 200          # maxThreads
      min-spare: 10     # minSpareThreads
    max-connections: 10000
    accept-count: 100
    connection-timeout: 20000
    keep-alive-timeout: 15000
    max-keep-alive-requests: 100
```

```java
// 或者用编程方式
@Bean
public WebServerFactoryCustomizer<TomcatServletWebServerFactory> tomcatCustomizer() {
    return factory -> {
        factory.addConnectorCustomizers(connector -> {
            Http11NioProtocol protocol = (Http11NioProtocol) connector.getProtocolHandler();
            protocol.setMaxThreads(200);
            protocol.setMinSpareThreads(10);
            protocol.setMaxConnections(10000);
            protocol.setAcceptCount(100);
            protocol.setConnectionTimeout(20000);
        });
    };
}
```

```
五、请求排队与拒绝的完整流程

客户端请求到达时，Tomcat的处理流程：

  ┌─────────────┐
  │ 请求到达     │
  └──────┬──────┘
         V
  是否有空闲Worker线程？
    │YES                │NO
    V                   V
  立即处理    maxThreads达到上限？
                │YES                │NO
                V                   V
          是否有可用连接配额？    创建新线程处理
          (maxConnections限制)
            │YES         │NO
            V            V
          入队等待      OS 的 TCP backlog 队列
          (acceptCount)   满 → 拒绝连接（TCP RST）
            │
            V
          线程空闲后从队列取出处理
            │
            V
          队列也满了？→ 拒绝连接

注意：上层（maxConnections + acceptCount）都满了之后，客户端看到的是
"连接被拒绝"或"连接超时"，不是 HTTP 错误（因为还没到 HTTP 层）。
```

# 22.Session管理机制

```
一、Session的创建

Session并不是浏览器一打开就创建的，而是"按需创建"：

request.getSession() 被调用时，Tomcat 才创建 Session。
  - request.getSession() → 如果有JSESSIONID且Session存在 → 返回已有Session
  - request.getSession() → 没有JSESSIONID或Session过期 → 创建新Session
  - request.getSession(false) → 如果没有Session → 返回null（不创建）
  - request.getSession(true) → 等同于无参版本

二、JSESSIONID —— Session的"门牌号"

Tomcat 创建的 Session 有一个唯一ID叫 JSESSIONID。
这个ID通过两种方式传递：

1. Cookie（默认方式）：
   响应头中：Set-Cookie: JSESSIONID=8A3F9B2C1D4E6F7G; Path=/oa
   浏览器下次请求时自动带上：Cookie: JSESSIONID=8A3F9B2C1D4E6F7G

2. URL Rewriting（URL重写）：
   当浏览器禁用Cookie时，Tomcat自动把JSESSIONID拼到URL上：
   /oa/user/list → /oa/user/list;jsessionid=8A3F9B2C1D4E6F7G
   通过 response.encodeURL("/oa/user/list") 实现

三、StandardManager：默认的Session管理器

StandardManager 把 Session 存在内存中（一个 ConcurrentHashMap）。

关键配置：
```

```xml
<!-- 在 Context 中配置 -->
<Manager className="org.apache.catalina.session.StandardManager"
         maxActiveSessions="-1"
         sessionIdLength="32" />
```

```
参数说明：
  - maxActiveSessions：最大活动Session数（-1表示无限制）
  - sessionIdLength：生成的JSESSIONID的长度（默认32字符）
  - pathname：Session持久化文件路径（默认：SESSIONS.ser）
  - processExpiresFrequency：Session过期检查频率（默认6，即每6次backgroundProcess检查1次）

四、Session的过期与清理

Session的"过期"是一个后台线程定期检查的。流程：

1. StandardManager 的 backgroundProcess() 方法被周期调用（约每10秒）

2. 遍历所有 Session，检查每个 Session 的 lastAccessedTime

3. 如果 (当前时间 - lastAccessedTime) > maxInactiveInterval → Session过期
   maxInactiveInterval 默认是30分钟（web.xml 中的 <session-timeout>30</session-timeout>）

4. 过期的 Session 被 remove：
   - 从 ConcurrentHashMap 中删除
   - 触发 HttpSessionListener.sessionDestroyed()
   - Session 中的属性（Attribute）如果没有其他引用，等待GC回收

五、PersistentManager：持久化Session

当Tomcat正常关闭时，StandardManager会把所有Session序列化到文件（SESSIONS.ser），
下次启动时反序列化恢复。

但对于生产环境（集群），通常用 PersistentManager：
```

```xml
<!-- File持久化：每个Session一个文件 -->
<Manager className="org.apache.catalina.session.PersistentManager"
         saveOnRestart="true"
         maxActiveSessions="-1"
         minIdleSwap="0"
         maxIdleSwap="30"
         maxIdleBackup="0">
    <Store className="org.apache.catalina.session.FileStore"
           directory="sessions" />
</Manager>

<!-- JDBC持久化：Session存数据库 -->
<Manager className="org.apache.catalina.session.PersistentManager"
         saveOnRestart="true">
    <Store className="org.apache.catalina.session.JDBCStore"
           driverName="com.mysql.cj.jdbc.Driver"
           connectionURL="jdbc:mysql://localhost:3306/tomcat_sessions"
           sessionTable="tomcat_sessions"
           sessionIdCol="session_id"
           sessionDataCol="session_data" />
</Manager>
```

```
六、Session的安全问题

1. JSESSIONID 泄露 = 等于交出账号！他人拿到你的JSESSIONID就能冒充你的身份

2. 防范措施：
   - 使用 HTTPS（防止传输过程中被窃听）
   - Cookie 上设置 HttpOnly（防止JavaScript读取Cookie，防XSS攻击）
   - Cookie 上设置 Secure（只在HTTPS连接中传输Cookie）
   - Session 固定攻击防护：登录成功后调用 request.changeSessionId() 换一个新的JSESSIONID
```

```xml
<!-- web.xml 中配置Cookie安全属性 -->
<session-config>
    <session-timeout>30</session-timeout>
    <cookie-config>
        <http-only>true</http-only>
        <secure>true</secure>
    </cookie-config>
</session-config>
```

# 23.Context配置与自动部署

```
一、Context的多种配置方式

Context（一个webapp）可以通过四种方式来配置，各有优先级：

方式一：/META-INF/context.xml（在webapp内部）
  - 放在 webapp/META-INF/context.xml
  - 随 war 包一起打包
  - 优先级：中

方式二：$CATALINA_BASE/conf/[EngineName]/[HostName]/[contextPath].xml
  - 例如：$CATALINA_BASE/conf/Catalina/localhost/oa.xml
  - 文件名就是 Context Path
  - 优先级：高

方式三：server.xml 中的 <Context> 元素（不推荐！）
  - 直接在 server.xml 的 <Host> 内写 <Context>
  - 缺点：修改后需要重启整个Tomcat
  - Tomcat 官网明确不推荐

方式四：自动部署产生
  - 把 war 包或目录丢到 webapps 目录下
  - Tomcat 自动创建一个默认 Context

二、Context 的核心属性
```

```xml
<Context path="/oa"                    <!-- Context路径（自动部署时不需要配） -->
         docBase="D:/myproject/oa"     <!-- webapp的实际文件路径 -->
         reloadable="false"            <!-- 是否监听类文件变化自动重加载 -->
         crossContext="false"          <!-- 是否允许调用其他Context的Servlet -->
         allowLinking="false"          <!-- 是否允许符号链接 -->
         sessionCookieName="JSESSIONID"<!-- Session Cookie名 -->
         sessionCookiePath="/"         <!-- Session Cookie的Path -->
         sessionTimeout="30"           <!-- Session超时（分钟） -->
         cookies="true"                <!-- 是否使用Cookie追踪Session -->
         unpackWAR="true"              <!-- 是否自动解压war包 -->
         swallowOutput="false"         <!-- 是否吞掉System.out/err输出 -->
>
    <!-- 监控WEB-INF/web.xml变化 -->
    <WatchedResource>WEB-INF/web.xml</WatchedResource>
    <!-- 监控类文件变化（reloadable=true时才有效） -->
    <WatchedResource>WEB-INF/classes</WatchedResource>
</Context>
```

```
三、autoDeploy vs deployOnStartup

这两个 Host 属性经常被混淆：

  deployOnStartup（默认true）：
    Tomcat 启动时，自动部署 appBase 下已有的所有 webapp

  autoDeploy（默认true）：
    Tomcat 运行时，持续监控 appBase 目录
    如果发现新的 war 包、新的目录、或者已有的 war 包被修改
    自动部署/重新部署该 webapp

  如果你只想在启动时部署，运行时不动：autoDeploy=false, deployOnStartup=true

四、热部署 vs 热加载

这两个概念经常混淆，但它们是不同的：

┌──────────┬──────────────────────┬────────────────────────────┐
│          │ 热部署（Hot Deploy）   │ 热加载（Hot Reload）         │
├──────────┼──────────────────────┼────────────────────────────┤
│ 触发条件 │ war包/目录新增或修改   │ WatchedResource 检测到变化  │
│          │ 检测到 Context 变化   │ 默认监控 WEB-INF/web.xml 和 │
│          │                      │ WEB-INF/classes 目录        │
├──────────┼──────────────────────┼────────────────────────────┤
│ 动作    │ 销毁整个Context并重建  │ 销毁ClassLoader并重建       │
│          │ (stop → destroy →     │ 重新加载所有类              │
│          │  create → start)     │ (仅 reload，不重新部署)     │
├──────────┼──────────────────────┼────────────────────────────┤
│ 影响    │ 整个webapp重新初始化   │ 仅类重新加载                │
│          │ Session全部丢失      │ Session保留                 │
├──────────┼──────────────────────┼────────────────────────────┤
│ 配置    │ Host的 autoDeploy     │ Context的 reloadable="true" │
└──────────┴──────────────────────┴────────────────────────────┘

五、WatchedResource

reloadable="true" 时，Tomcat 会监听 WatchedResource 指定的文件变化：

<WatchedResource>WEB-INF/web.xml</WatchedResource>
<WatchedResource>WEB-INF/classes</WatchedResource>
<WatchedResource>${catalina.base}/conf/web.xml</WatchedResource>

检测方式：每隔一段时间检查文件的 lastModified 时间是否变化。
如果有变化 → 触发 reload（销毁+重建WebappClassLoader）

六、自动部署的注意事项

1. 不要在 server.xml 中手写 Context
   除非你确信知道自己在做什么，否则让Tomcat自动部署管理Context

2. 生产环境通常关闭 autoDeploy
   避免误操作（比如误复制了一个旧war包上去导致自动部署覆盖了正确版本）

3. 大项目避免用 reloadable="true"
   频繁重加载会导致Metaspace内存碎片化，最终OutOfMemoryError: Metaspace
   开发时方便，生产环境一定关掉
```

# 24.WebSocket支持

```
Tomcat 7 开始支持 WebSocket（JSR 356），Tomcat 8+ 完美支持。

一、WebSocket 是什么？

传统 HTTP：客户端请求 → 服务器响应 → 连接断开（请求-响应模式）
WebSocket：客户端和服务器之间建立持久连接，双向实时通信

┌──────────────────────────────────────────┐
│  传统轮询：                                │
│  客户端每隔几秒发一次请求问"有新消息吗？"   │
│  缺点：浪费带宽、有延迟                    │
├──────────────────────────────────────────┤
│  WebSocket：                              │
│  一条持久连接，服务器有新数据可以主动推送   │
│  优点：实时性强、带宽省、延迟低            │
└──────────────────────────────────────────┘

二、WebSocket 协议握手

WebSocket 借 HTTP 的"东风"来建立连接：
```

```http
客户端发送（HTTP Upgrade 请求）：
GET /chat HTTP/1.1
Host: localhost:8080
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

```http
服务器返回（协议切换确认）：
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

```
三、Tomcat 中开发 WebSocket（服务端）

有两种方式：

方式一：注解方式（推荐）
```

```java
import javax.websocket.*;
import javax.websocket.server.ServerEndpoint;
import java.io.IOException;

@ServerEndpoint("/chat")
public class ChatEndpoint {

    @OnOpen
    public void onOpen(Session session) {
        System.out.println("新连接: " + session.getId());
    }

    @OnMessage
    public void onMessage(String message, Session session) {
        System.out.println("收到消息: " + message);
        // 广播给所有连接的客户端
        for (Session s : session.getOpenSessions()) {
            try {
                s.getBasicRemote().sendText("用户" + session.getId() + "说: " + message);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    @OnClose
    public void onClose(Session session) {
        System.out.println("连接关闭: " + session.getId());
    }

    @OnError
    public void onError(Session session, Throwable error) {
        error.printStackTrace();
    }
}
```

```
方式二：继承 Endpoint 类
```

```java
public class MyEndpoint extends Endpoint {
    @Override
    public void onOpen(Session session, EndpointConfig config) {
        session.addMessageHandler(new MessageHandler.Whole<String>() {
            @Override
            public void onMessage(String message) {
                // 处理消息
            }
        });
    }
}
```

```
四、Tomcat 对 WebSocket 的实现

Tomcat 内部对 WebSocket 的处理流程：

1. 请求到达 Connector，经过正常的 HTTP 解析
2. CoyoteAdapter 检测到这是一个 WebSocket 升级请求（Upgrade: websocket）
3. 执行 Upgraded 协议处理：
   - Filter 链正常执行（在握手之前）
   - 找到对应的 @ServerEndpoint
   - 发送 101 响应（协议升级）
   - 之后的数据帧走 WebSocket 协议，不再走 HTTP Processor
4. WebSocket 消息由专门的线程处理（默认是 Worker 线程池）

五、WebSocket 和 NIO 的关系

WebSocket 的"持久连接"特性天然适合 NIO：
  - 一个连接（一个SocketChannel）可以被 Selector 长期管理
  - NIO 的 poller 可以同时管理成千上万个 WebSocket 连接
  - 这和传统的HTTP请求-响应模式完全不同

这就是为什么 Tomcat 8+（默认NIO）对 WebSocket 的支持特别好。

六、Spring Boot 中的 WebSocket

Spring 对 WebSocket 做了很好的封装：
```

```java
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {
    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(new MyHandler(), "/ws")
                .setAllowedOrigins("*");
    }
}
```

```java
// 或者用更简单的 STOMP 协议（Spring推荐）
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketBrokerConfig implements WebSocketMessageBrokerConfigurer {
    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.enableSimpleBroker("/topic");
        config.setApplicationDestinationPrefixes("/app");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/websocket").withSockJS();
    }
}
```

# 25.Tomcat安全配置

```
安全问题在生产环境中是重中之重。Tomcat 虽然默认配置"能用"，但要"安全地用"，
需要做一些加固。

一、禁用 shutdown 端口

Tomcat 默认监听 8005 端口，收到 "SHUTDOWN" 字符串就会关闭。
这是个严重安全隐患！
```

```xml
<!-- 不安全 -->
<Server port="8005" shutdown="SHUTDOWN">

<!-- 安全做法：禁用 shutdown 端口 -->
<Server port="-1" shutdown="SHUTDOWN">
```

```
这样设置后，只能通过 kill 进程的方式来停止Tomcat（这正是生产环境推荐的方式）。

二、隐藏Tomcat版本信息

默认情况下，Tomcat 的错误页面会显示版本号：
  "Apache Tomcat/9.0.80"

攻击者可以根据版本号查找已知漏洞。隐藏版本信息很简单：
```

```xml
<!-- 在 Host 或 Engine 中添加 ErrorReportValve -->
<Host name="localhost" ...>
    <Valve className="org.apache.catalina.valves.ErrorReportValve"
           showReport="false"
           showServerInfo="false" />
</Host>
```

```
三、Realm 认证与授权

Realm 是 Tomcat 的"用户认证域"。支持的 Realm 类型：

1. MemoryRealm：用户信息存 XML 文件（仅开发测试用）
2. JDBCRealm：用户信息存数据库（密码明文，不安全）
3. DataSourceRealm：通过 JNDI 数据源连接数据库验证用户
4. JNDIRealm：通过 LDAP 验证用户（企业常用，对接 Active Directory）
5. UserDatabaseRealm：基于 tomcat-users.xml（Tomcat管理界面用）
6. LockOutRealm：可包装其他Realm，限制登录失败次数（防暴力破解）
7. CombinedRealm：组合多个Realm

示例：DataSourceRealm 配置
```

```xml
<!-- 1. 在 Context 中配置数据源 -->
<Resource name="jdbc/UserDB"
          auth="Container"
          type="javax.sql.DataSource"
          maxTotal="100"
          username="dbuser"
          password="dbpass"
          driverClassName="com.mysql.cj.jdbc.Driver"
          url="jdbc:mysql://localhost:3306/userdb" />

<!-- 2. 在 Engine 或 Host 中配置 Realm -->
<Realm className="org.apache.catalina.realm.DataSourceRealm"
       dataSourceName="jdbc/UserDB"
       userTable="users"
       userNameCol="username"
       userCredCol="password"
       userRoleTable="user_roles"
       roleNameCol="role_name" />
```

```
四、SSL/TLS 配置

为了保护数据传输安全（特别是登录密码），生产环境必须用HTTPS。

步骤1：生成密钥库（自签名证书示例）
```

```bash
keytool -genkey -alias tomcat -keyalg RSA \
        -keystore D:/keystore/tomcat.keystore \
        -validity 3650
```

```
步骤2：在 server.xml 中配置 SSL Connector
```

```xml
<Connector port="8443"
           protocol="org.apache.coyote.http11.Http11NioProtocol"
           maxThreads="150"
           SSLEnabled="true"
           scheme="https"
           secure="true"
           clientAuth="false"
           sslProtocol="TLS">
    <SSLHostConfig>
        <Certificate certificateKeystoreFile="conf/tomcat.keystore"
                     certificateKeystorePassword="changeit"
                     type="RSA" />
    </SSLHostConfig>
</Connector>
```

```
Tomcat 8.5+ 使用 <SSLHostConfig>，之前版本直接设置 keystoreFile 属性。

步骤3：强制 HTTP 跳转到 HTTPS
```

```xml
<!-- 在 web.xml 中配置 -->
<security-constraint>
    <web-resource-collection>
        <web-resource-name>Entire Application</web-resource-name>
        <url-pattern>/*</url-pattern>
    </web-resource-collection>
    <user-data-constraint>
        <transport-guarantee>CONFIDENTIAL</transport-guarantee>
    </user-data-constraint>
</security-constraint>
```

```
五、Security Constraint（安全约束）

web.xml 中可以配置细粒度的安全约束：
```

```xml
<security-constraint>
    <!-- 哪些资源受保护 -->
    <web-resource-collection>
        <web-resource-name>Admin Pages</web-resource-name>
        <url-pattern>/admin/*</url-pattern>
        <http-method>GET</http-method>
        <http-method>POST</http-method>
    </web-resource-collection>

    <!-- 哪些角色可以访问 -->
    <auth-constraint>
        <role-name>admin</role-name>
    </auth-constraint>
</security-constraint>

<!-- 定义登录方式 -->
<login-config>
    <auth-method>BASIC</auth-method>  <!-- BASIC / FORM / DIGEST / CLIENT-CERT -->
    <realm-name>User Database</realm-name>
</login-config>

<!-- 定义安全角色 -->
<security-role>
    <role-name>admin</role-name>
</security-role>
```

```
六、其他安全加固建议

1. 限制 HTTP 方法
```

```xml
<!-- 禁止 TRACE 方法（防跨站追踪攻击） -->
<security-constraint>
    <web-resource-collection>
        <web-resource-name>Restricted Methods</web-resource-name>
        <url-pattern>/*</url-pattern>
        <http-method>TRACE</http-method>
        <http-method>PUT</http-method>
        <http-method>DELETE</http-method>
        <http-method>OPTIONS</http-method>
    </web-resource-collection>
    <auth-constraint />
</security-constraint>
```

```
2. 删除默认应用
   - 删除 webapps 下的 ROOT、docs、examples、manager、host-manager
   - 生产服务器不需要这些

3. 用非 root 用户运行 Tomcat
   - Linux 下不要用 root 用户启动 Tomcat（万一被入侵，攻击者没有root权限）
   - 创建一个专门的 tomcat 用户

4. 设置 Cookie 安全属性（在 context.xml 中）
```

```xml
<CookieProcessor className="org.apache.tomcat.util.http.Rfc6265CookieProcessor"
                 sameSiteCookies="strict" />
```

```
5. 文件上传限制
```

```xml
<Connector maxPostSize="10485760" />  <!-- 限制POST最大10MB -->
```

# 26.Tomcat日志系统（JULI）

```
一、为什么Tomcat自创了JULI？

Tomcat 没有直接用 java.util.logging（JUL），而是创建了 JULI（Java Utility Logging Implementation）。
原因：

1. JUL 不支持"每个webapp独立的日志配置"
   - 所有webapp共享一个JUL配置
   - Tomcat需要每个Context有独立的日志控制

2. JUL 的 ClassLoader 问题
   - JUL 用单例模式，LogManager 是全局的
   - 在 Tomcat 的 ClassLoader 体系下，会导致内存泄漏

3. JULI 的增强
   - 支持 ${catalina.base} 这种变量替换
   - 支持按 webapp 隔离日志配置
   - 更好的 ClassLoader 感知

JULI的包路径：org.apache.juli（不在 java.util.logging 下，不冲突）

二、核心配置文件：logging.properties

$CATALINA_BASE/conf/logging.properties 是 Tomcat 全局日志配置：
```

```properties
# 日志处理器（Handler）
handlers = 1catalina.org.apache.juli.FileHandler, \
           2localhost.org.apache.juli.FileHandler, \
           3manager.org.apache.juli.FileHandler, \
           java.util.logging.ConsoleHandler

# 根日志级别
.level = INFO

# FileHandler配置
1catalina.org.apache.juli.FileHandler.level = FINE
1catalina.org.apache.juli.FileHandler.directory = ${catalina.base}/logs
1catalina.org.apache.juli.FileHandler.prefix = catalina.
1catalina.org.apache.juli.FileHandler.maxDays = 90
1catalina.org.apache.juli.FileHandler.encoding = UTF-8

# ConsoleHandler配置
java.util.logging.ConsoleHandler.level = FINE
java.util.logging.ConsoleHandler.formatter = org.apache.juli.OneLineFormatter
java.util.logging.ConsoleHandler.encoding = UTF-8

# 针对特定包设置日志级别
org.apache.catalina.core.ContainerBase.[Catalina].[localhost].level = INFO
org.apache.catalina.core.ContainerBase.[Catalina].[localhost].[/manager].level = INFO
org.apache.catalina.core.ContainerBase.[Catalina].[localhost].[/host-manager].level = INFO
```

```
三、JULI 的几个核心 Handler

1. FileHandler：按天滚动
   - prefix.prefix.YYYY-MM-DD.log 格式
   - maxDays 控制保留天数（超过自动删除）

2. AsyncFileHandler：异步写入
   - 日志先写入内存队列，再异步刷盘
   - 性能更好，但崩溃时可能丢失最后几条日志

3. ConsoleHandler：输出到控制台（System.err）

四、Access Log（访问日志）

这是另一个独立的日志系统——AccessLogValve，不经过JULI：
```

```xml
<Valve className="org.apache.catalina.valves.AccessLogValve"
       directory="logs"
       prefix="localhost_access_log"
       suffix=".txt"
       pattern="%h %l %u %t &quot;%r&quot; %s %b %D"
       renameOnRotate="true"
       maxDays="30" />
```

```
pattern 中的常用占位符：

┌──────────┬──────────────────────────────────────────┐
│ 占位符    │ 含义                                      │
├──────────┼──────────────────────────────────────────┤
│ %h        │ 客户端IP地址                              │
│ %l        │ 远程用户名（通常为-）                     │
│ %u        │ 认证用户名                                │
│ %t        │ 请求时间                                  │
│ %r        │ 请求行（GET /index.html HTTP/1.1）       │
│ %s        │ HTTP状态码                                │
│ %b        │ 响应体字节数（不含响应头）                 │
│ %D        │ 请求处理耗时（毫秒）                      │
│ %T        │ 请求处理耗时（秒）                        │
│ %{xxx}i   │ 请求头 xxx 的值                          │
│ %{xxx}o   │ 响应头 xxx 的值                          │
│ %I        │ 当前线程名                                │
│ %S        │ 用户Session ID                           │
│ %a        │ 客户端IP（同%h）                          │
│ %A        │ 服务器IP                                  │
│ %B        │ 响应体字节数（不含响应头，0显示为0）       │
│ %m        │ 请求方法                                  │
│ %q        │ 查询参数（带?前缀）                       │
│ %H        │ 请求协议（HTTP/1.1）                      │
└──────────┴──────────────────────────────────────────┘

常用组合：
  pattern="%h %l %u [%t] &quot;%r&quot; %s %b %Dms"
  输出示例：127.0.0.1 - - [15/May/2026:10:30:45 +0800] "GET /oa/index.html HTTP/1.1" 200 1234 5ms

五、每个webapp独立的日志配置

JULI 支持每个webapp有自己的 logging.properties，放在 /WEB-INF/classes/logging.properties
这样每个webapp可以独立控制自己的日志级别和输出位置。

六、日志乱码问题

最常遇到的日志乱码问题及解决方案：

1. 控制台日志乱码：
   logging.properties 中：java.util.logging.ConsoleHandler.encoding = UTF-8

2. FileHandler 日志乱码：
   logging.properties 中：xxx.FileHandler.encoding = UTF-8

3. Access Log 乱码：
   AccessLogValve 的 encoding 属性（需要 Tomcat 8.5+）

4. Windows 命令行乱码：
   要确认 cmd 的编码（chcp 65001 切换为UTF-8，chcp 936 切换为GBK）
   确认 Tomcat 日志编码和命令行编码一致

5. IDEA 控制台乱码：
   IDEA 的 Tomcat Local Run Configuration → Server → VM options
   添加：-Dfile.encoding=UTF-8
   同时设置 IDEA Console 的编码为 UTF-8
```

# 27.性能调优实战

```
Tomcat 性能调优是一个系统工程，从JVM到Connector到应用代码，每层都能优化。

一、JVM层优化

在 catalina.bat / catalina.sh 中设置 JAVA_OPTS：
```

```bash
# 典型的 JVM 配置
JAVA_OPTS="-server
  -Xms2048m -Xmx2048m           # 堆内存初始=最大，避免动态调整的开销
  -Xmn512m                       # 新生代大小
  -Xss256k                       # 线程栈大小（默认1MB，可以调小）
  -XX:+UseG1GC                   # ★ 使用G1垃圾回收器（低延迟）
  -XX:MaxGCPauseMillis=200       # G1最大停顿时间目标
  -XX:+DisableExplicitGC         # 禁止System.gc()触发Full GC
  -XX:+PrintGCDetails            # 打印GC日志
  -XX:+PrintGCDateStamps
  -Xloggc:${CATALINA_HOME}/logs/gc.log
  -Djava.security.egd=file:/dev/./urandom  # Linux熵值优化"
```

```
二、Connector 性能调优
```

```xml
<Connector port="8080"
           protocol="org.apache.coyote.http11.Http11NioProtocol"
           maxThreads="500"               # 工作线程数
           minSpareThreads="50"           # 最少空闲线程
           maxConnections="10000"         # 最大连接数
           acceptCount="200"              # 等待队列
           connectionTimeout="20000"      # 连接超时
           enableLookups="false"          # ★ 禁用DNS反向解析（性能提升明显！）
           compression="on"               # 开启压缩
           compressionMinSize="2048"      # 大于2KB才压缩
           compressibleMimeType="text/html,text/xml,text/plain,text/css,
                                 text/javascript,application/javascript,
                                 application/json"
           noCompressionUserAgents="gozilla, traviata"
           disableUploadTimeout="true"    # 上传时使用更长的超时
           URIEncoding="UTF-8"
           useBodyEncodingForURI="true"
           server="Unknown"               # ★ 隐藏Server响应头（安全+省字节）
/>
```

```
三、Host/Context 层优化
```

```xml
<Host name="localhost"
      appBase="webapps"
      unpackWARs="true"
      autoDeploy="false"         # ★ 生产环境关闭自动部署
      deployOnStartup="true"
      startStopThreads="4">     # 多线程启动/停止Context

    <!-- 静态资源使用默认Servlet处理 -->
    <Valve className="org.apache.catalina.valves.AccessLogValve"
           directory="logs"
           prefix="localhost_access_log"
           pattern="%h %l %u %t &quot;%r&quot; %s %b %D"
           buffered="true"      # ★ 启用异步写入（性能提升）
           maxDays="30" />
</Host>
```

```
四、静态资源优化

1. 配置静态资源缓存（在 web.xml 中的 DefaultServlet）：
```

```xml
<servlet>
    <servlet-name>default</servlet-name>
    <servlet-class>org.apache.catalina.servlets.DefaultServlet</servlet-class>
    <init-param>
        <param-name>cacheMaxSize</param-name>
        <param-value>10240</param-value>     <!-- 缓存大小（KB） -->
    </init-param>
    <init-param>
        <param-name>cacheTTL</param-name>
        <param-value>5000</param-value>      <!-- 缓存有效期（ms） -->
    </init-param>
    <init-param>
        <param-name>listings</param-name>
        <param-value>false</param-value>     <!-- ★ 禁用目录列表 -->
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>
```

```
2. 使用 Nginx 做反向代理处理静态资源：
   Nginx 处理静态文件（HTML/CSS/JS/图片）比 Tomcat 高效得多
   让 Tomcat 只干 Servlet 的活

五、连接池优化

如果你用 Tomcat 自带的 JDBC 连接池：
```

```xml
<Resource name="jdbc/MyDB"
          auth="Container"
          type="javax.sql.DataSource"
          factory="org.apache.tomcat.jdbc.pool.DataSourceFactory"
          maxActive="100"            # 最大活跃连接
          maxIdle="30"               # 最大空闲连接
          minIdle="10"               # 最小空闲连接
          initialSize="10"
          maxWait="10000"            # 等待最大时间（ms）
          testWhileIdle="true"       # 空闲时测试连接
          testOnBorrow="false"       # 借用时测试（影响性能）
          validationQuery="SELECT 1"
          timeBetweenEvictionRunsMillis="30000"
          removeAbandoned="true"     # 移除泄露的连接
          removeAbandonedTimeout="60" />
```

```
六、应用层面的优化

1. 减少 web.xml 的复杂性
   - 设置 metadata-complete="true" 跳过注解扫描
   - 不要配置几百个不需要的 Listener 和 Servlet

2. Filter 链精简
   - 每个 Filter 都会在请求链中增加一层
   - 不必要的 Filter 别配（只在需要的 URL pattern 上配）

3. Session 优化
   - 不要在 Session 中存放大量数据（大对象存缓存，Session只放ID）
   - 及时清理过期 Session（设置合理的 session-timeout）

4. 日志级别
   - 生产环境用 INFO 或 WARN，不要用 DEBUG
   - 控制台日志关掉（ConsoleHandler.level = OFF）
```

# 28.嵌入式Tomcat与Spring Boot

```
一、传统部署 vs 嵌入式启动

传统部署模式（你之前学的）：
  ┌──────────────┐     ┌──────────────┐
  │ 安装Tomcat   │     │ 打war包      │
  │ (解压即用)   │     │ 复制到       │
  │              │ ←── │ webapps目录  │
  └──────────────┘     └──────────────┘
    启动Tomcat → 自动部署war → 运行

嵌入式模式（Spring Boot的方式）：
  ┌──────────────────────────────────────┐
  │  java -jar myapp.jar                 │
  │  JVM 直接启动                         │
  │  应用代码中包含内嵌的Tomcat依赖         │
  │  main()方法中启动Tomcat              │
  └──────────────────────────────────────┘
    不需要预先安装Tomcat，一个jar包搞定一切

二、Spring Boot 是如何嵌入 Tomcat 的？
```

```java
// Spring Boot 2.x 的内嵌 Tomcat 启动流程（简化）

// 1. SpringApplication.run() 的最终会执行
//    ServletWebServerApplicationContext.onRefresh()
//    调用 createWebServer()

// 2. createWebServer() 获取 TomcatServletWebServerFactory
//    这是一个自动配置的 Bean

// 3. TomcatServletWebServerFactory.getWebServer() 做了什么：
public WebServer getWebServer(ServletContextInitializer... initializers) {
    // 3a. 创建 org.apache.catalina.startup.Tomcat 实例
    Tomcat tomcat = new Tomcat();

    // 3b. 设置基础目录（通常是临时目录）
    File baseDir = createTempDir("tomcat");
    tomcat.setBaseDir(baseDir.getAbsolutePath());

    // 3c. 创建 Connector
    //    默认监听 8080 端口（从 server.port 配置读取）
    Connector connector = new Connector("HTTP/1.1");
    connector.setPort(port);
    tomcat.getService().addConnector(connector);

    // 3d. 创建 Context（不用web.xml，用编程方式配置）
    Context context = tomcat.addContext("", baseDir.getAbsolutePath());

    // 3e. 把 Spring 的 Servlet/Filter/Listener 注册到 Context
    //    包括 DispatcherServlet（Spring MVC的核心Servlet）
    TomcatStarter starter = new TomcatStarter(...);
    context.addServletContainerInitializer(starter, ...);

    // 3f. 启动 Tomcat
    tomcat.start();

    // 3g. 返回 TomcatWebServer（实现了 WebServer 接口）
    return new TomcatWebServer(tomcat, getPort() >= 0);
}
```

```
三、关键区别

┌──────────────┬──────────────────────┬────────────────────────────┐
│              │ 传统部署              │ 嵌入式（Spring Boot）        │
├──────────────┼──────────────────────┼────────────────────────────┤
│ web.xml      │ 需要                  │ 不需要，用代码配置           │
│ 启动方式     │ startup.bat/脚本      │ main()方法直接启动           │
│ 部署方式     │ 打war包放到webapps    │ 打jar包，内嵌Tomcat          │
│ Tomcat版本   │ 独立安装              │ Maven依赖坐标控制版本        │
│ 多应用隔离   │ 天然支持（多个Context）│ 每个jar独立JVM进程           │
│ 热部署       │ 复制war包即可         │ 需用DevTools或JRebel        │
└──────────────┴──────────────────────┴────────────────────────────┘

四、Spring Boot 中自定义 Tomcat 参数
```

```yaml
# application.yml
server:
  port: 8080
  tomcat:
    uri-encoding: UTF-8
    max-threads: 200
    min-spare-threads: 10
    max-connections: 10000
    accept-count: 100
    connection-timeout: 20000
    accesslog:
      enabled: true
      directory: logs
      pattern: "%h %l %u %t &quot;%r&quot; %s %b %D"
      prefix: access_log
      suffix: .log
      max-days: 30
```

```java
// 编程方式深度定制
@Bean
public TomcatServletWebServerFactory tomcatFactory() {
    TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory();

    // 添加自定义 Connector
    factory.addAdditionalTomcatConnectors(createHttpsConnector());

    // 自定义 Context
    factory.addContextCustomizers(context -> {
        context.setSessionTimeout(30);
        context.setCookies(true);
    });

    // 自定义 Connector
    factory.addConnectorCustomizers(connector -> {
        Http11NioProtocol protocol = (Http11NioProtocol) connector.getProtocolHandler();
        protocol.setMaxThreads(500);
        protocol.setConnectionTimeout(30000);
    });

    return factory;
}
```

```
五、嵌入式 Tomcat 的优势

1. 部署极简：一个jar包，一条命令，到处运行
2. 版本可控：Maven/Gradle 精确控制Tomcat版本，不依赖服务器环境
3. 配置集中：所有配置在 application.yml 中，不需要分开管理 server.xml
4. 微服务友好：每个服务独立进程，独立JVM，天然隔离
5. 云原生兼容：容器化部署（Docker）时优势明显

六、嵌入式不是万能的

1. 静态资源性能：没有 Nginx 等反向代理时，静态资源都走Tomcat，性能较差
   - 解决方案：生产环境前面加 Nginx 处理静态资源

2. 多应用管理：如果一台机器要跑10个webapp，嵌入式需要启10个JVM进程，内存开销大
   - 传统Tomcat一个JVM跑10个Context，内存省得多

3. 管理界面缺失：没有 manager 这种可在线管理war包的界面
   - 解决方案：用 Spring Boot Actuator 监控

# 29.Tomcat集群与负载均衡

```
当单台Tomcat扛不住流量时，就需要横向扩展——用多台Tomcat组成集群。

一、集群拓扑

常见的集群架构：

                  ┌──────────┐
                  │  Nginx   │  (反向代理 + 负载均衡)
                  │  :80     │
                  └────┬─────┘
           ┌───────────┼───────────┐
           V           V           V
    ┌──────────┐ ┌──────────┐ ┌──────────┐
    │ Tomcat 1 │ │ Tomcat 2 │ │ Tomcat 3 │
    │  :8080   │ │  :8080   │ │  :8080   │
    └──────────┘ └──────────┘ └──────────┘

二、Session 复制（集群的核心问题）

集群中最棘手的不是负载均衡，而是 Session 共享。
用户在 Tomcat 1 上登录，下次请求被 Nginx 转发到 Tomcat 2，Tomcat 2 没有这个 Session，
用户就被踢出去要求重新登录了。

三种解决方案：

方案一：Session 复制（Tomcat 自己支持）
  每台 Tomcat 之间互相复制 Session，任何一台都有全量 Session
  优点：内置支持，不依赖外部组件
  缺点：网络开销大，不适合3台以上的集群

方案二：Session 粘性（Sticky Session，Nginx ip_hash）
  同一IP的请求始终转发到同一台 Tomcat
  优点：实现简单
  缺点：某台Tomcat宕机，上面的Session全丢；负载可能不均衡

方案三：集中式 Session 存储（推荐！）
  所有 Tomcat 把 Session 存到同一个 Redis/Memcached 中
  优点：性能好，扩展性强，宕机不影响
  缺点：需要引入额外组件

三、Tomcat 内置集群配置

在 server.xml 中配置 Cluster 组件：
```

```xml
<!-- 在 Engine 或 Host 中配置 -->
<Cluster className="org.apache.catalina.ha.tcp.SimpleTcpCluster"
         channelSendOptions="8">

    <!-- Session管理 -->
    <Manager className="org.apache.catalina.ha.session.DeltaManager"
             expireSessionsOnShutdown="false"
             notifyListenersOnReplication="true" />

    <!-- 信道：节点之间通信 -->
    <Channel className="org.apache.catalina.tribes.group.GroupChannel">
        <!-- 成员发现（组播方式） -->
        <Membership className="org.apache.catalina.tribes.membership.McastService"
                    address="228.0.0.4"
                    port="45564"
                    frequency="500"
                    dropTime="3000" />
        <!-- 消息发送 -->
        <Sender className="org.apache.catalina.tribes.transport.ReplicationTransmitter">
            <Transport className="org.apache.catalina.tribes.transport.nio.PooledParallelSender" />
        </Sender>
        <!-- 消息接收 -->
        <Receiver className="org.apache.catalina.tribes.transport.nio.NioReceiver"
                  address="auto"
                  port="4000"
                  autoBind="100"
                  selectorTimeout="5000"
                  maxThreads="6" />
        <!-- 拦截器链 -->
        <Interceptor className="org.apache.catalina.tribes.group.interceptors.TcpFailureDetector" />
        <Interceptor className="org.apache.catalina.tribes.group.interceptors.MessageDispatchInterceptor" />
    </Channel>

    <!-- 资源复制 -->
    <Valve className="org.apache.catalina.ha.tcp.ReplicationValve"
           filter="" />

    <Deployer className="org.apache.catalina.ha.deploy.FarmWarDeployer"
              tempDir="/tmp/war-temp/"
              deployDir="/tmp/war-deploy/"
              watchDir="/tmp/war-listen/"
              watchEnabled="false" />

    <ClusterListener className="org.apache.catalina.ha.session.ClusterSessionListener" />
</Cluster>
```

```
同时，你的 web.xml 中需要加上 <distributable/> 标签，告诉 Tomcat 这个应用支持分布式：
```

```xml
<web-app ...>
    <distributable/>
</web-app>
```

```
四、Nginx 反向代理 + 负载均衡配置

生产环境中最常用 Nginx + Tomcat 的组合：
```

```nginx
# nginx.conf
upstream tomcat_cluster {
    # 负载均衡策略
    # ip_hash;               # Session粘性
    # least_conn;            # 最少连接
    # 默认是轮询（round-robin）

    server 192.168.1.101:8080 weight=3;  # weight=3 表示承担3/5的流量
    server 192.168.1.102:8080 weight=2;

    keepalive 32;  # 与后端Tomcat保持的长连接数
}

server {
    listen 80;
    server_name www.example.com;

    # 静态资源由 Nginx 直接处理
    location /static/ {
        root /var/www;
        expires 30d;
    }

    # 动态请求转发到 Tomcat
    location / {
        proxy_pass http://tomcat_cluster;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # 超时设置
        proxy_connect_timeout 30s;
        proxy_read_timeout 60s;
        proxy_send_timeout 60s;
    }
}
```

```
五、集群注意事项

1. 时间同步：
   集群中所有Tomcat的时间必须一致（用 NTP 同步）
   否则 Session 过期判断会出问题

2. 静态资源同步：
   如果静态资源也放在 Tomcat 中，需要保证所有Tomcat的静态资源一致
   可以用 rsync 或 共享文件系统（NFS）

3. 数据库连接池：
   集群中每个Tomcat实例独立连接数据库
   总连接数 = 实例数 × 每实例连接池大小
   确保数据库能承受这个连接数

4. 灰度发布：
   先更新一台Tomcat，验证没问题后再更新其余的
   不要让所有Tomcat同时不可用
```

# 30.Tomcat 9→10 迁移：javax → jakarta

```
这是Tomcat历史上最大的一次不兼容变更。如果你在维护老项目，这一节必须看。

一、为什么改名？

整个故事是这样的：

JavaEE 最初由 Sun 公司主导，所有 API 都在 javax.* 包名下（javax.servlet、javax.persistence等）

2017年 Oracle 把 JavaEE 捐赠给了 Eclipse 基金会（因为Oracle自己不想维护了）。
Eclipse 基金会接手后，给它改名叫 Jakarta EE。

但是！Oracle 保留了 javax 商标的使用权，不允许 Eclipse 基金会在新的规范中
使用 javax 命名空间。

Eclipse 基金会没办法，只能把所有 javax.* 包名改成 jakarta.*。

这就是 Tomcat 10 中 javax.servlet → jakarta.servlet 的由来。
这不只是重命名，而是法律原因。

二、具体变更了什么？
```

```java
// ============ 旧代码（Tomcat 9） ============
import javax.servlet.Servlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.annotation.WebServlet;
import javax.servlet.Filter;

// ============ 新代码（Tomcat 10+） ============
import jakarta.servlet.Servlet;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import jakarta.servlet.annotation.WebServlet;
import jakarta.servlet.Filter;
```

```xml
<!-- Maven 依赖也变了： -->
<!-- Tomcat 9 及之前 -->
<dependency>
    <groupId>javax.servlet</groupId>
    <artifactId>javax.servlet-api</artifactId>
    <version>4.0.1</version>
    <scope>provided</scope>
</dependency>

<!-- Tomcat 10+ -->
<dependency>
    <groupId>jakarta.servlet</groupId>
    <artifactId>jakarta.servlet-api</artifactId>
    <version>6.0.0</version>
    <scope>provided</scope>
</dependency>
```

```
三、Spring Boot 的对应关系

┌──────────────┬──────────────────────┬───────────────────┐
│ Spring Boot   │ Tomcat版本            │ Servlet API        │
├──────────────┼──────────────────────┼───────────────────┤
│ 2.x          │ Tomcat 9.x           │ javax.servlet 4.0 │
│ 3.x          │ Tomcat 10.x          │ jakarta.servlet 5.0/6.0 │
└──────────────┴──────────────────────┴───────────────────┘

如果你要把 Spring Boot 2.x 升级到 3.x：
  - 所有 import javax.servlet.* → import jakarta.servlet.*
  - 所有 import javax.persistence.* → import jakarta.persistence.*
  - 所有 import javax.annotation.* → import jakarta.annotation.*

四、迁移工具

Apache 官方提供了迁移工具：
  https://github.com/apache/tomcat-jakartaee-migration

本质上就是一个字节码级别的批量替换工具：
  - 把 .class 文件中的 javax/ 字符串替换为 jakarta/
  - 支持 jar 包和 war 包的批量转换

五、如果你还没准备好升级

有些老项目的依赖（比如依赖某个老的开源库）还没有发布 jakarta 版本。
这时候怎么办？

选项1：继续用 Tomcat 9
  - Tomcat 9 还会维护一段时间
  - 它的 Servlet 4.0 (javax) 功能完全够用

选项2：用 Tomcat 的"兼容模式"
  - Tomcat 10 自带一个 migration 工具
  - 可以把 javax 的 webapp 自动转换为 jakarta
  - 但某些复杂场景可能不兼容

选项3：等所有依赖都支持 jakarta 后再升级
  - 目前主流框架（Spring、Hibernate、MyBatis）都已支持
  - 如果只是一些小众库不兼容，可以考虑替换
```

# 31.Tomcat设计模式分析

```
Tomcat 的设计质量在开源Java项目中属于顶级水平。它巧妙地运用了大量设计模式。
理解这些模式，不仅帮你读懂Tomcat源码，更让你在日常开发中\"有招可用\"。

一、模板方法模式（Template Method）

在哪用：LifecycleBase

我们已经讲过 LifecycleBase，它是模板方法模式的教科书级示例。
LifecycleBase 定义了 init/start/stop/destroy 的标准流程（模板），
具体的子类实现 initInternal/startInternal/stopInternal/destroyInternal（具体步骤）。

LifecycleBase.init() 的模板：
  1. 状态检查
  2. setState(INITIALIZING)
  3. initInternal()    ← 子类自己实现
  4. setState(INITIALIZED)
```

```java
// 模板方法在Tomcat中的另一个例子
public abstract class ValveBase implements Valve {
    @Override
    public void invoke(Request request, Response response) {
        // 模板方法：先做通用检查
        if (isAsyncSupported()) {
            request.setAsyncSupported(true);
        }
        // 再调用子类的核心逻辑（也是模板）
        invokeInternal(request, response);
    }
    protected abstract void invokeInternal(Request request, Response response);
}
```

```
二、责任链模式（Chain of Responsibility）

在哪用：Pipeline-Valve

这是我们在第12节专门讲的内容。每个容器的 Pipeline 中串联多个 Valve，
请求依次穿过每个 Valve，每个 Valve 可以：
  - 处理请求并继续传递（调用 getNext().invoke()）
  - 拦截请求不往下传（直接返回）

这和 Servlet 中的 Filter 链本质上是同一种模式。

三、观察者模式（Observer）

在哪用：LifecycleListener、ContainerListener、PropertyChangeListener

Tomcat 中事件机制无处不在：
  - LifecycleEvent：组件状态变化时通知所有 LifecycleListener
  - ContainerEvent：容器添加/删除子容器时通知 ContainerListener
  - PropertyChangeEvent：属性值变化时通知 PropertyChangeListener

四、外观模式（Facade）

在哪用：RequestFacade、ResponseFacade

你在 Servlet 中拿到的 HttpServletRequest 并不是 Tomcat 内部的 Request 对象，
而是 RequestFacade。它把内部复杂的方法\"藏起来\"，只暴露 Servlet 规范允许的方法。
```

```java
// RequestFacade 的设计意图
public class RequestFacade implements HttpServletRequest {
    // ★ 持有真正的 Request 对象
    protected Request request;

    // 暴露给 Servlet 的安全方法
    @Override
    public String getParameter(String name) {
        return request.getParameter(name);
    }

    // 内部方法（如 recycle()）不暴露给 Servlet
    // 防止你在 Servlet 中不小心调用导致意外行为
}
```

```
为什么要用门面模式？
1. 安全：Servlet 开发者看不到也不应该调用 Tomcat 内部方法
2. 防止滥用：比如 recycle() 这个方法会重置所有请求属性，
   如果在 Servlet 中调了会导致灾难性后果
3. 版本兼容：修改内部实现不影响外部 API

五、命令模式（Command）

在哪用：SocketProcessor

SocketProcessor 把\"处理一个连接\"这个操作封装成一个对象，交给线程池执行。
这就是命令模式的经典用法：把操作封装为可调度的任务。

六、策略模式（Strategy）

在哪用：ProtocolHandler 的选择

Connector 可以通过配置选择不同的 ProtocolHandler：
  - Http11NioProtocol（NIO策略）
  - Http11AprProtocol（APR策略）

Tomcat 在运行时根据配置选择合适的策略（算法族），且可以随时切换。

七、单例模式（Singleton）—— 但是单例 + 多线程！

在哪用：Servlet 实例

你在 web.xml 中配置一个 Servlet，Tomcat 默认只创建一个实例（单例）。
但是！这个单例被大量线程并发访问。
所以 Servlet 是\"无状态单例\"——不保存请求级别的成员变量。

```java
@WebServlet("/user")
public class UserServlet extends HttpServlet {
    // ★ 危险！这个成员变量会被所有线程共享
    // private String currentUser;

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) {
        // ★ 正确做法：用局部变量
        String currentUser = req.getParameter("username");
        // 局部变量在栈上，每个线程有自己的栈，线程安全
    }
}
```


```
八、适配器模式（Adapter）

在哪用：CoyoteAdapter

CoyoteAdapter 把一个接口（Coyote的Request/Response）适配为另一个接口
（Catalina的Request/Response），让不兼容的 Coyote 和 Catalina 能协同工作。

九、组合模式（Composite）

在哪用：Container 层次结构

Engine → Host → Context → Wrapper 这四级容器就是典型的组合模式。
父容器和子容器有统一的 Container 接口，对上层代码来说，
不需要关心当前操作的是哪个层级的容器。

十、享元模式（Flyweight）

在哪用：ByteChunk、CharChunk、MessageBytes

Tomcat 处理HTTP请求时需要频繁地操作字节和字符，为了避免每次 new String 和 new byte[]，
Tomcat 大量使用 Chunk 对象（享元），复用缓冲区，减少 GC 压力。

总结：
Tomcat 让你看到了设计模式在真实大型项目中的应用。下次学习设计模式时，
可以打开 Tomcat 源码，对照着看，比看书本能学到更多。
```

# 32.常见问题排查

```
一、端口占用（最最常见）

现象：启动Tomcat时报错 \"Address already in use: bind\"

原因：8080端口（或其他Connector端口）已经被占用

排查步骤：
```

```bash
# Windows
netstat -ano | findstr 8080
# 找到占用端口的PID，记下来
taskkill /PID <PID> /F

# Linux
lsof -i:8080
# 或者
netstat -tlnp | grep 8080
kill -9 <PID>
```

```
常见占用场景：
  - 上一个Tomcat没有正常关闭（双击了startup.bat两次）
  - IDEA 中已启动了一个Tomcat，又在外面启动了一个
  - 其他程序（Nginx、**DB等）占用了同样的端口
  - 端口号配错了（默认8080）

二、OutOfMemoryError

1. Java heap space（堆内存溢出）
   原因：堆内存不够，通常是大对象过多或者内存泄漏
   解决：
     - 增大堆内存：-Xmx2048m
     - 用 jmap 分析堆：jmap -histo <PID> | head -20
     - 用 MAT 分析 heap dump 文件

2. PermGen space（JDK7及以前）
   原因：永久代被类的元数据占满（如频繁热部署太多类）
   解决：
     - JDK7: -XX:MaxPermSize=256m
     - 或者升级到JDK8+（类信息移到Metaspace，自动扩容）

3. Metaspace（JDK8+）
   原因：频繁热部署加载了大量类，Metaspace满了
   解决：
     - 增大Metaspace：-XX:MaxMetaspaceSize=256m
     - 减少热部署频率
     - 检查是否有 ClassLoader 泄漏

三、启动慢（Linux 最常遇到）

现象：Tomcat 卡在 \"Deployment of web application\" 很久

最常见原因及解决：
```

```bash
# 1. SecureRandom 熵值不足（最常见的罪魁祸首！）
#    在 JAVA_OPTS 中加入：
JAVA_OPTS="-Djava.security.egd=file:/dev/./urandom"

# 2. DNS 解析慢
#    在 /etc/hosts 中确保：
127.0.0.1 localhost localhost.localdomain <你的主机名>

# 3. Jar 扫描慢
#    在 context.xml 中配置跳过不必要的Jar扫描（参见第18节）
```

```
四、中文乱码问题汇总

乱码是 JavaWeb 中最烦人的问题之一，涉及好多层：

┌──────────────┬────────────────────────────────────┐
│ 哪里乱码     │ 解决方案                             │
├──────────────┼────────────────────────────────────┤
│ IDEA控制台   │ VM options: -Dfile.encoding=UTF-8   │
│              │ IDEA Settings → File Encodings     │
│              │ Project Encoding → UTF-8            │
├──────────────┼────────────────────────────────────┤
│ server.xml   │ <Connector URIEncoding=\"UTF-8\"     │
│              │          useBodyEncodingForURI=\"true\"│
├──────────────┼────────────────────────────────────┤
│ JSP 页面     │ <%@ page contentType=\"text/html;     │
│              │           charset=UTF-8\" %>         │
├──────────────┼────────────────────────────────────┤
│ Servlet      │ request.setCharacterEncoding(\"UTF-8\")│
│              │ response.setCharacterEncoding(\"UTF-8\")│
├──────────────┼────────────────────────────────────┤
│ 数据库       │ jdbc:mysql://host/db?               │
│              │   characterEncoding=utf8            │
├──────────────┼────────────────────────────────────┤
│ logging      │ xxx.FileHandler.encoding = UTF-8    │
│ .properties  │                                     │
└──────────────┴────────────────────────────────────┘

五、HTTP 请求 404（路径匹配问题）

最常见的几个原因：
  - URL 拼写错误（多数时候就是这个低级错误）
  - Context Path 没写对（访问 /oa/user 而不是 /user）
  - web.xml 中 servlet-mapping 写错
  - 注解 @WebServlet(\"/user\") 忘记写斜杠
  - webapp 没有被部署（检查 webapps 目录或 server.xml 配置）
  - 访问的是静态文件但放在了 WEB-INF 下（WEB-INF下的资源不可直接访问）

六、HTTP 请求 500（服务器内部错误）

一般都需要看日志（logs/localhost.yyyy-MM-dd.log）：
  - ClassNotFoundException：缺少jar包，检查 WEB-INF/lib
  - NoSuchMethodError：jar包版本冲突
  - NullPointerException：代码逻辑错误，看堆栈定位

七、HTTP 请求 405（方法不允许）

  - doGet/doPost 方法没有 Override（父类 HttpServlet 的默认实现就是返回405）
  - 检查：方法签名是否正确
    doGet(HttpServletRequest req, HttpServletResponse resp)
    参数类型必须是 HttpServletRequest 而不是 ServletRequest
  - 注解 @WebServlet 必须用在正确的方法上

八、线程池满导致请求被拒

现象：日志中出现 \"RejectedExecutionException\"

原因：并发超过了 ThreadPoolExecutor 的最大处理能力

快速解决：
  - 增大 maxThreads 和 maxConnections
  - 加一个 Nginx 在前面缓冲请求

根本解决：
  - 检查是否有慢接口导致线程长时间不释放
  - 优化数据库查询、增加缓存
```

# 33.Tomcat学习资源与进阶方向

```
一、官方资源

1. 官方文档（第一手资料）
   https://tomcat.apache.org/tomcat-10.1-doc/
   必读：Architecture、Configuration Reference、Security Considerations

2. 官方 Wiki
   https://cwiki.apache.org/confluence/display/TOMCAT/
   各种 How-To、FAQ、性能调优指南

3. GitHub 源码
   https://github.com/apache/tomcat
   Star + Watch，看 Commit 记录是很好的学习方式

二、推荐书籍

1. 《How Tomcat Works》（中文版：《深入剖析Tomcat》）
   - 适合：刚学完 Servlet，想了解 Tomcat 内部原理
   - 特点：从零开始逐步构建一个Servlet容器，浅显易懂
   - 推荐指数：★★★★★（入门首选）

2. 《Tomcat内核设计剖析》
   - 适合：有一定Tomcat使用经验，想深入源码
   - 特点：豆瓣高分，图文并茂，重点剖析核心组件
   - 推荐指数：★★★★★

3. 《Tomcat架构解析》
   - 适合：面试前突击或快速梳理架构
   - 特点：对 Pipeline-Valve、ClassLoader 讲解清晰
   - 推荐指数：★★★★

三、进阶学习路线

阶段一：会用（你现在的水平）
  - 能在IDEA中配置Tomcat，部署webapp
  - 理解Servlet生命周期，会用Filter和Listener
  - 会配置server.xml的常见参数

阶段二：懂原理（读完本章后的目标）
  - 理解请求从Socket到Servlet的完整链路
  - 理解Connector/Container的架构设计
  - 理解类加载机制和热部署原理
  - 能看懂部分关键源码

阶段三：能调优
  - 根据业务特点选择合适的 I/O 模型
  - 合理配置线程池参数
  - 能解决生产环境的内存泄漏、死锁等问题
  - 能设计高可用的集群架构

阶段四：能扩展
  - 会写自定义 Valve 实现特定拦截逻辑
  - 会写自定义 LifecycleListener 管理资源
  - 能在 Spring Boot 中定制 Tomcat 配置
  - 理解整个 Tomcat 启动机制的扩展点

阶段五：能贡献
  - 能给 Tomcat 提交 Bug Fix
  - 能参与 Tomcat 源码讨论
  - 能将 Tomcat 的设计思想应用到自己的框架中

四、面试高频考点汇总

对照这些考点，看看自己能答出多少：

1. ★★ Tomcat的整体架构（Server/Service/Connector/Container）
2. ★★ Connector 的工作流程（Acceptor → Poller → Worker → Processor → Adapter）
3. ★★ Tomcat 的类加载机制（为什么打破双亲委派？加载器层次？热部署原理？）
4. ★★ Pipeline-Valve 机制（和 Filter 链的关系？）
5. ★  BIO / NIO / NIO2 / APR 的区别和选择
6. ★  server.xml 中 Connector 各参数的含义
7. ★  Tomcat 的启动流程（Bootstrap → Catalina → Server → Service → ...）
8. ★  一个 HTTP 请求在 Tomcat 中的完整处理流程
9. ★  如何排查和解决内存泄漏（PermGen/Metaspace/Heap）
10. ★  Tomcat 集群和 Session 共享方案
11. ☆  WebSocket 在 Tomcat 中的实现
12. ☆  StandardContext 的启动过程
13. ☆  热部署 vs 热加载的区别
14. ☆  Tomcat 中 Lifecycle 状态机的设计

如果你对以上大部分问题都能答出来，那恭喜你，你已经把 Tomcat 学透了。
如果还有不会的，翻回对应章节再看一遍，然后打开源码验证——这是最好的学习方法。

最后提醒一句：掌握框架的原理不是目的，写出更好的代码才是。
理解 Tomcat 的架构设计，能让你在以后接触 Spring Boot / Spring MVC / MyBatis 等框架时，
\"知其然，更知其所以然\"。
```
