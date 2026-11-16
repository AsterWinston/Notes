本篇覆盖 Tomcat 的高频面试题，按"主题块 → 详细分块 → 题目 → 解答"组织。四个主题块：架构与请求处理、类加载与部署、会话管理、调优安全与整合。Servlet 规范层问题见 4_java_web.md，Spring Boot 整合见 10_springboot.md，反向代理部署形态见 12_nginx.md，类加载理论基础见 2_jvm.md。

# 1. 架构与请求处理

## 1.1 组件层级

### 1.1.1 Tomcat 是什么？和 Web 服务器、应用服务器什么关系？

**结论**：Tomcat = Apache 开源的**轻量级 Servlet 容器 + HTTP 服务器**——对外实现 Servlet/JSP/EL/WebSocket 规范，对内提供网络通信、协议解析、生命周期管理与多线程调度；是 Boot 的默认内嵌容器，也是独立部署 war 的经典选择。

**"容器"职责四件**（与 4_java_web.md 4.4.1 同题展开）：① Coyote/Connector 负责 TCP 收发与 HTTP 报文解析，把字节流包装成 Request/Response 对象；② Catalina 容器树负责 Servlet/JSP 的加载、初始化、请求派发、销毁全程；③ 线程池调度——每请求一个工作线程调 service（单实例多线程模型，4_java_web.md 1.1.3）；④ 配套服务：Jasper（JSP 编译，产物在 work 目录）、JNDI、集群复制、安全约束（Realm/Valve）。

**与完整应用服务器分界**：WebLogic/WebSphere/GlassFly 类应用服务器实现 Jakarta EE 全家桶（EJB、JMS、CDI、JPA、Connector 规范…）；Tomcat 只做 Web 层子集。"轻量"的底气在于：Spring 生态把事务、持久化、消息等都自己解决了，企业应用 90% 场景用不到 EJB——SSM/Spring Boot 跑在 Tomcat 上就是历史答案。

**版本脉络**（2026 口径）：8.5/9.x 属 javax.* 时代（9.x 维护中）；**10.1 = jakarta.* 命名空间 + Servlet 6.0 + Java 11**；**11.x = Servlet 6.1 + Java 17，移除了 APR 连接器与 HTTP/2 Server Push**。javax→jakarta 与 Boot 2/Boot 3 的代际完全对应（见 10_springboot.md 3.2.3），容器与应用的"时代"必须配对，混装直接 `ClassNotFoundException`。

**独立版目录结构**（追问常考"你知道 Tomcat 目录吗"）：

| 目录 | 内容 |
|------|------|
| bin | 启动/停止脚本，bootstrap.jar（自举入口）、catalina.jar（启动类） |
| conf | server.xml、web.xml（全局默认，含 DefaultServlet/JSP）、context.xml、tomcat-users.xml、catalina.properties（类加载器路径） |
| lib | Common 类加载器的共享库（jakarta servlet-api 等规范类、Tomcat 内部 API） |
| logs | catalina.out、localhost_access_log.* |
| webapps | 部署目录（ROOT 默认应用） |
| work | **每个 Context 的工作目录**：JSP 翻译编译产物、SESSIONS.ser 会话持久化文件都在这 |
| temp | catalina.pid、临时文件（catalina.base 下） |

**易错**："Tomcat 就是 Web 服务器"只对一半——动态处理归它，但生产入口通常还有 Nginx：静态文件、TLS 终结、限流防护由代理层承担，Tomcat 专注业务（部署形态见 12_nginx.md）。

### 1.1.2 Tomcat 的整体架构是怎样的？各组件职责？

**结论**：一句话——**"一层网络，一层路由，最底下才是 Servlet"**。组件分两条线：Connector 线（Server→Service→Connector，Coyote 网络侧）与容器树线（Engine→Host→Context→Wrapper，Catalina 容器侧），CoyoteAdapter 是两者的桥。

```text
Server（一个实例一个，port/shutdown）
 └─ Service（N 个 Connector + 1 个 Engine，可多个 Service）
     ├─ Connector（协议端点：TCP accept + HTTP 解析 + 线程池）
     └─ Engine（顶级容器，defaultHost）
         └─ Host（虚拟主机，按域名路由）
             └─ Context（一个 Web 应用）
                 └─ Wrapper（一个 Servlet）
```

| 组件 | 对应配置 | 职责要点 |
|------|---------|---------|
| Server | `<Server port="8005">` | 整个 Tomcat 实例，统一管理各 Service 启停；shutdown 命令端口（加固话题） |
| Service | `<Service name="Catalina">` | 把一组 Connector 和一个 Engine 捆成完整服务单元；HTTP 8080 与 HTTPS 8443 两个 Connector 就挂同一 Service |
| Connector | `<Connector port protocol>` | 网络 I/O 全部细节：endpoint（NIO/NIO2）+ 协议解析（Http11Processor）+ 线程池 + processor 缓存；"网"与"容器"的分界 |
| Engine | `<Engine defaultHost>` | 接收 Adapter 递来的请求，按域名找 Host；一个 Service 内唯一 |
| Host | `<Host name appBase>` | 虚拟主机：一个 IP 托管多域名（1.1.3） |
| Context | `<Context>` / webapps 目录 | 一个应用：独立类加载器、ServletContext、Manager（会话）、Realm；生命周期与部署全在这层 |
| Wrapper | 由 web.xml/注解生成 | 一个 Servlet 的"管理壳"：负责其加载、实例化、init/service/destroy 调度、load-on-startup |

**隐藏的第三条线**：每个 Container 都挂一条 **Pipeline（管道）**，管道里是若干 **Valve（阀门）**，末尾的 basic valve 指向"进入下一级容器"——Engine→Host→Context→Wrapper 的逐层路由，物理上就是一层层 Pipeline 走完（Valve 详解见 4.2.1）。

**易错与追问**：① "Engine 唯一吗"——一个 Service 内唯一，多 Service 可多 Engine；② "谁解析 HTTP"——Connector 里的 Processor，**不是** Servlet；到 Wrapper 时已是解析完的 Request 对象；③ Connector 与 Wrapper 是"1 个进程 N 个连接对 M 个 Servlet"的多对多，中间靠线程池解耦（1.2.2）。

### 1.1.3 一台 Tomcat 如何部署多站点/多应用？

三个租户层级，粒度从细到粗：

1. **同域多应用（Context 级）**：war 或目录丢进 `webapps/`，按目录名定 context path（`/appA`），`ROOT` 对应 `/`；映射时按 **URI 前缀最长匹配**选 Context。Boot 时代等价物是网关路径路由（19_spring_cloud_gateway.md）。
2. **多域名（Host 级虚拟主机）**：`<Host name="www.b.com" appBase="b_sites" unpackWARs="true" autoDeploy="true">` + `<alias>`；Engine 按**请求 Host 头**（`request.getServerName()`）匹配 Host 的 name，匹配不上走 `defaultHost`。需要 DNS/hosts 先把域名指到同一 IP。
3. **多端口/多协议（Service 级）**：不同 Connector 组合成多个 Service，各带独立线程池；HTTP/HTTPS 通常仍用一个 Service 两个 Connector（8080/8443，`redirectPort` 配合安全约束跳 HTTPS，4.2.2）。

**HTTPS 多域名**不用多端口：一个 SSL Connector 内多个 `SSLHostConfig` 按 **SNI**（ClientHello 里的域名）分发证书——注意虚拟主机匹配（HTTP Host 头）与证书匹配（SNI）是**两套独立机制**，配置要保持同步（`strictSni` 默认 true 拒绝不匹配）。

**生产口径**：单 Tomcat 多站点在新系统里基本退役——主流是**一个应用一套容器实例**（war 拆微服务、再到每 Pod 一 Boot 进程，32_kubernetes.md），隔离性、爆炸半径、发布独立性都优于多站点合宿。但虚拟主机概念必须会答，且 Nginx 的 `server{}` 按 server_name 分流与此完全同构（12_nginx.md）。

**易错**：多站点共享同一进程——A 应用 OOM/跑满 CPU 直接拖死 B；共享 lib 里的类被 A"动手"（静态状态）B 会被牵连（2.1.3）。这是"合宿"被抛弃的根本原因，答题时补上这句就超出背题型答案。

## 1.2 请求处理与线程模型

### 1.2.1 描述一个请求在 Tomcat 中的完整处理流程

**九步链路**（背主线，细节按追问展开）：

1. **Acceptor** 调 `accept()` 拿到 SocketChannel，设非阻塞、配 8K 读写缓冲，封装 NioChannel 注册进 Poller 事件队列；
2. **Poller** 的 Selector `select()`（默认 1s 超时）发现该连接有可读事件；
3. Poller 把就绪通道封装 **SocketProcessor 任务提交 Worker 线程池**；
4. Worker 执行任务：`Http11Processor` **状态机解析**请求行/请求头（请求体延迟到业务读取时再解析），填充 Coyote 内部 `Request/Response` 对象；
5. **CoyoteAdapter.service()**：包装出 facade 层的 `RequestFacade/ResponseFacade`（应用拿到的 HttpServletRequest，回收时释放），从网络侧进入容器侧；
6. **Engine Pipeline**：Engine 级 Valve 依次执行 → basic valve 按域名路由到 Host；
7. **Host Pipeline** → 按 context path 匹配到 Context；**Context Pipeline**（访问日志、安全约束 Valve 在这层生效，4.2）；
8. **Wrapper Pipeline** → 最终 Wrapper 的 Valve 构建 **Filter 链**，链走完后调 `Servlet.service()`——Spring MVC 应用即 DispatcherServlet（其后流程见 8_spring_mvc.md 1.1.1，Servlet 生命周期见 4_java_web.md 1.1.2）；
9. 业务写响应 → 缓冲区 commit 输出 → 任务结束，Worker 归还线程池；**keep-alive 连接重新注册 OP_READ 回到第 2 步等下一个请求**（同一连接多次在 Poller/Worker 间往返）。

**对象与线程的流转要点**：一个请求内对象经历"Coyote Request（内部）→ Facade（给应用）→ recycle（池化复用，processorCache 默认 200）"；线程在 Poller→Worker 交接，**异步 Servlet（AsyncContext）则归还线程后业务在别的线程继续**（4_java_web.md 3.3.1）。异常没人处理时，一路兜回 **ErrorReportValve** 生成容器错误页（Boot 场景由 BasicErrorController 接管错误页语义，见 10_springboot.md 4.2.1）。

**答题策略**：先报"Acceptor→Poller→Worker→Adapter→Engine/Host/Context/Wrapper 四级管道→Filter→Servlet→原路写回"骨架，再按面试官兴趣展开某一跳。与 8_spring_mvc.md 1.1.1 是同一请求的两段——容器段 + MVC 段，串起来答是满分答卷。

### 1.2.2 NIO 模型的 Acceptor/Poller/Worker 三种线程各干什么？

**结论**：三角色 = **"接电话的、听电话响的、讲电话的"**——连接建立、就绪分发、业务处理分离，用极少线程撑住大量连接。

| 角色 | 默认数量 | 职责 |
|------|---------|------|
| Acceptor | 1（acceptorThreadPriority=5） | 死循环 `ServerSocketChannel.accept()`；新通道设非阻塞、挂 per-connection 读写缓冲（`socket.appReadBufSize=8192`），封装 NioChannel 塞进 Poller 注册队列，**不碰任何数据** |
| Poller | 随核数缩放（`max(1, cores/4)` 量级，早期固定 2） | 持 Selector：先消费 Acceptor 的注册队列，再 `select()` 轮询就绪事件；可读/可写就绪 → 提交 SocketProcessor 给线程池；select 超时 1000ms 兼做**过期连接清理**（selectorTimeout） |
| Worker | maxThreads=200 | 执行 SocketProcessor：读字节 → HTTP 解析 → 派发处理 → 写响应；读不完/写不出（返回半包状态）→ 重新注册兴趣集，回 Poller 等下次就绪 |

**与 BIO 的本质区别**：BIO 一连接占一线程且阻塞在读上——1 万空闲长连接要 1 万线程（栈内存 + 上下文切换双爆炸）；NIO 空闲连接只占一个通道位（maxConnections，默认 10000/新版 8192）+ 约 16K 缓冲 + fd，**线程数与连接数解耦**，这是 Tomcat 高并发的地基。

**高频追问**：① "解析 HTTP 在哪个线程"——Worker（SocketProcessor 里），Poller 只分发；② "Worker 阻塞在慢 SQL 会怎样"——该线程被业务占死，但不影响连接层（连接由 Poller 继续照看）——表现为"能建连、响应慢"，与 maxConnections 打满的"建连都难"不同（1.2.3 症状对照，容量话题见 4.1.2）；③ "Poller 挂了会怎样"——select 异常（历史上 epoll spin 等 JDK bug）Tomcat 会重建 Selector；Poller 数太少高并发下分发可能成瓶颈，可调；④ 响应写出默认也在 Worker 完成，超大文件靠 **sendfile** 走内核零拷贝（与 compression 互斥，4.1.3）。

### 1.2.3 maxConnections、acceptCount、maxThreads 分别控制什么？

**结论**：三道闸门层层递进——**线程（maxThreads）→ 连接计数（maxConnections）→ OS 半连接/accept 队列（acceptCount）**，最后才到"拒绝"。官方文档语义：请求超出空闲线程 → 新连接继续被接受进 **maxConnections** 封顶（此时"接受但不处理"，挂起等线程）→ 连接数也到顶 → 交给 **OS backlog（acceptCount 控制长度）** → 再满 → 连接被拒/超时。

| 参数 | 默认 | 管什么 | 打满的表现 |
|------|------|--------|-----------|
| maxThreads | 200 | Worker 线程池上限 = 同一瞬间"处理中"的请求数 | 线程全 busy，新请求就绪了没人处理，RT 飙升 |
| maxConnections | NIO 10000（10.1+ 起 8192；-1 不计上限） | 通道注册表容量：所有连接（含空闲 keep-alive）计数 | 新连接建得上但不读数据（客户端视角"请求卡住"） |
| acceptCount | 100 | maxConnections 满后的 OS 层等待队列深度 | **Connection Refused / SYN 超时** |
| connectionTimeout | 60000ms（出厂 server.xml 写 20000） | accept 后等首行请求数据的最长时间 | 慢客户端/攻击连接长期占位 |

**易错三连**：① 把 acceptCount 说成"请求排队队列"——它是**OS 层连接队列**，"请求等线程"发生在 Tomcat 自己维护的队列/连接闸门里，不是 backlog；② "Connection refused ≠ 线程不够"——refused 是 acceptCount 满，"能连上但卡住"才是线程/maxConnections 的事，排障先分清症状；③ maxConnections 包含空闲 keep-alive 连接——长连接网关场景（下游挂着上万 idle 连接）先到的是它而不是 maxThreads，`appReadBufSize/appWriteBufSize` 乘连接数还要算内存账。

**组合心算**：默认配置的理论姿态 = 200 线程处理中 + 若干排队 + 约 1 万连接在册 + 100 OS 排队——对绝大多数业务应用，瓶颈应出现在 maxThreads 或下游（4.1.2），而不是连接闸门；连接闸门异常先满往往是慢响应堆积所致。

### 1.2.4 Tomcat 的 I/O 模型演进过哪些？APR 是什么？

**协议选项现状**（`protocol` 属性，Tomcat 11 文档口径）：`Http11NioProtocol`（**默认**）、`Http11Nio2Protocol`（AIO 补全模型， niche）；历史角色：

- **BIO**（`Http11Protocol`）：一连接一线程时代，8.5 起移除——问到就当演进史讲；
- **APR**（`Http11AprProtocol`）：**JNI 调 Apache Portable Runtime + OpenSSL 本地库**，历史卖点：SSL 硬件级加速（JSSE 慢的年代）、原生 sendfile、更低内存。**Tomcat 11 已移除 APR 连接器**——现代 JDK 的 TLS（AES-NI 硬件指令）与内核 sendfile 已补齐差距，装 JNI 依赖（apr/tomcat-native）的运维成本不再值得；OpenSSL 作为 TLS 实现的能力仍可通过 Tomcat Native 给 NIO 使用；
- **NIO2**：Java7 AIO（CompletionHandler 回调模型），与 NIO 的 Selector 轮询模型相对；实践中 NIO 足够，NIO2 存在感低但仍在；
- **虚拟线程**：`useVirtualThreads=true`（Tomcat 10.1+ / Java 21）——线程池形态被"每请求一虚拟线程"取代，等待不再占平台线程（4.1.1）。

**一句话选型**："新项目 = 默认 NIO，什么都不动"；知道"当年为 SSL 装 APR"的历史，再补"这条路线已被 JDK TLS 演进淘汰、Tomcat 11 移除"，就覆盖了面试官想考的时效判断。HTTP/2 不靠换连接器：在 Connector 内嵌 `<UpgradeProtocol className="org.apache.coyote.http2.Http2Protocol">` 即可（h2/h2c，服务端推送在 Servlet 6.1/Tomcat 11 移除）。

# 2. 类加载与部署

## 2.1 类加载器体系

### 2.1.1 Tomcat 的类加载器层次是怎样的？各加载什么？

**分层全景**（自上而下是委派方向，2_jvm.md 3.2.3 的完整版）：

```text
Bootstrap（JVM）        —— java.*、JDK 平台类
  └ System（应用类加载器）—— CLASSPATH：bin/catalina.jar、bootstrap.jar（Tomcat 自举代码）
      └ Common            —— $CATALINA_HOME/lib 等（catalina.properties 的 common.loader）
                              规范 API（jakarta.servlet-api.jar）+ Tomcat/所有应用共享库
          ├ Catalina      —— conf/Catalina（引擎专属， historically 已并入 lib）
          ├ Server        —— conf/Server（同上）
          └ Shared        —— shared/（同上）：Tomcat 6 起三者默认与 Common 同目录
              ├ WebappClassLoader × N（每 Context 一个）—— WEB-INF/classes → WEB-INF/lib
              └ JasperLoader × M（每 JSP 一个）—— 编译出的 Servlet 类，父为所在应用的 WebappClassLoader
```

**各层职责记忆点**：`java.*` 归 JVM；启动器（Bootstrap.main 所在）归 System；**servlet-api 等规范类与全应用共享的 jar 归 Common**（"容器提供的类"物理上就在 lib）；业务类与业务依赖归各应用自己的 WebappClassLoader。

**历史包袱追问**：为什么 Catalina/Shared 三个加载器"名存实亡"——早期想区分"Tomcat 自用/全应用共享/仅 Web 应用共享"三层路径，但实践中要么没人用要么引发混乱，Tomcat 6 起默认配置把三者都指回 lib（目录注释掉），日常只需要 Common；配置在 `conf/catalina.properties`。

**与 SPI 的纠缠**（追问深水区）：驱动/实现类往往放在应用 lib，而注册表（DriverManager、JNDI InitialContext、ServiceLoader 的**线程上下文类加载器**）站在 JDK/容器侧——"父侧静态注册表看不见子侧实现"正是双亲委派的经典裂缝（2_jvm.md 3.2.2；Tomcat 场景的后果与清理责任见 2.2.2）。

### 2.1.2 Tomcat 如何打破双亲委派？为什么要打破？

**加载顺序**（WebappClassLoader，`delegate=false` 默认）：

1. 已加载缓存（resolved classes）；
2. JVM 核心类强制走 Bootstrap（`java.*`）；**规范"受保护包"（如 jakarta.servlet.*）也委派优先**——防止应用劫持容器要与之交互的接口类；
3. 搜本应用 `WEB-INF/classes`；
4. 搜本应用 `WEB-INF/lib`；
5. 仍未找到才委派父加载器（Common→System→Bootstrap）。

对照标准双亲委派"先父后己"，Tomcat 对普通类是 **"先己后父"**——这就是"打破"的确切形态；`Context` 的 loader 配 `delegate="true"` 可切回标准顺序。

**为什么打破（三条设计理由）**：

1. **应用隔离**：多站点合宿时 A 用 fastjson 1.x、B 用 2.x，各自先加载各的，天然互不污染（2.1.3）；
2. **自包含部署**：war 打包全部依赖即可跑，不要求把 jar 装进容器 lib——部署独立性；
3. **热部署基础**：换应用 = 换新 ClassLoader 重新全量加载，"换 loader 即换类"（2.2.1）——不打破委派，类先被父加载器占住，loader 替换就失去意义。

**为什么留"委派优先"的口子**：容器与应用要对话——容器把请求交给"它认知里的 `jakarta.servlet.Servlet`"，若应用用自己的加载器重定义一份 Servlet 接口，cast 当场碎裂（ClassCastException）。规范 API 必须容器说了算。

**经典事故**：war 里误打包 servlet-api（Boot 场景对应把 `spring-boot-starter-tomcat` 配成 provided 之外的错法）——轻则一份"影子类"永不生效，重则与容器版本不一致引发诡秘冲突；正确姿势是 scope=provided 交给容器/内嵌 starter 提供（3_maven.md 2.1.3、10_springboot.md 2.2.2）。

### 2.1.3 应用之间如何实现隔离？ClassNotFoundException/NoClassDefFoundError 如何排查？

**隔离本质**：JVM 里**类的身份 = ClassLoader 实例 + 全限定名**。同名不同 loader = 不同 Class——两个应用各自加载的 `com.x.Foo` 互相 `instanceof` 为 false、static 变量互不可见；每个 Context 还独享 ServletContext、JNDI `java:comp/env`、Manager 与会话配置。**"A 改了某静态缓存会不会影响 B"——不会，这正是 loader 隔离的意义；但放进 lib（Common）就会**：全应用共享同一份 Class，静态状态、版本都变成公共财产，双刃剑。

**三种"找不到/对不上"的辨析**：

| 症状 | 根因方向 |
|------|---------|
| `ClassNotFoundException` | 显式加载失败：类根本不在 loader 可见路径（缺 jar、Class.forName/SPI 找错 loader） |
| `NoClassDefFoundError` | 编译时有、运行时没有：依赖没打包/被 exclude/静态初始化首轮失败（**第一次 Caused by 才是真相**，之后只会反复抛 NCDFE） |
| `ClassCastException: A 不能转 A` | 同名类被两个 loader 各加载一份——先看 CCE 两侧类的 loader hash（Arthas `sc -d`） |

**排查清单（按序走）**：

1. **包内确认**：`unzip -l app.war | grep xxx`——jar 在不在 WEB-INF/lib、class 在不在 classes；Maven scope 是否 provided/被 exclusion 误伤（依赖树，3_maven.md 2.1.2）；
2. **冲突确认**：war 内与容器 lib 同名 jar（版本不同）——除规范包外应用侧先加载（2.1.2），lib 那份被遮蔽，行为随"谁赢了"漂移；
3. **时代确认**：javax vs jakarta 错配（应用 9.x 时代产物跑在 10.1/11 容器上必报 `ClassNotFoundException: jakarta.servlet.*`，见 10_springboot.md 3.2.3）；
4. **加载源观测**：`-Xlog:class+load=info`（JDK9+）打印每个类由哪个 loader 从哪个文件加载，一击定位遮蔽问题；
5. **JDK 版本变体**：`UnsupportedClassVersionError`（major version 冲突）——不是找不到类，是类太"新"。

## 2.2 热部署与泄漏

### 2.2.1 Tomcat 如何实现热部署？为什么生产要关闭？

**定义**：不重启容器进程、只重载单个 Context。三层机制配合：

1. **变更感知**：Host 的 backgroundProcess 周期扫描（默认 10s）——`autoDeploy` 发现 webapps 新增 war/目录则部署；Context 监视 `WEB-INF/web.xml` 与 classes/lib 的修改时间（`reloadable="true"` 或默认策略），发现变化触发 reload；
2. **重载过程**：停止（Servlet destroy、会话钝化持久化）→ **丢弃旧 WebappClassLoader** → 新建 loader 重新全量加载 classes/lib → Servlet 重新 init → 会话活化。核心思想：**换 loader = 换一整套类**（2.1.2 设计理由在这里兑现）；
3. **配套层**：JSP 改动由 Jasper 按时间戳重编译（work 目录产物更新，4_java_web.md 4.1.1）；Spring 容器**不会**跟着 reload——传统 SSM 热部署实为整个应用重启上下文；现代开发流 = **Boot devtools 双类加载器热重启**（base/restart 分层，见 10_springboot.md）或 JRebel 字节码增强。

**生产关闭（autoDeploy=false + 不做热重载）四条理由**：

1. **Metaspace 泄漏风险**：旧 loader 若有线程/ThreadLocal/静态注册表钉住就死不掉，N 次 reload 后 OOM: Metaspace（2.2.2）；
2. **重载窗口不一致**：destroy 与 re-init 之间的请求得到半成品响应，没有原子性；
3. **扫描开销与误触发**：后台监视 file 时间戳，运维脚本 touch 一下就是一次"意外发版"；
4. **现代发布流程取代**：不可变镜像整体换实例（23_docker.md、32_kubernetes.md、33_cicd.md），热部署只留给开发机。

**追问**："把 class 拷进 work 目录能生效吗"——能（监视 classes 变化触发 reload），这正是早年 IDEA + Tomcat 的土办法；但 watch 粒度粗、代价是整个 Context 重启。

### 2.2.2 Tomcat 常见的内存泄漏有哪些？如何排查与预防？

**泄漏模型**（多应用共享容器的特有病，Boot 一进程一应用天然不踩）：卸载应用 = 丢 WebappClassLoader——但 **GC Root 到该 loader 的强引用链**还在，类与元数据永久驻留 Metaspace。五大钉住路径：

1. **未结束的线程**：应用自起的非守护线程/Timer/线程池没随 Context 停止——线程对象的 contextClassLoader 与所执行类的静态引用双向钉住 loader。Tomcat 卸包时会做**泄漏探测**打日志（"This web application appears to have started a thread named [xxx]..."），并尽力 interrupt/清理（clearReferencesThreads），但"尽力"意味着别指望它；
2. **ThreadLocal 残留**：容器 Worker 线程池是**跨应用复用**的长命线程，业务往 ThreadLocal 塞了应用的类实例又没 remove——每条都是通向旧 loader 的锚（机制见 1_java.md 3.4）；
3. **JDBC 驱动注册**：驱动类由 WebappClassLoader 加载，却注册进 **DriverManager 的 JVM 级静态表**——经典"父侧注册表、子侧实现"裂缝（2.1.1）；Tomcat 卸包会注销（clearReferencesDrivers），但正确姿势仍是应用自己在 `contextDestroyed` 注销；
4. **静态集合/单例注册表**：自研缓存、shutdown 没退出的服务（Zookeeper/NIO 框架的后台线程与静态 instances 表）、JNDI/RMI 绑定残留；
5. **序列化/内省缓存**：`ObjectStreamClass`、`java.beans.Introspector` 等缓存 keyed by Class——引用着旧类；Tomcat 有清理，业务自己的反射缓存要自己清。

**症状与排查**：反复热部署后 **Metaspace 频繁 Full GC / `OutOfMemoryError: Metaspace`**；`jcmd <pid> GC.class_histogram` 或 **heap dump → MAT**：看同名类出现多份 Class 对象、WebappClassLoader 实例数 > 应用数，用 Dominator Tree 找"谁抱着 loader"（线程对象是头号嫌疑）。工具细节见 2_jvm.md。

**预防纪律**：① 自启线程/资源"谁创建谁关闭"，`ServletContextListener.contextDestroyed` 里对称释放；② ThreadLocal 用完 finally remove；③ 生产关 autoDeploy/reloadable（2.2.1）；④ 用 Boot 内嵌独立进程部署——把"多应用共享 loader"这个问题连根拔掉；⑤ 兜底：滚动重启。

# 3. 会话管理

## 3.1 Manager 家族

### 3.1.1 Tomcat 内部如何管理 Session？StandardManager 有哪些机制？

**定位**：会话管理是 **Context 级组件**——每个 Web 应用挂一个 `Manager`（org.apache.catalina.Manager 接口，实现族见 3.1.3）。4_java_web.md 2.1.2 讲了"JSESSIONID 从请求到对象"的外部链路，本题拆内部。

**StandardManager 机制清单**：

- **存储**：`ConcurrentHashMap<String, Session>`（session id → StandardSession）；`maxActiveSessions` 默认不限，超限拒绝新会话并告警；会话对象用完回收到 `sessionCacheSize` 缓存池减少 GC；
- **id 生成**：`SessionIdGenerator`——SecureRandom 随机字节编码而成（不是 UUID 顺序可猜的那种），可选 `jvmRoute`/RouteId 后缀（`id.node1`），给负载均衡做会话粘滞识别用；Cookie 名 `JSESSIONID`，由 SessionCookieHandler/配置决定 HttpOnly/SameSite（4_java_web.md 2.1.1/2.2.1）；
- **过期检查**：容器后台线程驱动 `backgroundProcess()`，每 `backgroundProcessorDelay`（10s）× `processExpiresFrequency`（6）≈ **60s 一轮**遍历检查 `lastAccessedTime + maxInactiveInterval`（默认 1800s）——所以过期是"分钟级近似"而非秒级精确（与 27_redis.md 里 Redis 惰性+定期删除的过期哲学同构）；
- **钝化/活化**：持久化/复制前后触发 `HttpSessionActivationListener`（2.2.1 重载流程里的 passivate/activate 就是它）；
- **事件**：创建/销毁回调 `HttpSessionListener`（在线人数统计的标准挂点）。

**设计要点**：会话是**容器内存对象**——这个设定决定了后面所有题目的走向：重启没了（3.1.2）、集群要复制（3.1.3）、现代方案往外挪（3.2.1）。Session 里放的东西必须 Serializable 才有资格被复制/持久化，不可序列化属性在复制场景被**静默丢弃**（只留 WARN）——"本地好好的、集群上丢状态"的经典根因。

### 3.1.2 Tomcat 重启会丢 Session 吗？SESSIONS.ser 是干什么的？

**分三种情形**：

1. **正常优雅关闭**：StandardManager（继承 LocalStoreManager）在 stop 时把所有可序列化会话写进 `work/Catalina/<host>/<app>/SESSIONS.ser`；启动时读回、过期的丢弃——**同目录重启不丢**；
2. **kill -9 / 崩溃 / OOM**：来不及落盘，全丢；
3. **换节点/重调度**：SESSIONS.ser 是**本机私有文件**，不共享、不复制——换台机器就是从零开始，容器化里 Pod 重建文件系统也没了（32_kubernetes.md）。

**PersistentManager 补强**：在 Manager 外套一层 `Store`（FileStore/JDBC Store）——`swapThreshold` 触发的 **swap** 把闲置会话换出内存落盘/落库（访问时换回），`activeSessions` 限制内存驻留数，持久化策略可配"仅换出时写"或"始终写库"。它解决的是**容量与重启存活**，仍然不是集群共享。

**边界必须说清（这题的考点在这）**："重启不丢"≠"高可用"：SESSIONS.ser 只防单机优雅重启；节点故障、扩缩容、滚动发布要的是会话与节点解耦——那是 3.2.1 的题。另外反序列化恢复只恢复对象状态，监听器逻辑版本变了照样可能炸，生产上靠它兜底可以、当方案不行。

### 3.1.3 DeltaManager 和 BackupManager 的集群复制有什么区别？

**共同底座**：`<Cluster>` + Tribes 通道——成员发现（历史靠组播，新部署多用静态成员列表）、消息序列化传输、可选 EncryptInterceptor 加密复制流量。Session 属性 set/remove 触发复制包；**不可序列化属性直接不复制**（3.1.1 老问题）。

| 维度 | DeltaManager（全复制） | BackupManager（主备复制） |
|------|----------------------|--------------------------|
| 策略 | 变更广播给**所有**节点：增量包（只传变化的属性）+ 新节点加入时补全量 | 每个会话随机指定**一个**备份节点，只同步给它；所有权(owner)信息广播给集群 |
| 每节点数据 | 全量会话 | 自有 + 部分备份（约 1/n 份额） |
| 带宽放大 | O(节点数)——节点越多风暴越大 | O(1) 每会话 |
| 故障接管 | 任意节点都有全量，直接接管 | 会话的备份节点接任为新 owner（多一跳、且触发所有权再平衡，成员进出时开销集中爆发） |
| 适用 | **小集群（2~4 节点）+ 会话量小** | 大集群降复制流量，官方推荐场景 |

**必答题尾**："生产用哪个？"——都不用。Tomcat 原生集群复制是三时代产物（配置玄学、流量大、与会话对象耦合），现代口径见 3.2.1。但机制本身要讲得清，因为它把"复制语义"（增量/所有权/序列化）的通用难题演示了一遍。

## 3.2 现代会话共享

### 3.2.1 集群/微服务下的会话共享，现代方案怎么选？

四条路线，按工程优先级：

1. **Token 自包含（JWT）——微服务默认**：干脆取消"会话"，凭证里带声明，服务端只验签——无状态、天然水平扩展；代价是注销难（黑名单/短时效）、载荷大小与保密限制（11_spring_security.md、27_redis.md）；
2. **集中式存储（Spring Session + Redis）——传统会话语义的正解**：`spring-session-data-redis` 一个依赖 + `RedisHttpSession` 注解，Filter 层把 `request.getSession()` 偷换成 Redis 仓库（key=会话 id，Hash 结构 + 索引 + 过期通知），**容器无关**（Tomcat/Jetty/Undertow 通吃），重启/扩容/换节点会话都在——Manager 概念整个绕开；写策略（默认 ON_SAVE：响应提交前统一 flush）决定并发覆盖粒度（见 27_redis.md）；
3. **网关/认证服务统一会话**：会话归属集中到鉴权层，后端服务全裸无状态——Spring Security + OAuth2/SSO 形态（11_spring_security.md），与方案 1 常在一条架构线上；
4. **会话粘滞（保底权宜）**：LB 按源 IP/cookie 路由到固定节点（Nginx ip_hash/一致性哈希 + jvmRoute，见 12_nginx.md）——**不是共享**：节点挂会话即丢、扩缩容命中重排、滚动发布抖动；只适合"会话便宜、变更谨慎"的存量系统。

Tomcat 原生复制（3.1.3）与 Redis 化 Manager（如 tomcat-redis-session-manager 替换容器 Manager）都归"上一代姿势"：前者重、后者绑容器；**新系统答案就是方案 1 或 2**，面试要把这两条讲出取舍，而不是罗列名词。

# 4. 调优安全与整合

## 4.1 性能调优

### 4.1.1 Tomcat 线程池有哪些关键参数？和 JDK 线程池有什么不同？

**自研实现**（org.apache.tomcat.util.threads.ThreadPoolExecutor）：继承 JDK 语义，两处定制——① **核心线程也允许超时回收**（JDK 默认核心线程永久驻留，需 allowCoreThreadTimeOut）；② **TaskQueue 假满技巧**：线程数未到 maxThreads 时 offer 故意返回 false，逼迫先扩线程、扩满才真排队——即 **"先扩线程后入队"**，与 JDK 默认"先入队后扩线程"相反（对比动机见 1_java.md 5.2.4）。

**Connector 侧参数**（Boot 里对应 `server.tomcat.*`，见 10_springboot.md 4.1.1）：

| 参数 | 默认 | 说明 |
|------|------|------|
| maxThreads | 200 | 池上限；所有连接共享 |
| minSpareThreads | 10 | 常驻核心数（"预热"底量） |
| threadsMaxIdleTime | 60000ms | 超出核心数的空闲线程存活期 |
| maxQueueSize | Integer.MAX_VALUE | 内部任务队列容量（与 1.2.3 的 OS 层 acceptCount 是两层东西） |
| threadNamePrefix | http-nio-8080-exec- | 排障认线程靠它 |
| connectionTimeout | 60000（出厂 server.xml 写 20000） | 等首行数据 |

**两个进阶开关**：① **Service 级共享 `<Executor>`**——多个 Connector 挂同一线程池（HTTP/HTTPS 合并容量），此时 maxThreads 等参数失效；② **`useVirtualThreads=true`**（Tomcat 10.1+/Java 21）：每请求一个虚拟线程，阻塞不再是罪，maxThreads 意义淡出——但**下游连接池成为唯一闸门**（4.1.2），且注意 JDK21 时代 synchronized 钉载体的问题（JEP 491 后基本解除）。

**观测**：JMX `Catalina:type=ThreadPool`（currentThreadCount/currentThreadsBusy）、`thread 'http-nio-8080-exec-3'` 栈（Arthas thread / jstack）判断"忙线程堆在哪"——排障先于调参。

### 4.1.2 高并发下 Tomcat 容量如何估算？调参的正确姿势？

**Little's Law 起步**：稳态并发 ≈ **QPS × 平均响应时间(秒)**。例：目标 2000 QPS × 80ms = 160 在途请求 → maxThreads 至少 160 + 20% 余量 ≈ 200（恰好默认值——"默认 200"本身就是按普通业务画像定的）。反推吞吐上限：maxThreads / RT = 极限 QPS，RT 100ms → 2000 QPS/实例，要更多就加实例。

**调参三条纪律**：

1. **先看 RT 再动线程**：线程忙满通常是**慢在下游**（慢 SQL、GC 停顿、RPC 超时），200 个线程都卡在一条 SQL 上时加到 400 只会把数据库打得更惨——CPU 饱和或下游饱和时加线程是负优化；
2. **瓶颈在整条链路**：Tomcat 并发 × 实例数要 ≤ 连接池总供给（HikariCP 默认才 10，连接池话题见 16_mysql.md）、≤ MQ/缓存连接预算；队列类参数（acceptCount/maxQueueSize）不要拿来"扛压"——排队放大超时、雪崩时全链都在等死请求（快速失败 + 入口限流熔断归网关层，见 20_sentinel.md、19_spring_cloud_gateway.md）；
3. **压测定拐点**：阶梯加压画 RT-QPS 曲线，拐点前的并发才是容量；观测三件套：busy 线程数、GC 停顿、错误率（工具与压测法见 33_cicd.md 周边）。

**现代答案的落点**：单机调参的边际收益越来越小——**水平扩展 + 无状态化（3.2.1）+ 弹性伸缩**才是主路线（23_docker.md、32_kubernetes.md）；调参用于把单实例画像跑准，扩容用于把总量扛住。

### 4.1.3 compression 与长连接如何配置？

**压缩（Connector 属性）**：`compression="on|force|数字"`（force 无视大小强压）+ `compressionMinSize`（默认 2048B）+ `compressibleMimeType`（默认 text/html、text/xml、text/plain、text/css、js/json/xml 一族）；已是 gzip/br/deflate 等 Content-Encoding 的自动跳过。**两个坑**：① **sendfile 优先级高于压缩**——NIO 对 >48KB 静态文件走零拷贝直发，结果是"开了压缩静态大文件却不压"（`useSendfile=false` 或接受不压）；② 压缩吃本机 CPU、小响应压了白压。**现代分工**：静态/文本压缩交给入口的 Nginx/CDN（gzip + 预压缩 gzip_static/brotli，见 12_nginx.md），Tomcat 上 `server.compression.enabled=true` 只管动态接口即可（Boot 映射）。

**长连接三参数**：`maxKeepAliveRequests`（默认 100，-1 不限——建议显式有限值，防单连接无限霸占）、`keepAliveTimeout`（**默认继承 connectionTimeout**，建议单独调短到 5~15s：慢客户端/Slowloris 类攻击靠挂 idle 连接吃满 maxConnections，见 1.2.3）、`useKeepAliveResponseHeader`（响应带 `Keep-Alive: timeout=x` 告知客户端闲置预算，RFC 草案姿势）。**与上游对齐**：Nginx upstream 的 `keepalive` 空闲池必须 < 后端 keepAliveTimeout，否则 LB 拿到已被后端关掉的连接 → 偶发 502（这条链路见 12_nginx.md）。

**收益认知**：长连接省的是 TCP + TLS 握手与慢启动，HTTPS 下收益更大（握手 CPU 贵）；HTTP/2 之后多路复用进一步把"连接数"压小——maxConnections 压力变小，压力全部前移到线程/下游（回到 4.1.2）。

## 4.2 扩展点与安全

### 4.2.1 Valve 是什么？与 Filter 有什么区别？

**机制**：每个 Container（Engine/Host/Context/Wrapper）挂一条 **Pipeline**，Valve 是管道上的拦截点；`StandardPipeline` 的 basic valve 指向"进入下一级容器"——所以请求路径天然形成 **Engine valves → Host valves → Context valves → Wrapper valve（→ Filter 链 → Servlet）** 的四层嵌套拦截（1.2.1 第 6~8 步）。

| 维度 | Valve | Filter |
|------|-------|--------|
| 归属 | Tomcat 私有（org.apache.catalina.Valve），非规范 | Servlet 规范，跨容器可移植 |
| 作用粒度 | **容器级**：Engine 级一次管全站所有应用 | URL 模式/Servlet 名级 |
| 拿到的对象 | 内部 Request/Response（能碰 connector、mapping 细节） | ServletRequest facade |
| 时序位置 | 更早（进 Servlet 前的容器层） | 请求已在目标 Servlet 门前 |
| 典型职责 | 基建：日志、黑白名单、报错页、SSO | 业务横切：编码、鉴权、审计 |

**必会 Valve 清单**：`AccessLogValve`（pattern `%h %l %u %t "%r" %s %b %D`——%D 毫秒耗时是排障利器；新版本可 pattern="JSON" 直出结构化日志进采集管道；Boot：`server.tomcat.accesslog.*`）、`ErrorReportValve`（容器错误页生成者，Server 头/版本暴露在这收口）、`RemoteAddrValve/RemoteHostValve`（IP 黑白名单，管理路径的常驻保安：`<Valve className="...RemoteAddrValve" allow="127\.\d+\.\d+\.\d+"/>`）、`SingleSignOnValve`（同 Host 多 Context 的会话级 SSO）、`RewriteValve`、`PersistentValve`（单属性持久化）。

**三件套总答**（面试连招 Filter/Interceptor/AOP 的延伸版）：**Valve（容器）→ Filter（Servlet 规范）→ Interceptor（MVC）→ AOP（Bean 方法）**，越往下越贴近业务、越往上越贴近流量；框架实现基本不用 Valve 因为可移植性——Boot 内嵌时代 Valve 的入口是 `factory.addEngineValves/addContextValves` 定制器（4.3.1）。

### 4.2.2 Tomcat 的 Realm 认证和 HTTPS 如何配置？生产加固要点？

**Realm = 容器声明式安全的用户库**：`authenticate()` 校验凭证返回 Principal + 角色，挂在 Engine/Host 层。种类：`MemoryRealm`（tomcat-users.xml，演示用，密码配 CredentialHandler 做哈希加盐——SHA-256/PBKDF2 起步，别裸 MD5）、`JDBCRealm`（直连用户表）、`DataSourceRealm`（走 JNDI 数据源，**推荐**：连接池与改表都方便）、`JNDIRealm`（LDAP/AD 对接企业目录）、`JAASRealm`。

**声明式链路**：web.xml `security-constraint`（URL 模式 + 方法 → `auth-constraint` 角色；`transport-guarantee=CONFIDENTIAL` 自动跳 redirectPort 的 HTTPS）+ `login-config`（BASIC 401 弹窗 / FORM 自定义页 POST `j_security_check` / CLIENT-CERT 证书）——realm-name 与容器 Realm 对应。**现状判断要答出来**：细粒度权限、JWT、OAuth2 时代，新项目用 Spring Security 接管认证授权（11_spring_security.md），容器 Realm 只剩企业目录 SSO 兼容与"历史系统维护"场景；但它是 Tomcat 安全题的标准考点。

**HTTPS（Connector）**：8443 配 `SSLEnabled=true` + `SSLHostConfig`（keystore 现代默认 PKCS12；`Certificate` 嵌套支持一 connector 多域名证书 = SNI，`strictSni=true` 拒张冠李戴；TLS1.2/1.3 由 protocols/ciphers 控制；`client-auth="need|want"` 双向认证；OpenSSL 可作 JSSE 之外的实现——APR 连接器时代已结束，11.x 移除，见 1.2.4）。

**生产加固清单（问"Tomcat 怎么加固"照此答）**：① **TLS 终结上移到 Nginx/SLB**，内网明文 + `X-Forwarded-*` 还原（Boot `forward-headers-strategy`，见 12_nginx.md）；② **AJP 不用就关**（8009，Ghost of AJP CVE-2020-1938：属性注入可读任意文件/响应劫持）；③ **shutdown 端口** 8005 设随机密码或 -1；④ 删 manager/host-manager 与 example 应用（弱口令远程部署 = RCE 门票），保留则 RemoteAddrValve + 强角色；⑤ 版本暴露收敛：Connector `server` 属性覆盖 Server 头 + ErrorReportValve 处理；⑥ 专用低权限账号运行、`relaxedQueryChars` 等放行项按需最小化。

## 4.3 与 Spring Boot 整合

### 4.3.1 Spring Boot 如何启动嵌入式 Tomcat？

**调用链**（10_springboot.md 1.1.2 的容器段展开）：

```text
SpringApplication.run → refreshContext → refresh()
  └ onRefresh() → createWebServer()
      ├ getWebServerFactory()：取 ServletWebServerFactory Bean（工厂选型见 4.3.2）
      └ factory.getWebServer(...)：
          new Tomcat() → baseDir 设临时目录 → 建 Connector(Http11NioProtocol, port)
          → 建 Engine 并 addHost（autoDeploy=false，无 webapps 扫描）
          → 建 Context（docBase 临时、TomcatEmbeddedWebappClassLoader——直接以应用类加载器为父，
             单应用所以无需 loader 隔离，2.1 整套复杂度在此消解）
          → 收集 Servlet/Filter/Listener 注册 Bean（ServletContextInitializer 族）逐个挂进 Context
          → tomcat.start()：容器树 init/start，Connector 绑端口开始 accept
          → 包成 TomcatWebServer 返回，发布 WebServerInitializedEvent（见 10_springboot.md 2.1.3）
关闭：context.close() → WebServer.stop() → tomcat stop/destroy（优雅停机见 10_springboot.md 4.1.2）
```

**配置通道**（三选一按侵入度）：`server.*`/`server.tomcat.*` 属性（端口、线程、访问日志、压缩，均由 `ServletWebServerFactoryCustomizer` 反射映射到工厂）；`WebServerFactoryCustomizer<TomcatServletWebServerFactory>` Bean（属性覆盖不到的原生对象：加 Valve、relaxedQueryChars、改协议细节——4.2 各题的嵌入式等价写法）；直调 Connector 的定制回调（TomcatConnectorCustomizer/ProtocolHandlerCustomizer）。

**与独立部署的分界**：war + SpringBootServletInitializer 走"外部容器装载应用"的 SPI 路线（10_springboot.md 2.2.2、4_java_web.md 1.1.4）；Fat Jar 内嵌路线则**应用持有 Tomcat 对象**（打包原理见 10_springboot.md 2.2.1）。追问"port=0 干嘛的"——测试里让系统随机端口，`getPort()` 取实际值，集成测试标配。

### 4.3.2 嵌入式容器如何切换？有哪些坑？

**切换动作**（exclude + include，唯一正解）：

```xml
<dependency>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions><exclusion>
        <artifactId>spring-boot-starter-tomcat</artifactId>
    </exclusion></exclusions>
</dependency>
<dependency><artifactId>spring-boot-starter-jetty</artifactId></dependency>
```

**为什么必须 exclude（机制，见 10_springboot.md 4.1.1）**：Boot 用 `ServletWebServerFactoryAutoConfiguration` 内三个嵌套工厂配置（EmbeddedTomcat/Jetty/Undertow）按 classpath 探测，固定 `@Import` 顺序 + `@ConditionalOnMissingBean` 让位——**多种容器共存时先声明者静默胜出，实际就是 Tomcat 赢**：不报错、不警告，应用照跑在 Tomcat 上，"以为切了其实没切"是最隐蔽的事故；只有手工注册多个工厂 Bean 才会触发 `multiple ServletWebServerFactory beans` 启动失败。**验收办法**：看启动日志（`Tomcat started on port(s)` / `Undertow started on address`），或 jstack 里线程名（`http-nio-8080-exec` vs `XNIO-1 task` vs `jetty-`）。

**参数映射**：线程/连接概念同名但配置键与默认不同——`server.tomcat.threads.max` ↔ `server.jetty.threads.max` ↔ Undertow 的 `server.undertow.threads.io/worker`（XNIO 是 **io 事件线程 + worker 业务线程**两级模型，对应 1.2.2 的 Poller/Worker 思想）；accesslog、maxconnections 等各自一套 key，迁移时逐项核对而不是换 starter 就完事。

**差异清单（"切换必测"四大件）**：① **JSP**：Undertow 无 Jasper，用 JSP 直接出局；② **会话**：3.1 整个 Manager 家族是 Tomcat 私产，换容器后集群复制/SESSIONS.ser 全没了——反正现代方案本来就该外置（3.2.1），这条反而成了顺水推舟；③ **WebSocket/长连接**：配置面与默认线程池行为不同（见 36_websocket.md），压测覆盖；④ **观测**：JMX MBean、线程命名、访问日志格式全变，监控面板与告警规则跟着改。

**选型认知收尾**：默认 Tomcat（生态最全、资料最多）；Undertow 赢在内存占用与 XNIO 精简模型；Jetty 赢在长连接/非标准协议的灵活度与老牌 HTTP/2 经验。**换容器是"实现细节"级别的决定，不是架构问题**——切换成本主要在行为差异回归，不在 pom 那三行。