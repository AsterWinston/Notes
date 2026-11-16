本篇覆盖 Spring Cloud 与微服务体系的高频面试题，按"主题块 → 详细分块 → 题目 → 解答"组织。五个主题块：微服务架构理论、Spring Cloud 体系、服务治理核心机制、配置与可观测、架构演进与落地。各组件深水区另成篇章：注册与配置细节见 17_nacos.md，Feign 原理见 18_feign.md，网关见 19_spring_cloud_gateway.md，流量治理见 20_sentinel.md，分布式事务见 21_seata.md、34_distributed_theory.md。

# 1. 微服务架构理论

## 1.1 架构选择

### 1.1.1 微服务相比单体有哪些优缺点？什么团队不该上微服务？

**优点（拆的收益）**：
1. **独立部署**：单服务独立构建/测试/发布，发布频率高、爆炸半径小；
2. **独立伸缩**：按服务粒度扩缩（订单扩容、报表不动），资源利用率高（与 5_tomcat.md 4.1.2"单机调参边际收益递减、水平扩展才是主路线"互为因果）；
3. **故障隔离**：一个服务挂掉不拖垮全站（前提是加了熔断隔离，20_sentinel.md，否则是级联雪崩）；
4. **技术异构与团队自治**：按域选型、两个披萨团队端到端负责——**康威定律的正向用法**：架构与组织对齐。

**缺点（拆的账单）**：
1. **分布式复杂度**：网络延迟/丢包成为常态，超时、重试、幂等、熔断全要显式设计（3.2.3、13_rocketmq.md 3.2.3）；
2. **数据一致性**：单机 ACID 换成分片自治 + 最终一致——Saga/TCC/事务消息各方案（21_seata.md）；
3. **运维门槛**：CI/CD、容器编排、注册配置、监控告警、链路追踪**一样不能少**——没有基础设施的团队上微服务等于裸奔（4.2 与 33_cicd.md 是前置条件，不是售后配件）；
4. **调试定位难**：一次请求穿十个服务，没有 traceId 体系寸步难行（4.2.1）；
5. **粒度错了两头不讨好**：拆太碎 → "分布式单体"（发布必须整体协调、调用链漫长）；拆太粗 → 单体优势没丢、分布式成本全交。

**"不该上"的判断（这题的区分度）**：团队 <10 人、业务边界未稳、没有自动化部署与观测能力、QPS 用单体能扛——**先模块化单体（Spring Modulith 思想：包边界+模块测试，将来拆得起）**，演进式拆分胜过推倒重来。面试收口句："微服务解决的是**组织扩展**问题，代价是**技术复杂度**——它不是'更先进的架构'，是'规模逼出来的权衡'。"

## 1.2 拆分与设计

### 1.2.1 微服务的粒度/边界如何确定？

**主流方法论 = DDD 限界上下文**：每个上下文有独立的领域模型与语言（"订单"在交易域和履约域含义不同），**上下文边界 = 服务边界**；识别手段：事件风暴（命令→事件→聚合）、子域分类（核心/支撑/通用域决定投入与拆并优先级）。

**四条工程校验（拆完拿它们体检）**：
1. **数据自治**：该服务独占自己的表，跨服务无 join、无共享库（1.2.3）——做不到就是拆错了；
2. **变更频率/伸缩画像**：高频变更、独立伸缩需求的模块才值得独立——把"常一起改的"拆到两个服务，集成成本翻倍；
3. **可独立测试发布**：起一个服务能完成功能闭环测试；发布不需要"先升 A 再升 B"的时序耦合；
4. **团队匹配**：一个服务 ≤ 一个团队负责到底（含值班）——服务数远大于团队数的拆法必然失控。

**反面信号**：调用链上"三个服务永远一起出现"（该合并）；一个字段改动牵五个服务（边界切在模型中间了）；服务间大量"透传型接口"（中间层是空气）。**2026 口径**："微服务"热度已过峰值，共识转为**适度拆分**——单体优先、按域演进、模块单体（Modulith）是默认起点，拆是最后手段不是第一选择（5.2.1 展开）。

### 1.2.2 服务拆分后会出现哪些新问题？Spring Cloud 分别用什么应对？

**问题→方案映射表**（这题价值在"系统性"，背组件名不算懂）：

| 新问题 | 本质 | 应对 | 组件落点 |
|--------|------|------|---------|
| "A 调 B，B 在哪" | 地址动态化 | 服务注册与发现 + 客户端负载均衡 | Nacos/Eureka + LoadBalancer（3.1、3.2） |
| 入口统一 | 对外只暴露一个域名 | API 网关：路由/鉴权/限流 | Gateway（19_spring_cloud_gateway.md）、Nginx 更外层（12_nginx.md） |
| 故障传播 | 超时堆积拖垮调用方 | 超时预算 + 熔断降级 + 隔离 | Sentinel/Resilience4j（3.2.3、20_sentinel.md） |
| 配置漂移 | N 个实例配置要一致下发 | 分布式配置中心 + 动态刷新 | Nacos Config/Apollo（4.1） |
| 跨服务排障 | 一次请求散在 N 台机器 | 链路追踪 + 指标 + 集中日志（三支柱） | Micrometer Tracing/SkyWalking（4.2） |
| 数据一致性 | 本地事务跨不出服务 | 最终一致：事件驱动/Saga/TCC/事务消息 | MQ（13_rocketmq.md）、Seata（21_seata.md） |
| 服务间鉴权信任 | 谁在调我、带着谁的身份 | 统一凭证：JWT 透传/内部签名头 | 11_spring_security.md 4.3.1 |
| 发布风险 | 改一个服务也可能全站炸 | 灰度/蓝绿/滚动 + 可观测门禁 | 12_nginx.md 3.3.1、33_cicd.md |

**易错**：把 Spring Cloud 当"微服务本身"——它是**这些分布式模式的标准件货架**，模式先于框架存在（Consul/K8s 体系解决同一批问题，5.1.2）；也注意反向：不是每个问题都要上组件，小系统"注册中心 + 网关 + 日志集中"三件套起步即可。

### 1.2.3 微服务的数据层如何设计？数据库能共享吗？

**原则：Database-per-Service**——服务独占数据（至少独占 schema 与迁移权），跨服务共享库表是**头号反模式**："改一列要通知五个服务"直接把微服务拆回单体耦合；数据库才是最难拆、拆错了最疼的部分。

**由此派生的三个设计决策**：
1. **跨服务查询组合**：API 组合（调用方并行拉多个服务再拼装，网关聚合层常见——19_spring_cloud_gateway.md）；或**数据冗余到读模型**：CQRS——变更走命令服务，查询走异步重建的宽表/ES（binlog 订阅是标准搬运工，14_canal.md 1.1.1）；分页深聚合场景优先考虑后者，"实时报表全靠 join RPC"是性能灾难；
2. **跨服务写的一致性**：放弃强一致幻想——领域事件 + MQ 最终一致（13_rocketmq.md 2.3.1 事务消息正是为此而生）、必要时 Saga 编排/编舞补偿（21_seata.md）；**分布式事务是兜底不是常态**，出现频率高说明边界拆错；
3. **数据所有权与访问语义**：跨服务只许走 API 不许直连库（哪怕"就查一下"）；同服务内部才允许 join；历史遗留"读对方库"要立还债计划。

**面试延伸**：分库分表在微服务语境的位置——服务拆分按**业务域**垂直切，分库分表按**数据量**水平切（22_sharding_sphere.md），两者正交：一个订单服务自己的库仍可能需要分片。"微服务拆完每个服务单库还扛得住吗"——先读扩展（缓存/副本）、再写优化（分片）、最后才是多写架构，顺序别答反。

# 2. Spring Cloud 体系

## 2.1 框架定位与版本

### 2.1.1 Spring Cloud 是什么？组件版图怎么组织？

**定位**：Spring 对**分布式系统常见模式**的工具集总集——不是单一框架，而是"一系列项目 + 版本列车（Release Train）BOM + starter 体验"的组合货架（2.1.2）：好开箱体验覆盖典型场景，扩展点覆盖其余。

**按职责读版图**（2026 现役 + 各归属篇章）：

| 职责 | 现役组件 | 深水区 |
|------|---------|--------|
| 注册发现/元数据 | Spring Cloud Commons 的 DiscoveryClient SPI：接 Nacos/Eureka/Consul/K8s | 17_nacos.md |
| 配置管理 | Config、Bus、Nacos Config | 4.1 |
| 服务调用 | OpenFeign；函数式 WebClient | 18_feign.md |
| 负载均衡 | Spring Cloud LoadBalancer | 3.2.2 |
| 网关路由 | Spring Cloud Gateway（响应式 + WebMVC 双形态） | 19_spring_cloud_gateway.md |
| 熔断容错 | Spring Cloud Circuit Breaker（抽象）+ Resilience4j/Sentinel 实现 | 20_sentinel.md |
| 消息驱动 | Spring Cloud Stream（Kafka/RabbitMQ binder） | 31_kafka.md、37_rabbitmq.md |
| 可观测 | Micrometer（指标）+ Micrometer Tracing（链路） | 4.2 |
| 其他 | Function/Task（FaaS/批）、Kubernetes 集成 | — |

**关键认知**：Commons 定义的**SPI 层**（DiscoveryClient、LoadBalancerClient、ConfigData API）才是 Spring Cloud 的真身——各注册中心/配置中心是**可插拔实现**（Boot 3 后配置侧统一走 `spring.config.import`）。所以"Spring Cloud 换注册中心要换全家桶"是误解，接口不变、starter 换一个即可（17_nacos.md）。

### 2.1.2 Spring Cloud 版本如何命名？和 Boot 怎么配对？演进关键节点？

**命名史**：Hoxton 及以前用伦敦地铁名当版本号（Dalston→Edgware→…→Hoxton）；**2020.0 起改日历版本 `YYYY.MINOR.PATCH`**（与 Boot 同规则），地铁名降级为别名延续至今（官方兼容表 2026 口径）：

| Release Train | 别名 | 对应 Boot |
|---------------|------|-----------|
| **2025.1.x** | Oakwood | **4.0.x / 4.1.x**（自 2025.1.2） |
| 2025.0.x | Northfields | 3.5.x |
| 2024.0.x | Moorgate | 3.4.x |
| 2023.0.x | Leyton | 3.2.x / 3.3.x |
| 2022.0.x | Kilburn | 3.0.x / 3.1.x |
| 2021.0.x | Jubilee | 2.6.x / 2.7.x |
| 2020.0.x | Ilford | 2.4.x / 2.5.x |
| Hoxton | — | 2.2.x / 2.3.x |

**硬约束**：Spring Cloud 列车与 Boot 版本**必须按表配对**（先定 Boot 再取对应 BOM，`spring-cloud-dependencies` import 进 dependencyManagement；官方推荐 Initializr/兼容表核对，错配的典型症状是自动配置类找不到/NoSuchMethod）。

**演进四节点（背脉络而不是背清单）**：
1. **Netflix 套件退场（2020.0 起）**：Hystrix/Ribbon/Zuul1 维护模式——Hoxton 是末代全家桶，之后熔断交给 Circuit Breaker + Resilience4j/Sentinel、负载交给 SC LoadBalancer；
2. **2022.0 = Jakarta 代**：基座 Boot 3/Framework 6（10_springboot.md 3.2.3 的 javax→jakarta 在微服务侧同步发生）；**Sleuth 停更 → Micrometer Tracing**（4.2.1）；
3. **2023.0 起 Gateway 双形态**：新增 WebMVC 版（spring-cloud-starter-gateway-server-webmvc）——响应式不是唯一门票；
4. **2025.1 = Boot 4/Framework 7 代**（2025.11 Boot 4 GA，模块化重构、去遗留 API）。

**Alibaba 侧口径**：Spring Cloud Alibaba 发版节奏滞后官方列车——2022.x 对齐 Boot 3.0、2023.x 对齐 3.2.x；**Boot 3.5/4.0 的 SCA 适配版本以官方版本说明为准**，面试报"看 SCA releases 对齐表"的严谨态度比背错数字值钱。

### 2.1.3 Spring Cloud、Spring Cloud Alibaba、Dubbo 怎么选？

**先分清三层**："Spring Cloud"是模式货架（语言无关的概念），**Spring Cloud（官方列车）**与 **Spring Cloud Alibaba**是同一货架的两家供货——SCA 不是替代而是**补充+组合**（它按 Commons 接口做实现：Nacos 实现 DiscoveryClient/ConfigData，Sentinel 接 Circuit Breaker）。

**选型对照**：

| 维度 | 官方列车（Netflix/Consul 血统组件） | Spring Cloud Alibaba | Dubbo |
|------|-------------------------------------|---------------------|-------|
| 代表栈 | Eureka（式微）+ Config/Bus + Gateway + Resilience4j | Nacos + Sentinel + Seata + Dubbo-F 可选 | RPC 框架为中心（3.x Triple 协议、应用级发现） |
| 通信形态 | REST/HTTP 为主 | REST 为主，生态同 | 二进制 RPC，内网吞吐/延迟优 |
| 注册中心 | 多样 | Nacos（注册+配置一体） | 注册中心可复用 Nacos/ZK |
| 成熟度（2026） | 官方列车稳定，但"全家桶感"下降（Eureka/Config 老化） | 国内活跃主力；Boot 新代际适配有滞后期 | Dubbo 3 + Triple/_mesh 友好_，跨语言改进后仍在 Java 内网场景强势 |
| 组合关系 | **常常混搭**：网关用 Gateway/SCG、治理用 Sentinel、RPC 用 Dubbo、对外 REST | | |

**Dubbo vs Spring Cloud 调用层**（这半题必考）：HTTP/REST——通用、易调试、跨团队/跨语言零门槛、网关生态现成；RPC（Dubbo/gRPC）——序列化紧凑 + 长连接多路复用，**内网高频调用省 CPU 与延迟**（对比账见 18_feign.md 与 31_kafka.md 生态里的 gRPC 讨论）。**2026 实践口径**：对外边界 HTTP + 对内热点链路 RPC/gRPC 混合是常态；Dubbo 3 的应用级服务发现与 Triple 协议正是向 K8s/mesh 靠拢（5.1.2）。

**收口**：没有"哪个框架先进"，只有"组织现状"——存量 Dubbo 继续 Dubbo 治理三件套；新建 Java 云原生：**SCA 为默认（Nacos+Sentinel+Seata 一体化省事）、官方列车按需补位（Gateway/OpenFeign）**；多语言/重度 K8s：把"注册发现/熔断"让位给平台，Spring Cloud 退居配置与调用封装（5.2.1 展开）。

## 2.2 服务间通信

### 2.2.1 服务之间有哪几种通信方式？怎么选？

**按"同步/异步 × 协议"摆谱**：

| 方式 | 协议/载体 | 形态 | 适用 |
|------|----------|------|------|
| **REST（HTTP/1.1/2）** | JSON over HTTP | OpenFeign/WebClient 调用 | 默认选择：跨团队、跨语言、经网关对外 |
| **RPC** | 二进制 TCP（Dubbo 协议/gRPC=HTTP2+Protobuf） | 接口桩编译期契约 | 内网高频调用、强契约、多语言 gRPC |
| **消息（异步）** | MQ（13_rocketmq.md） | 事件驱动解耦 | 削峰、跨时间解耦、最终一致（1.2.3） |
| **GraphQL/WebSocket/SSE** | HTTP 族 | 按需聚合/服务端推 | BFF 聚合（GraphQL）、长连接推送（36_websocket.md） |

**三个决策维度**：① **同步阻塞还是异步解耦**——能异步就异步（流程长、允许最终一致的动作交给 MQ），同步链只留给"用户此刻就要结果"；② **契约强度**——IDL/接口桩（gRPC/Dubbo/Feign 共享 jar）防"字段悄悄改名"事故，纯 JSON 裸 HTTP 跨团队慎用；③ **性能账**——内网 P99 敏感链路：gRPC/二进制序列化 > HTTP/2 > HTTP/1.1+JSON，但"性能差一档"很少推翻"生态与团队熟悉度"，**先 REST 起步、热点再 RPC** 是务实路径。

**易错**：把"通信方式"答成"选哪个框架"——先给维度框架再报组件名，答题层次立刻不同；服务间通信与"数据同步"（binlog 订阅拉平冗余，14_canal.md）是两种范式：前者拉答案、后者追变化，架构上互补。

### 2.2.2 Java 侧 HTTP 客户端怎么选？RestTemplate、WebClient、Feign 什么关系？

**代际与形态**：
- **RestTemplate**：同步阻塞老前辈——Boot 仍可用但进入维护模式（官方指引新项目转 WebClient 或声明式接口；**Boot 4 里已从类路径默认移除、需显式引入 spring-web 的独立模块**——2026 口径要报得出这条变化）；
- **WebClient**（Spring WebFlux 系）：响应式/函数式 API，Mono/Flux、背压友好；纯阻塞业务里用 `.block()` 属于"两头不占"；
- **OpenFeign**：**声明式接口 + 注解**，底层 Client 可插（默认 JDK HttpURLConnection → OkHttp/Apache HttpClient 5 或 WebClient bridge）；与 LoadBalancer/CircuitBreaker 集成最顺（18_feign.md）；
- **Spring 6/Boot 3 新秀：HTTP Exchange（@HttpExchange）**——声明式接口不再绑 Feign，WebClient/RestClient 均可驱动，编译期无动态代理开销；配合 **RestClient**（同步 API 现代化，替代 RestTemplate 的正解）出现。

**选型口径**：同步业务、要"像调本地方法"→ OpenFeign（生态最成熟）或 @HttpExchange + RestClient（更轻、少一层的演进方向）；响应式链路（WebFlux/网关内）→ WebClient；裸调一次性接口 → RestClient。追问"为什么 Gateway 里不用 Feign"——SCG 是 WebFlux 栈，阻塞客户端会卡 event loop（19_spring_cloud_gateway.md）。

# 3. 服务治理核心机制

## 3.1 服务注册与发现

### 3.1.1 服务注册与发现的基本原理？客户端注册和服务端注册有何不同？

**最小闭环三角色**：提供者启动时把"我是谁（服务名）+ 我在哪（IP:Port）+ 我的元数据（版本/ Zone/权重）"登记到**注册中心**；消费者按服务名**查询实例列表**（本地缓存 + 定期增量刷新）；健康靠**心跳续约**（典型 5s~30s 一报，超时阈值后剔除）或平台探测。核心矛盾：**注册表的实时性 vs 稳定性**——剔除太快误杀、太慢把流量送给死人（3.1.2 各家在这条轴上的取舍）。

**两种范式**（按"谁来感知实例健康"分）：
1. **客户端注册/自注册**（Eureka/Nacos/Consul agent）：应用 SDK 自己上报心跳——好处：语义丰富（权重/灰度标可随实例上报）、跨平台（不依赖 K8s）；代价：应用要"知道自己在哪"（容器环境 IP 探测要配）、SDK 侵入每语言一份；
2. **服务端注册/第三方代理**（K8s Service + Endpoints/EndpointSlice、Consul catalog 外灌）：**平台**替你登记——Pod ready 探针即健康真相、DNS/虚拟 IP 完成寻址、应用零感知；这是云原生默认答案（5.1.2、32_kubernetes.md）。

**消费侧两种寻址**：客户端负载均衡（消费者拿全量列表自己选实例，3.2.1——Spring Cloud 路线）vs 服务端负载均衡（消费者只认识一个 VIP/DNS，代理层选实例——K8s Service/Nginx/SLB 路线）。"Spring Cloud 注册发现 + LoadBalancer"是前者的完整样板。

**追问**："注册中心和 DNS 什么区别"——DNS 也是服务发现（粗粒度、TTL 缓存、无权重/元数据），K8s CoreDNS 把两者缝合；注册中心胜在**实时剔除与富元数据**（路由/灰度靠它），两者在 mesh/平台化里边界正越来越糊（5.1.2）。

### 3.1.2 Nacos、Eureka、Consul、ZooKeeper 做注册中心怎么选？

**先立判据**（CAP 与运维模型是表，这四家真正差在**定位**）：

| 维度 | Nacos 2.x/3.x | Eureka 2.x | Consul | ZooKeeper |
|------|--------------|------------|--------|-----------|
| CAP 倾向 | **AP/CP 可切**：临时实例 Distro(AP)、持久实例 Raft(JRaft,CP) | AP | CP（Raft） | CP（ZAB） |
| 健康模型 | 客户端心跳（临时实例）+ 服务端探测（HTTP/TCP） | 客户端心跳 + 自我保护 | Agent 本地探测 + Serf  gossip | 会话心跳 + 临时节点 |
| 通信/推送 | 2.x 起 **gRPC 长连接**（推模式变更推送，1.x 是 UDP） | 客户端定时拉 | 拉/流均有 | Watcher 推 |
| 多 DC | 支持（分组/单元化） | 弱 | **原生 WAN 联邦** | 不支持 |
| 配置中心 | 内置一体 | 无 | 内置 KV | 能用但痛苦 |
| 定位现状（2026） | 国内 Java 云主流首选 | Netflix 停更后存量维护；国内平台少见新增 | 非 Java 栈/多 DC 场景活跃 | **专职协调服务**，当注册中心是历史姿势 |

**两个高频追问点**：
1. **Eureka 自我保护**：15 分钟窗口内**实际收到的续约低于预期的 85%**（阈值 `renewal-percent-threshold`）→ Server **停止批量剔除**并保全集——AP 哲学的极端体现："宁可放过死实例，不可错杀活实例"（网络分区时错杀=全站雪崩）；副作用是消费端短时间拿到死地址——靠客户端重试/熔断兜（3.2.3）。Nacos 的思路近似（阈值剔除 + 保护模式开关）；
2. **ZK 为什么不配当注册中心**：CP 语义下分区期**拒绝写入**（新实例注册不上）、会话抖动临时节点批量消失引发"注册表风暴"；ZAB 过半写放大的运维成本与"地址表高频变更"的画像也不合——ZK 的主场是**分布式锁/选主/协调**（2_jvm/SPI 之外它在 Kafka ZK 时代、Canal HA 里都只干协调）。

**口径**：新建 Java 微服务 → Nacos（注册+配置一体、控制台可用、gRPC 推模式时效好，细节 17_nacos.md）；多 DC/异构栈公司 → Consul；纯 K8s 平台化团队 → 直接用 K8s Service，注册中心都不上（5.1.2）；Eureka 只存在于存量维护，**答"选型"时报"不建议新增使用"即可显时效感**。

## 3.2 负载均衡与流量容错

### 3.2.1 客户端负载均衡和服务端负载均衡有什么区别？微服务里负载均衡发生在哪几处？

**分界一句话**：**谁持有实例列表并做选择决策**——客户端 LB：实例列表下发到调用方进程内，选择逻辑在调用方（LoadBalancer/Feign/Dubbo）；服务端 LB：调用方只打一个固定入口（VIP/DNS/网关），选择逻辑在中间代理（Nginx/LVS/K8s kube-proxy/SLB/云 ALB）。

**微服务请求全链路其实过了好几层 LB**（把这张"层次表"讲出来就超纲得分）：

```text
DNS/GSLB →（云 LB/Nginx 12_nginx.md 3.1.1）→ 网关（SCG 19_spring_cloud_gateway.md）
→ 应用内客户端 LB（本服务调下游，3.2.2）→（可选）Sidecar（Envoy，5.1.2）→ 目标实例
```

**各自优劣**：客户端 LB——**少一跳**、感知服务语义（权重/元数据路由/同 Zone 优先）、与注册中心联动实时；代价是**每语言一份实现**、故障剔除逻辑散在调用方。服务端 LB——语言无关、策略集中易治理；代价是多一跳、传统代理不懂服务元数据（做不了按标签路由——mesh 就是来补这一刀的）。**趋势判断**：K8s/mesh 化团队把决策权收编回基础设施（"客户端 LB 太重、服务端 LB 太盲"的折中）；Java 全家桶则客户端 LB 仍是主力——两条路线并存，选型看平台战略而非性能。

### 3.2.2 Spring Cloud LoadBalancer 的工作原理？和 Ribbon 有何区别？

**背景**：Ribbon 2018 年进维护模式，2020.0 起被官方剔除——SC LoadBalancer 是继任者（阻塞 + 响应式双栈原生）。

**工作流程**（一次 `http://order-service/api/orders` 调用的解剖）：
1. Feign/RestTemplate/WebClient 挂上 LoadBalancer 拦截器，识别服务名形式的 URI；
2. `BlockingLoadBalancerClient` 向 **ServiceInstanceListSupplier** 要实例列表（底层是 DiscoveryClient/Nacos 查询 + **Caffeine 缓存**，缓存刷新周期配合注册中心推送）；
3. `ReactorLoadBalancer<ServiceInstance>` 选择一个实例——默认 **RoundRobinLoadBalancer**（轮询，支持加权/同 Zone 过滤的 supplier 装饰链：AvailabilityZone → Health-check(可选) → 权重）；
4. 服务名替换成真实 `ip:port` 发出请求；失败处理交给上层重试/熔断（它自己不含重试语义）。

**与 Ribbon 对照**：算法面 Ribbon 丰富（Rule 族）vs SCLB 克制（轮询 + 随机 + 自定义 Bean，`@LoadBalancerClient`/配置类替换）；Ribbon 老式配置（`{clientName}.ribbon.*`）vs SCLB 标准 Spring 配置（`spring.cloud.loadbalancer.*`）；Ribbon 无响应式 vs SCLB 全栈；健康检查 Ribbon 内置定时 Ping vs SCLB 委托服务发现组件/可选 health supplier。**迁移要点**：自定义 IRule → 实现 `ReactorLoadBalancer` 或 supplier 装饰；粘性会话/一致性哈希等高级策略需自补（对比 Nginx 侧算法清单 12_nginx.md 3.1.1——应用层 LB 的能力面普遍比代理层薄）。

### 3.2.3 限流、降级、熔断是什么关系？微服务里这套能力归谁？

**三个动作的精确分工**（用"谁在防谁"讲最清楚）：
- **限流（入口自保）**：控制**进来的量**超过自身容量——主动预防，超阈值快速拒绝/排队（令牌桶 12_nginx.md 3.2.1、QPS/并发线程数 20_sentinel.md）；
- **熔断（调用止损）**：控制**出去的对坏依赖的调用**——异常比例/慢调用比例触发 OPEN，快速失败给下游喘息（半开试探恢复）；它防的是级联雪崩：A 等 B 的超时线程堆满 A，A 的上游 C 再等 A——一根链全冻住；
- **降级（业务妥协）**：控制**体验与核心功能的取舍**——限流拒绝和熔断打开都需要"然后给什么"：缓存旧值/默认值/功能裁剪/排队异步化。**限流熔断是机制、降级是策略**；降级还可以由容量规划主动触发（大促关评价、关推荐）。

**协同顺序**：入口限流 → 调用带超时预算 → 超时/异常率触发熔断 → 三者都落到降级响应——一句话："**拦流量、断故障、留底线**"。

**能力归属（微服务语境）**：治理 SDK 层（Sentinel/Resilience4j，进程内决策、与注册中心元数据联动）、网关层（19_spring_cloud_gateway.md 的 RequestRateLimiter、20_sentinel.md 网关适配）、入口代理层（Nginx limit_req）、平台层（mesh/K8s，5.1.2）——**多层并存时以最外层粗限 + 进程内细熔默认为组合**（12_nginx.md 3.2.1 的单实例视角问题在 20_sentinel.md 集群限流解）。深答归 20_sentinel.md 各题，本篇到此为止。

# 4. 配置与可观测

## 4.1 配置管理

### 4.1.1 为什么要配置中心？Nacos Config、Apollo、Spring Cloud Config 怎么选？

**没有配置中心的痛**（三支柱叙事）：改一个超时值要重新打包/重启 N 个服务；环境×实例×密钥的配置矩阵手工维护必然漂移（"测试好、生产炸"头号根因）；配置变更无审计、无法灰度、出事无法秒级回滚。**配置中心 = "运行中应用的外部化配置 + 变更即生效 + 治理三合一"**。

**三家对照**（2026 口径）：

| 维度 | Spring Cloud Config | Nacos Config | Apollo |
|------|--------------------|--------------|--------|
| 存储 | Git/Vault/JDBC | 内嵌 derby/MySQL 集群 | MySQL |
| 生效通道 | 拉 + **Bus 广播**（/actuator/busrefresh，MQ 转发事件） | **gRPC 长连接推送**（1.x 为 HTTP 长轮询，4.1.3） | HTTP 长轮询通知 |
| 灰度 | Profile/Label 弱 | Beta/分批灰度内置 | 灰度发布原生强 |
| 治理 | 无界面（Git 即审计） | 控制台：版本回滚/监听查询/命名空间隔离 | **企业级：审批流/权限/审计/多集群环境** |
| 部署 | 轻（Config Server+Git） | 轻（与注册一体） | 重（Portal/Admin/Config Service 多组件） |
| 适配 | Boot 3 `spring.config.import: configserver:` | SCA 全家桶、国内主流 | 独立体系、Java/.NET 客户端成熟 |

**选型**：已用 Nacos 做注册 → 配置一锅端（一个控制台两件事，**国内默认答案**）；大企业强治理诉求（审批/合规/多环境矩阵）→ Apollo；"配置本来就 Git 管、团队小" → Config + Bus 够用（但动态刷新链路要养 MQ——**运维复杂度被低估的老坑**，Bus 广播丢失=实例配置分裂，4.1.2）。密钥类配置走 Vault/KMS 思路，不进通用配置明文。

### 4.1.2 @RefreshScope 动态刷新的原理？

**两条生效路径**（先分清再谈原理，这题一半人答混）：
1. **@ConfigurationProperties Bean：免代理自动 rebind**——配置变更触发 `EnvironmentChangeEvent`，`ConfigurationPropertiesRebinder` 对所有 @ConfigurationProperties Bean **原地重新绑定**字段（不重建实例、引用不变）——官方推荐的姿势；
2. **@Value 字段散落在普通 Bean：需要 @RefreshScope**——它把 Bean 换成 **CGLIB 代理**：真身缓存在 GenericScope 的 ConcurrentHashMap（refresh 作用域）；配置刷新事件 → 清缓存销毁真身 → **下次调用代理时按新 Environment 重建**（懒重建：没人访问就不重建，有状态字段会重置，这也是坑，见下）。

**完整事件链**（Nacos 版）：控制台发布 → Nacos Server 推送变更 → 客户端 `NacosConfigListener` 收到 → 更新 Spring `Environment`（PropertySource 替换）→ 发 `RefreshEvent` → `ContextRefresher.refresh()`：重算 changed keys → 依次发 EnvironmentChangeEvent / RefreshScopeRefreshedEvent → Rebinder rebind + RefreshScope 清缓存。**Bus 版**：`POST /actuator/busrefresh` → 经 RabbitMQ/Kafka 广播到每个实例，各实例跑同一套 refresh——**MQ 就是刷新事件的可靠性载体**（37_rabbitmq.md 的广播语义在此兑现）。

**三个实践坑**：① @RefreshScope Bean 里的**后台线程/缓存初始化**在重建时不会重做或重复做——监听器注册类字段重建后丢监听；有昂贵初始化状态的 Bean 别放 refresh 作用域，用 @PostConstruct 重建幂等设计；② `@Scheduled` 的 cron 想动态改，光 @RefreshScope 不够（任务不重排）——需重读注册；③ `management.endpoints` 暴露要收紧（busrefresh/refresh 端点被外扫=全员配置风暴）。

### 4.1.3 配置中心如何让客户端"实时"知道配置变了？推拉机制怎么设计？

**三种通道对照**（这是所有"服务端变更如何触达客户端"问题的母题——注册列表、开关、灰度配置同此理）：

| 通道 | 机制 | 时效/成本 | 谁在用 |
|------|------|----------|--------|
| 定时轮询 Pull | 客户端每 N 秒拉全量比对（MD5） | 最简、时效=N 秒 | Config 客户端、Eureka 客户端拉注册表 |
| **长轮询 Long Polling** | 客户端发起带超时的请求（典型 30s），服务端**有变更立刻返回、无变更挂起到期返回空**；客户端收到立即发起下一轮 | 秒级近实时 + 无状态服务端——**性价比之王** | **Apollo**（notification 按 namespace 挂起 60s）、Nacos 1.x |
| 长连接推送 | 双向流（gRPC/WebSocket）服务端主动推 | 毫秒级、需连接治理（心跳、会话粘性、百万连接规模） | **Nacos 2.x**（注册+配置统一走 gRPC 通道）、网关注册表推送 |

**长轮询细节**（追问必答的"它凭什么"）：服务端把请求挂进队列（如 Nacos 1.x 的 `LongPollingRunQueue`，29.5s < 30s 主动返回防网关切断）；变更时遍历挂起队列比对 **MD5/checksum**，不一致才返回"变了"——**返回的只是版本号，客户端再发一次增量拉配置**（"通知与传输分离"，与 14_canal.md 位点思想同源）。与 HTTP 长轮询/WebSocket 的取舍见 36_websocket.md；"推拉结合 + 全量兜底"是通用范式（17_nacos.md 对账机制详谈）。

**服务端视角的工程账**：长轮询吃线程/连接（同步容器下 Tomcat 挂请求占用——5_tomcat.md 1.2.3 容量题的具象场景），高规模实例数的配置推送要算账——这正是 Nacos 2 切 gRPC（一条多路复用连接挂全订阅）的动因。

## 4.2 可观测性

### 4.2.1 分布式链路追踪的原理？Spring Cloud Sleuth 为什么换成 Micrometer Tracing？

**核心模型 = 跨进程上下文传播**：一次请求分配 `traceId`；每个工作单元一段 `spanId`（父子串联成树，记录起止与调用关系）；**跨服务靠 HTTP 头（W3C traceparent 标准）/ RPC attachment/MQ 消息属性携带上下文**——日志与上报数据都盖同一 traceId，"一个请求一条树"。Sleuth 时代：Spring 各切面（RestTemplate/Feign/Gateway/MQ）埋点 + 采样上报 Zipkin。

**换代的三层原因（这题的 2026 正解）**：
1. **Sleuth 停更**：Spring 选择把"观测"标准化到 Micrometer 体系——Boot 3/Framework 6 起以 **Observation API**（一次埋点同时产出指标 + 链路）+ **Micrometer Tracing**（桥接 Brave 或 OTel 后端，再出 Zipkin/Jaeger/OTLP 导出）替代；
2. **OpenTelemetry 成为事实标准**：W3C traceparent 取代 B3 头、OTLP 协议统一传输——Sleuth 的私有语义与 OTel 冲突，桥接层归位到 Micrometer；
3. **零侵入 Java Agent 路线成熟**：OTel Agent 字节码增强覆盖全栈框架（36 种+），应用侧连埋点都不写——Spring 埋点退居"纯手动补盲区"。

**落地四要点**：① **采样策略**（头部采样控量/尾部采样保异常全链路，百万级 QPS 全量上报=自杀）；② **异步/线程池传播**——traceId 与 SecurityContext 同病：ThreadLocal 不跨线程（1_java.md 3.4；装饰执行器/ContextSnapshot 方案）；③ **业务 ID 关联**（订单号打进 span attribute，"按业务查链路"才是排障刚需，13_rocketmq.md 4.3.2 轨迹同理）；④ 日志 pattern 带 traceId（`%X{traceId}` MDC）——链路系统没存的那部分靠日志按 id 捞。

**易错**："Sleuth 没了=链路追踪没了"——能力还在（starter 换成 `spring-boot-starter-actuator` + micrometer-tracing-bridge-*），配置键变了；Zipkin（存储查询 UI）与 Jaeger/Tempo/Lightstep 是后端选型差异（24_elastic_search.md 自建大流量存储思路亦可套用）。

### 4.2.2 微服务可观测的"三支柱"是什么？SkyWalking 在生态里什么位置？

**三支柱各答各的问题**：

| 支柱 | 工具 | 回答 | 微服务痛点适配 |
|------|------|------|---------------|
| **Metrics 指标** | Micrometer + Prometheus/Grafana | "系统健康吗？趋势如何" | QPS/RT/错误率 RED + 资源 USE；告警从这出（4.1.3 推送之外的全局面） |
| **Tracing 链路** | Micrometer Tracing/OTel + Zipkin/Tempo | "这一笔慢在哪一跳" | 跨服务瀑布图定位（4.2.1） |
| **Logging 日志** | 结构化输出 + ELK/Loki | "到底发生了什么" | 集中化 + traceId 关联（5.2.1、24_elastic_search.md） |

**协同姿势**：仪表盘异常 → 下钻 trace 采样 → 按 traceId 拉日志——"指标报警、链路定位、日志定性"是完整闭环，缺一环排障都断腿。**第四支柱论**：profiling/事件流（pprof、连续剖析）正在补第四角——答到即前沿感。

**SkyWalking 位置**：Apache 开源 **一体化 APM**——探针（Java agent 字节码增强、多语言）+ 分析后端 + 存储（ES/H2/BanyanDB）+ UI 开箱全送：服务拓扑、实例/端点级指标、慢链路、告警；对 Spring 栈可作 OTLP/自有协议接收端，**和"自建 Micrometer+Prometheus+Zipkin+ELK 组合"是同一目标的两种买法**。取舍账：一体化省事、开箱拓扑与火焰图丰富，但探针强绑定自家协议、与 Micrometer 指标体系并行两套口径要治理；组合派灵活、标准化好，运维件数多。中小团队/国内落地：**SkyWalking 全家桶是务实解**；平台化大厂多为 OTel 统一采集 + 自建存储栈。追问"APM 和 Micrometer Tracing 重复吗"——采集语义可以合流（OTel 标准），UI/存储/告警能力互补，选型时二选一的通常是**接收端与存储**（17_nacos.md 生态同理：观测件与治理件各归各位）。

# 5. 架构演进与落地

## 5.1 落地与演进

### 5.1.1 从单体到微服务，如何设计落地路线？

**先纠偏**："微服务改造"不是重构任务的名称，是**组织与基础设施的迁移工程**——技术拆解只占三分之一，另两个三分之一是数据库拆分与平台建设。顺序错了就会得到"分布式单体"（拆了服务、没解耦发布与数据）。

**标准五步（演进式，绞杀者模式为纲）**：
1. **基建先行**：CI/CD 流水线（33_cicd.md）、集中日志、指标告警、注册配置——**没有观测就拆分 = 闭眼开车**（1.1.1 的账单）；
2. **划边界不动代码**：按 DDD 上下文盘资产（1.2.1），先在**单体内模块化**（Modulith/ArchUnit 守住依赖方向）——模块都守不住的服务拆出去只会更乱；
3. **先切边缘、后切核心**：选"变更频繁 + 无状态 + 数据耦合轻"的切口（通知、报表、BFF 聚合层），跑通"注册-网关-链路-发布"全流程；**核心交易域最后动**，且以" strangler 绞杀"：新服务接管部分路由，旧单体逐步退场，兼容层（防腐层 ACL）隔离新旧模型互译；
4. **数据拆分是最难一跳**：先"共库分 schema + 服务间禁直连"，再"读走 API/事件缓存（1.2.3、14_canal.md）"，最后物理分库——**代码先行、数据断后**，过渡期双写+对账是常态（13_rocketmq.md 2.3.2 本地消息表的用武之地）；
5. **组织对齐**：服务与团队所有权表（2.1.3 康威定律兑现）、值班与容量基线，发布门禁（指标不达标自动回滚）固化。

**高频事故清单（主动报出显实战）**：拆完调用链 8 跳以上（粒度错）；共享数据库没拆成（假微服务）；为拆而拆的"透传服务"；没有灰度就大爆炸切换。**收口句**："微服务落地顺序 = 观测先行 → 边界纸面化 → 边缘试点 → 模块内建 → 数据最后"，每一步都可停在当前形态而不亏——这才是演进式架构的本钱。

### 5.1.2 什么是 Service Mesh？和 Spring Cloud 有什么区别？

**定义**：把服务间通信的治理逻辑（负载均衡、重试超时、熔断限流、mTLS、路由/灰度、遥测）**从应用代码里搬进基础设施**——每个服务实例旁挂一个 **Sidecar 代理（Envoy）**接管进出流量，控制面（Istio 的 istiod 等）下发策略；**sidecar 间通信语言 = xDS**，应用只需要说"我要调 http://svc"。**Ambient 模式**（Istio 1.24+ 主推）去掉边车，节点层 L4 + 独立 L7 代理，运维成本减半——2026 谈 mesh 必带这句时效。

**与 Spring Cloud 对照**：

| 维度 | Spring Cloud（SDK 治理） | Service Mesh（平台治理） |
|------|--------------------------|--------------------------|
| 实现位置 | 应用进程内（jar） | 进程外（sidecar/节点代理） |
| 语言 | Java 专属 | **语言无关**（Go/Python/Node 同待遇） |
| 升级 | 改依赖、发版全量滚动 | 平台滚动，应用无感 |
| 性能 | 进程内调用零开销 | **每跳两次本地代理**（ms 级内延迟 + CPU 账） |
| 语义深度 | 懂业务框架上下文（Feign 接口、事务、熔断规则联动注册元数据） | 只懂 L4-L7 流量，不懂方法级语义 |
| 运维 | 低（就是 jar） | 高（控制面 + 边车注入 + 网络排查变复杂） |

**判断口径（两者不是替代关系，答题要分三层）**：① 纯 Java、团队吃透 Spring 生态、K8s 用得不深 → Spring Cloud 足够，mesh 是负资产；② **多语言异构 + 平台团队强势 + 治理策略要统一热下发** → mesh 接管南北向/东西向流量治理，应用 SDK 瘦身（Spring Cloud 只留 OpenFeign 门面 + Nacos 配置 + Micrometer 指标）；③ 现实终态是"**分层混用**"：入口治理（网关/mesh）归平台、业务级容错/配置归 SDK——Istio + Spring Cloud 共存是 2026 大厂常态（Dubbo 3 应用级发现与 mesh 友好也是同一股潮流，2.1.3）。**成本红线**：K8s 上 sidecar 让每 Pod 多一进程、控制面本身要 HA——"治理需求没到那个量级就别上 mesh"是标准劝退句。

## 5.2 趋势与判断

### 5.2.1 2026 年，微服务还需要 Spring Cloud 吗？趋势是什么？

**先给事实**：Spring Cloud 仍在正常发版（2025.1 列车已对齐 Boot 4，2.1.2），官方定位没变——"分布式模式的标准件货架"。但**它的货架正在被云原生平台部分收编**，面试这题考的是你有没有动态判断。

**被收编的三件**：
1. **注册发现** → K8s Service/EndpointSlice + DNS + 就绪探针（32_kubernetes.md）：Spring Cloud Kubernetes 直接实现 DiscoveryClient 接口无缝替换；Java 侧自管注册中心的净收益只剩"富元数据路由"与跨云部署——**容器内默认平台寻址**已是大厂路线；
2. **网关** → **Gateway API**（K8s 标准资源，SCG 之后 Ingress 的继任）+ Envoy Gateway/Istio ingress——七层入口策略归平台，SCG 退守"业务聚合型 BFF 网关"（鉴权聚合、协议适配这类懂业务的活）；
3. **追踪** → OpenTelemetry 统一采集（4.2.1），Sleuth 之死只是序幕——指标侧 Micrometer 也因 OTLP 出口而"埋点归标准、视图各选"。

**还活着且更聚焦的四件**：配置管理（ConfigData/Nacos 的配置语义 K8s ConfigMap 给不了灰度与监听体验，4.1.3）、声明式调用（OpenFeign/HTTP Exchange + Circuit Breaker——**业务级容错永远属于应用**）、事件驱动（Spring Cloud Stream 的 binder 抽象仍是 Java 消费 Kafka/RocketMQ 的顺手皮）、Seata/Nacos/Sentinel 组成的**中文生态一体化**（SCA 生命力所在，2.1.3）。

**2026 选型姿态（这题的满分结论）**："Spring Cloud 从'微服务全家桶'退位为 **'Java 应用的分布式补充件'**——平台能给的（寻址、入口、mTLS、基础遥测）交给 K8s/mesh/OTel；平台给不了的（配置治理、业务容错、事件语义、业务网关）继续 Spring Cloud。"再补一句反向保险：非 K8s/虚机部署或中小团队，Spring Cloud 全家桶仍是**最省事的默认**——趋势判断 + 场景底线，两头都有才叫会用。