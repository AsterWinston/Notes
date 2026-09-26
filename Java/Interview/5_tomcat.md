# 1. 知识点
- **Tomcat定位**：Apache开源的轻量级Servlet容器+Web服务器，实现Servlet/JSP规范，广泛用于Java Web应用部署，Spring Boot默认嵌入式容器
- **整体架构**：Server（最顶层，代表整个Tomcat实例）→ Service（包含一组Connector和Engine）→ Connector（接收请求，支持多种协议和I/O模式）→ Engine（Servlet引擎，负责请求处理分发）→ Host（虚拟主机，一个IP部署多个域名站点）→ Context（Web应用，一个应用）→ Wrapper（一个Servlet）
- **Server组件**：代表整个Tomcat服务器实例，一个JVM只有一个Server实例，负责启动和关闭所有Service
- **Service组件**：将一组Connector和一个Engine组合在一起，提供完整的请求处理服务。一个Server可包含多个Service（不同端口、不同协议）
- **Connector组件**：负责接收客户端TCP连接并解析HTTP请求，将请求封装为标准的ServletRequest对象后交给Engine处理。支持多种协议和I/O模式
- **Engine组件**：顶级容器，负责将请求映射到对应的虚拟主机(Host)，将处理结果返回给Connector。一个Engine下可以有多个Host
- **Host组件**：虚拟主机，根据域名将请求映射到对应Web应用。如`www.a.com`→应用A，`www.b.com`→应用B，实现单物理机托管多站点
- **Context组件**：代表一个Web应用（war包或目录），包含多个Wrapper。每个Context有独立的ServletContext和类加载器
- **Wrapper组件**：代表一个Servlet实例，是最底层的容器，负责Servlet的加载、初始化、请求调用和销毁
- **Connector线程模型（Endpoint）**：Tomcat支持多种I/O模型来处理网络请求：BIO(阻塞IO，已废弃)、NIO(非阻塞IO/多路复用，默认)、NIO2(AIO异步IO)、APR(Apache Portable Runtime，通过JNI调本地库)
- **NIO Endpoint工作流程**：包含三大角色：Acceptor线程（负责接收TCP连接并注册到Poller）、Poller线程（负责轮询已注册连接的就绪事件并交给Worker）、Worker线程（线程池，从Poller接收Socket并处理HTTP请求）
- **Acceptor线程**：默认1个，调用ServerSocketChannel.accept()接收新连接，将SocketChannel注册到Poller的事件队列中
- **Poller线程**：默认2个，使用Selector多路复用器轮询已注册连接上的读写就绪事件。当Socket状态就绪时，Poller将其封装为任务提交到Worker线程池
- **Worker线程**：从线程池中获取，负责解析HTTP请求（解析请求行、请求头、请求体），生成ServletRequest/Response对象，调用Engine的Pipeline处理请求，并将响应写回客户端
- **请求处理全流程**：客户端请求→Acceptor接收TCP连接→注册到Poller事件队列→Poller检测到就绪事件→提交给Worker线程池→Worker解析HTTP请求→Engine Valve处理→Host Valve→Context Valve→Wrapper Valve（触发Filter链→Servlet.service()）→响应原路返回→Worker释放回线程池
- **Tomcat线程池**：Tomcat不使用JDK标准ThreadPoolExecutor，而是自己实现（扩展JDK的ThreadPoolExecutor，增加了对核心线程空闲超时的支持）。关键参数：corePoolSize（核心线程）、maxPoolSize（最大线程）、keepAliveTime（空闲线程存活时间）
- **线程池关键参数**：maxThreads（最大工作线程数，默认200）、minSpareThreads（最小空闲线程数，默认10）、maxIdleTime（线程空闲超过该时间被回收，默认60000ms）
- **Connector参数**：port（监听端口，默认8080）、protocol（协议和IO模式，HTTP/1.1即NIO）、acceptCount（等待队列满后最大等待连接数，排队满后拒绝）、maxConnections（最大并发连接数，NIO下默认10000）、connectionTimeout（连接超时ms）
- **请求队列(acceptCount)**：当所有工作线程都忙时，新请求进入操作系统级的等待队列（backlog）；队列满后再有新连接到来则拒绝连接（Connection Refused）。acceptCount即指定这个队列长度
- **类加载器架构**：Tomcat打破双亲委派模型实现Web应用隔离，核心类加载器：Bootstrap（JVM核心类，rt.jar）→ System（CLASSPATH中的类）→ Common（Tomcat自身和所有Web应用共享，${catalina.base}/lib）→ WebappClassLoader（每个Web应用独立，加载/WEB-INF/classes和/WEB-INF/lib）
- **Common/Catalina/Shared加载器**：从Tomcat 6开始，这三个加载器默认都指向${catalina.base}/lib目录（即Common），Catalina和Shared的独立目录在默认配置下是注释掉的，一般只需要Common
- **WebappClassLoader打破双亲委派**：优先从Web应用自身加载类（先搜索/WEB-INF/classes和/WEB-INF/lib），未找到才委托给父加载器。实现Web应用间的类隔离（应用A的lib中的类不会影响应用B）
- **WebappClassLoader的好处**：1) 每个应用使用自己的lib版本，互不干扰；2) 应用可覆盖Tomcat的共享类（如覆盖老版本servlet-api，但这是不推荐的做法）；3) 支持热部署时重新创建类加载器
- **类找不到问题排查**：NoClassDefFoundError（编译时存在，运行时找不到，通常是依赖冲突或容器提供的类版本不对）、ClassNotFoundException（类根本不存在）、排查步骤：检查/WEB-INF/lib中是否有jar、是否有同名不同版本的冲突、是否被父加载器拦截
- **Tomcat调优**：优化线程池参数(maxThreads/minSpare对高并发)、调整acceptCount(请求排队深度)、设置compression(开启GZIP压缩)、调整connectionTimeout、设置maxKeepAliveRequests、JVM参数调优(堆大小/GC策略)
- **压缩配置**：compression="on"、compressionMinSize（最小压缩大小，默认2048字节）、compressableMimeType（可压缩的MIME类型，如text/html,text/xml,text/plain,application/json,application/javascript）
- **maxKeepAliveRequests**：HTTP Keep-Alive模式下，一个TCP连接最多处理的请求数，到达后连接关闭。设置-1为无限。平衡连接复用和资源占用
- **Session管理**：StandardManager（默认，内存存储，重启丢失）→ PersistentManager（持久化到文件或JDBC数据库，重启恢复）→ DeltaManager（集群Session复制，全节点复制，适用于小集群）→ BackupManager（集群Session备份，只备份到一个备份节点，适用于大集群）→ 集成外部存储（推荐用Spring Session+Redis，统一管理）
- **StandardManager细节**：Session存储在ConcurrentHashMap中，定期检查过期Session（每60秒，backgroundProcessorDelay），正常关闭时序列化Session到SESSIONS.ser文件，启动时反序列化恢复
- **集群Session复制**：DeltaManager（全节点复制，每个节点的Session变更同步到所有其他节点，节点多时网络开销大，适合4节点以下）；BackupManager（每个Session选一个备份节点，变更只同步到备份节点，节点故障时从备份节点恢复，适合大集群）
- **Redis Session Manager**：开源方案（如tomcat-redis-session-manager），将Session存储到Redis中，所有Tomcat节点共享Redis的Session，实现无状态、高可用集群；但更推荐Spring Session抽象层统一管理
- **安全Security**：Tomcat通过Realm（领域/用户角色数据源）和Valve（类似Servlet Filter的阀门）实现安全控制
- **Realm种类**：MemoryRealm（tomcat-users.xml，开发用）、JDBCRealm（数据库用户表认证）、JNDIRealm（LDAP目录服务认证）、JAASRealm（JAAS框架认证）、DataSourceRealm（数据源认证，推荐）
- **Valve阀门**：类似Servlet的Filter，在Engine/Host/Context级别拦截请求。常用：AccessLogValve（访问日志）、RemoteAddrValve（IP黑白名单）、SingleSignOn（单点登录SSO）、CrawlerSessionManagerValve（爬虫Session管理）
- **SSL/TLS配置**：Connector中配置SSL证书，使用NIO+OpenSSL（SSLHostConfig），支持HTTPS。参数：keystoreFile（证书路径）、keystorePass（证书密码）、sslProtocol（TLS版本）。生产环境推荐在反向代理层（Nginx）配置HTTPS
- **嵌入式Tomcat**：Spring Boot默认集成tomcat-embed-core系列jar（非传统Tomcat安装版），通过TomcatServletWebServerFactory创建嵌入式Tomcat实例，无需独立安装部署
- **Spring Boot启动嵌入式Tomcat过程**：refresh()→onRefresh()→createWebServer()→TomcatServletWebServerFactory.getWebServer()→实例化Tomcat对象→创建Connector(CoyoteAdapter)→启动Tomcat实例→TomcatWebServer作为一个Bean管理
- **嵌入式Tomcat参数定制**：实现WebServerFactoryCustomizer<TomcatServletWebServerFactory>或配置文件中server.tomcat.*属性，定制线程池、连接超时、URI编码等参数
- **替换嵌入式容器**：移除spring-boot-starter-tomcat依赖，引入spring-boot-starter-jetty或spring-boot-starter-undertow，Spring Boot自动检测并切换
- **Tomcat目录结构**：bin（启动/关闭脚本）、conf（配置文件server.xml/web.xml/tomcat-users.xml）、lib（共享类库）、logs（日志）、webapps（部署的应用目录）、work（工作目录，JSP编译后的Servlet存于此）、temp（临时文件）

# 2. 面试题

## 2.1 Tomcat的整体架构是怎样的？各个组件的作用是什么？
Tomcat采用**分层容器架构**，从外到内依次为：
- **Server**：最顶层，代表整个Tomcat服务器（一个JVM一个），管理Server的启动关闭（start/shutdown端口）
- **Service**：将一组Connector和一个Engine绑定在一起，形成完整的请求处理单元。一个Server可包含多个Service
- **Connector**：连接器，负责接收TCP连接、解析HTTP/AJP等协议请求、封装ServletRequest/Response。支持多种I/O模型（NIO/NIO2/APR）
- **Engine**：唯一，Servlet引擎核心，将请求分发给匹配的Host
- **Host**：虚拟主机（基于域名），一台物理机通过多个Host绑定不同域名服务不同应用
- **Context**：一个Web应用（war包），有独立的类加载器和ServletContext
- **Wrapper**：最细粒度，代表一个Servlet，管理Servlet生命周期

请求流转：Client → Connector(TCP接收、协议解析) → Engine(查找Host) → Host(根据Context路径匹配) → Context(查找Wrapper) → Wrapper(调用Filter链 → Servlet.service())

## 2.2 Tomcat Connector的NIO模型工作流程是怎样的？涉及哪些线程？
NIO模型包含三种线程角色：

1. **Acceptor线程**（默认1个）：
   - 死循环调用`ServerSocketChannel.accept()`接收新TCP连接
   - 将得到的SocketChannel设置为非阻塞模式
   - 封装为NioChannel后注册到Poller的事件队列

2. **Poller线程**（默认2个，`Math.min(2, Runtime.availableProcessors())`）：
   - 维护一个Selector复用器
   - 轮询处理事件队列中新增的通道注册
   - 通过Selector.select()轮询就绪事件（READ/WRITE）
   - 将就绪的Socket封装为SocketProcessor任务提交到Worker线程池

3. **Worker线程**（线程池，默认200）：
   - 从线程池取出执行SocketProcessor
   - 解析HTTP请求（状态机方式解析请求行、请求头、请求体）
   - 生成Request/Response对象
   - 调用CoyoteAdapter将请求交给Engine的Pipeline处理
   - 最终调用Servlet.service()
   - 响应数据写回客户端

## 2.3 Tomcat的类加载机制是怎样的？为什么要打破双亲委派模型？
Tomcat的类加载器层次结构（自顶向下委托方向）：
- **Bootstrap ClassLoader**：加载JVM核心类（rt.jar）
- **System ClassLoader**：加载CLASSPATH中指定的类
- **Common ClassLoader**：加载`${catalina.base}/lib`下的类（Tomcat自身和所有Web应用共享）
- **WebappClassLoader**（每个Web应用独立一个）：加载`/WEB-INF/classes`和`/WEB-INF/lib`下的类

**打破双亲委派的原因**：
WebappClassLoader加载类时**优先自己加载**（在自己的目录中搜索），找不到才委托父加载器。原因：
1. **应用隔离**：不同Web应用可能依赖同一库的不同版本，每个应用的WebappClassLoader独立加载，互不影响
2. **可替换性**：Web应用可以覆盖Tomcat共享库中的类（但不推荐）
3. **热部署**：重新部署时丢弃旧的WebappClassLoader并创建新的，实现类级别的热替换

核心规则：WebappClassLoader先自己加载（打破委派），但如果类属于JVM核心类或Tomcat指定的不能覆盖的包（如javax.servlet.*），则会先委托父加载器。

## 2.4 Tomcat如何实现Web应用之间的隔离？
通过**每个Web应用拥有独立的WebappClassLoader**实现。每个应用都在自己的类加载器中查找/WEB-INF/classes和/WEB-INF/lib中的类。由于不同WebappClassLoader加载的类在JVM中是不同的（即使全限定名相同，不同的ClassLoader实例加载的类也是不同的Class对象），自然实现了隔离。

这种设计带来：
- 应用A使用Spring 4.x，应用B使用Spring 5.x，互不冲突
- 应用A的静态变量修改不影响应用B
- 每个Context有独立的ServletContext、配置、数据源等

## 2.5 知道Tomcat哪几种Session管理方式？各适合什么场景？
1. **StandardManager**（默认）：Session存储在内存的ConcurrentHashMap中，正常关闭时序列化到SESSIONS.ser文件并启动时恢复。单机部署，重启丢失未持久化的Session

2. **PersistentManager**：Session持久化到文件或JDBC数据库，比StandardManager更可靠的持久化，重启可恢复。适合对Session可靠性要求较高的单机场景

3. **DeltaManager**（全节点复制）：Session变更广播到集群所有节点，所有节点都拥有全量Session。适合**小集群（2-4个节点）**，节点越多网络开销越大

4. **BackupManager**（主备复制）：每个Session选择一个备份节点，变更只同步到该备份节点。节点故障时其他节点从备份节点获取Session。适合**大集群**，网络开销小

5. **集成外部存储**（推荐方案）：通过Spring Session将Session存储到Redis/MySQL/MongoDB等外部存储中。Tomcat节点完全无状态，水平扩展随意，支持跨多种容器（Tomcat/Jetty/Undertow）统一管理。生产环境最常用

## 2.6 Tomcat如何调优？有哪些关键参数？
**线程池调优**：
- `maxThreads`（默认200）：最大工作线程数，经验公式：CPU密集型N+1，IO密集型2N（N=CPU核数）。可根据实际QPS和响应时间动态调整
- `minSpareThreads`（默认10）：最小空闲线程数，保证一定预热
- `maxIdleTime`（默认60000ms）：线程空闲超时回收

**连接器调优**：
- `acceptCount`（默认100）：请求等待队列深度，所有线程忙时新请求进入排队，超过后拒绝
- `maxConnections`（NIO默认10000）：最大并发连接数，达到后新连接被接受但不处理（等超时）
- `connectionTimeout`（默认60000ms）：连接建立后等待请求数据的超时时间
- `maxKeepAliveRequests`（默认100）：一个长连接最多处理请求数，防止一个连接占用过久

**其他优化**：
- `compression="on"`：开启GZIP压缩，text/html等文本类型压缩率可达70%+，降低带宽
- `compressionMinSize="2048"`：小于该大小的响应不压缩（太小压缩意义不大）
- 禁用AJP连接器（不使用Apache httpd反向代理时）
- JVM层面：设置合理堆内存、GC策略（G1GC生产推荐`-XX:+UseG1GC`）
- `autoDeploy="false"`：生产环境关闭自动部署，避免性能开销和安全风险

## 2.7 Spring Boot是如何启动嵌入式Tomcat的？
流程：
1. `SpringApplication.run()`中调用`refresh()`方法刷新IoC容器
2. `ServletWebServerApplicationContext`的`onRefresh()`方法中调用`createWebServer()`
3. 通过`ServletWebServerFactory`的实现（`TomcatServletWebServerFactory`）创建WebServer
4. `TomcatServletWebServerFactory.getWebServer()`：
   - 实例化`org.apache.catalina.startup.Tomcat`对象
   - 创建Connector并设置端口、协议等参数
   - 创建Engine、Host、Context（StandardRoot）
   - 注册Servlet、Filter、Listener到Context
   - 调用`tomcat.start()`启动Tomcat
5. 返回`TomcatWebServer`实例，注册为Spring容器中的一个Bean
6. 应用停止时调用`TomcatWebServer.stop()`关闭Tomcat

可以通过`WebServerFactoryCustomizer`或`application.properties`中的`server.tomcat.*`属性定制嵌入式Tomcat的行为。

## 2.8 如何替换Spring Boot的嵌入式Tomcat为Jetty或Undertow？
1. 在pom.xml中排除tomcat依赖并引入目标容器：
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
```
2. Spring Boot自动检测classpath中的Servlet容器实现（Tomcat→Jetty→Undertow按优先级），不需要额外配置

选择建议：Tomcat（全面稳定，用户基数最大）、Jetty（轻量级，适合嵌入式场景）、Undertow（高性能，非阻塞，使用较少的系统资源）

## 2.9 Tomcat的maxThreads和acceptCount分别控制什么？高并发场景如何设置？
- **maxThreads**：工作线程池的最大线程数，同时处理请求的线程上限。所有线程都忙时新请求进入等待队列。默认200
- **acceptCount**：等待队列的长度。所有线程都忙且等待队列已满时，新连接被拒绝（Connection Refused）。默认100

**高并发场景设置思路**：
- 通过压测确定单机QPS和平均响应时间
- 根据Little's Law：并发数 ≈ QPS × 平均响应时间(s)
- maxThreads应该大于计算出的并发数
- acceptCount不宜过大（请求在队列中等待太久会超时），也不宜过小（高峰流量直接拒绝）
- 但maxThreads不是越大越好，线程数过多会导致：频繁上下文切换、内存消耗增大、OOM风险。IO密集型适当增大（2N），CPU密集型不宜过大（N+1）

## 2.10 Tomcat的Valve是什么？与Servlet的Filter有什么区别？
**Valve**是Tomcat容器级别的拦截器，工作于Engine/Host/Context层面，通过Pipeline机制在容器间传递请求。

**与Filter的区别**：
| 维度 | Valve | Filter |
|------|-------|--------|
| 工作层面 | Tomcat容器级别 | Servlet规范级别 |
| 适用范围 | Tomcat特有，不是Java EE标准 | Java EE标准，跨容器通用 |
| 作用域 | 可作用于Engine/Host/Context | 作用于特定URL模式 |
| 配置方式 | server.xml中配置 | web.xml或@WebFilter注解 |
| 性能 | 直接操作Tomcat内部对象，略高 | 通过标准ServletRequest/Response |

常用Valve：AccessLogValve（访问日志）、RemoteAddrValve（IP黑白名单）、SingleSignOn（SSO票据验证）

## 2.11 Tomcat的APR是什么？什么场景使用？
**APR (Apache Portable Runtime)** 是通过JNI调用本地C/C++库处理网络请求的方式。相比纯Java NIO：
- 性能更高（本地代码处理SSL、压缩等）
- 支持OpenSSL（比JSSE更高效）
- 内存占用更小
- 但需要安装本地库（apr、openssl）

使用场景：对性能要求极高的场景、大量的SSL/TLS加密通信。但维护成本较高（需要操作系统级别安装依赖库），一般场景下NIO 2/NIO 已经足够，APR在现在不是必需品。

## 2.12 如何排查Tomcat中ClassNotFoundException/NoClassDefFoundError问题？
1. **确认类是否存在**：检查/WEB-INF/lib/中是否有包含该类的jar包；检查/WEB-INF/classes/下是否有对应class文件
2. **判断是哪个ClassLoader的问题**：
   - 如果类应该由WebappClassLoader加载（应用自己的类），检查jar是否打包进了war
   - 如果类应由Common ClassLoader加载（Tomcat或共享库），检查${catalina.base}/lib下是否有该jar
3. **检查版本冲突**：`mvn dependency:tree`或直接在IDE查看依赖树，确认是否有多个版本的jar被引入
4. **检查是否被上级加载器拦截**：某些包（如javax.servlet）会被双亲委派机制先由父加载器加载，如果父加载器没有但WebappClassLoader有，需要确认是否正常
5. **ClassNotFoundException vs NoClassDefFoundError**：
   - CNFE：编译时类不存在或动态加载时找不到（如Class.forName）
   - NoClassDefFoundError：编译时存在，运行时初始化失败（如静态初始化报错或类被加载器加载失败），检查该类的静态依赖是否完整

## 2.13 Tomcat如何实现热部署？
热部署指的是**不重启整个Tomcat**而重新加载单个Web应用。实现原理：
- 监听/WEB-INF/web.xml和/WEB-INF/lib目录的变化（backgroundProcessor线程定期检查）
- 检测到变化后，销毁当前Context并重建：
  1. 停止并销毁所有Servlet实例
  2. 丢弃旧的WebappClassLoader
  3. 创建新的WebappClassLoader
  4. 重新加载/WEB-INF/classes和/WEB-INF/lib中的类
  5. 重新初始化Servlet

注意事项：
- 旧ClassLoader可能不会立即被GC回收（如果有线程持有它的引用），可能导致OOM（PermGen/Metaspace泄漏）
- 热部署在开发环境有用，**生产环境应关闭autoDeploy="false"**（防止误操作和性能问题）

## 2.14 如何防止Tomcat内存泄漏？
常见原因和解决方案：
1. **热部署导致Metaspace泄漏**：重部署时旧WebappClassLoader未被回收，其加载的类驻留在Metaspace中。排查：是否有ThreadLocal持有WebappClassLoader加载的类的引用；是否有JDBC驱动注册在DriverManager中未注销
2. **ThreadLocal泄漏**：线程池中的线程长期存活，如果ThreadLocal没有调用remove()，其引用的对象（可能来自已卸载的应用）无法被回收。解决：使用后务必finally中ThreadLocal.remove()
3. **连接未释放**：数据库连接、HTTP连接等在finally中关闭
4. **静态集合**：ConcurrentHashMap等静态集合存储大量对象不清理
5. **Session过多**：合理设置session过期时间，使用Redis替代内存存储

排查工具：内存dump分析（jmap + MAT / JProfiler）、GC日志分析、JVM监控（JConsole/VisualVM）
