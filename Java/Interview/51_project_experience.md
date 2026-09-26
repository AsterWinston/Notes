# 1. 知识点
- **技术选型理由**: 面试官关注为什么选这个技术而非其他(如Spring Cloud而非Dubbo的原因为生态完整、社区活跃、与Spring全家桶集成好)
- **技术选型回答框架**: 1)问题场景 2)候选方案对比(A/B/C) 3)选择依据(业务需求、团队能力、维护成本) 4)落地效果
- **常见选型对比**: 注册中心(Nacos/Eureka/Consul)、RPC框架(Dubbo/gRPC/Feign)、网关(Spring Cloud Gateway/Zuul/Kong/APISIX)、消息队列(Kafka/RocketMQ/RabbitMQ)
- **项目架构图**: 面试时需要能画出项目整体架构，包括微服务划分、网关、注册中心、配置中心、APM链路追踪、SQL数据库、MQ消息队列、容器平台、CI/CD流水线
- **QPS/TPS指标**: 需要清楚项目的量化指标：日均请求量、接口QPS峰值、数据库TPS、用户量级(DAU/MAU)、消息队列积压量
- **QPS峰值**: 一般指最忙接口的每秒请求数峰值，如订单接口峰值5000 QPS。回答时说明常规和峰值
- **TPS**: 每秒事务数，通常用于衡量数据库或支付系统的处理能力，如支付接口TPS 1000
- **日活与并发**: DAU与QPS的换算：QPS ≈ DAU × 人均请求数 / 86400 × 集中度系数(如80%请求集中在20%时间)
- **线上难题分类**: CPU飙高(死循环/频繁GC/正则回溯)、OOM(JVM内存泄漏/大对象)、慢SQL(缺失索引/大表Join)、MQ积压、Redis大Key/热点Key、DB死锁、分布式事务超时
- **CPU飙高排查**: top找到高CPU进程 → `top -H -p <pid>`找高CPU线程 → `printf '%x\n' <tid>`转十六进制 → `jstack <pid> | grep <hex_tid>`定位代码。或使用Arthas的`thread -n 3`直接看最忙线程
- **Full GC频繁排查**: `jstat -gcutil <pid> 1000`观察GC情况 → `jmap -dump:live,format=b,file=heap.dump <pid>`导出dump → MAT/JProfiler分析大对象、泄漏
- **慢SQL治理**: 开启慢查询日志(`slow_query_log`)、`EXPLAIN`分析执行计划(关注type、key、rows、Extra)、优化索引(覆盖索引/最左前缀)、SQL改写(避免SELECT */分页优化/大表Join拆分)
- **死锁排查**: `SHOW ENGINE INNODB STATUS`查看LATEST DETECTED DEADLOCK、分析最近死锁详情、`information_schema.INNODB_TRX`查看当前事务、调整加锁顺序或改乐观锁
- **MQ消息积压**: 确认消费者是否正常、临时增加消费者实例、批量消费(一次处理多条)、跳过非关键消息、优化消费逻辑(SQL/业务)
- **事故复盘(RCA)**: Root Cause Analysis根因分析，五问法(5 Whys)深挖根因，记录时间线、影响范围、处理过程、改进措施、责任人
- **RCA模板**: 1)事故概述(时间/影响范围/持续时长) 2)时间线(发现→响应→定位→修复→恢复) 3)根因分析(直接原因/间接原因/根本原因) 4)改进措施(短期/长期) 5)监控告警完善
- **性能优化流程**: 发现问题(监控告警/压测) → 定位瓶颈(Arthas/JProfiler/jstack/jmap) → 分析原因 → 制定方案 → 实施 → 验证对比前后指标
- **N+1查询问题**: ORM懒加载导致循环查询。如查询用户列表后循环查询每个用户的订单。解决：批量查询(IN条件)、JOIN查询、使用`@BatchSize`/`@Fetch(FetchMode.SUBSELECT)`
- **数据库连接池优化**: 监控连接数(HikariCP的HikariPoolMBean)、调整maxPoolSize、检查连接泄漏(长时间未释放)、设置合理的connectionTimeout
- **系统可观测性三支柱**: Logging(日志-ELK)、Metrics(指标-Prometheus+Grafana)、Tracing(链路追踪-SkyWalking/Jaeger)
- **日志规范**: 统一格式(JSON)、包含traceId(链路追踪)、区分级别(DEBUG/INFO/WARN/ERROR)、敏感信息脱敏、避免大日志(大对象截断)
- **Prometheus+Grafana**: Prometheus采集指标(通过Micrometer暴露actuator端点)、Grafana可视化面板、配置告警规则(AlertManager)
- **SkyWalking**: APM链路追踪，自动采集服务间调用拓扑、接口耗时、SQL耗时、错误率。Agent无侵入(字节码增强)
- **Git Flow规范**: 分支策略(Master/Main、Develop、Feature、Release、Hotfix)。Feature分支从Develop拉取，合并回Develop；Release从Develop拉取发布到Master
- **CI/CD流程**: 代码Push → Jenkins/GitLab CI触发 → 编译构建 → 单元测试+代码扫描 → 构建Docker镜像 → 推送到镜像仓库 → K8s滚动更新
- **Code Review规范**: 关注点：逻辑正确性、异常处理、性能、安全、代码风格、是否可测试。使用GitLab MR/GitHub PR做code review
- **上线Checklist**: 代码review通过、测试报告、DB变更脚本、配置变更、灰度方案、回滚方案、监控告警配置、值班安排
- **DDD领域驱动设计**: 战略设计(限界上下文、子域划分)和战术设计(聚合根、实体、值对象、领域事件)。核心思想：业务领域驱动技术架构
- **微服务拆分原则**: 按业务边界拆分(DDD限界上下文)、高内聚低耦合、数据库独立(每服务独立DB)、拆分粒度适中(太细增加复杂度)
- **数据中台**: 统一数据服务层，整合各业务线数据，提供标准化数据API。核心：数据治理、数据质量、数据安全、数据服务化
- **QPS vs TPS**: QPS(Queries Per Second) — 每秒查询数，衡量接口/网关层面的请求处理能力; TPS(Transactions Per Second) — 每秒事务数，衡量业务事务处理能力(如支付/下单)，通常TPS ≈ QPS×业务复杂度系数。QPS用于API网关，TPS用于数据库/支付系统
- **接口调优策略**: (1) SQL优化: 加索引/优化查询/分页/缓存; (2) 代码优化: 合并查询/懒加载/异步处理/减少不必要对象创建; (3) 中间件: Redis缓存/MQ异步削峰/Nginx负载均衡; (4) 横向扩展: K8s增加副本/Docker弹性扩缩

# 2. 面试题

## 2.1 你所在项目的技术架构是怎样的？请画图描述
标准回答框架：
1) **前端层**：Vue/React + Nginx + CDN，管理后台和H5/小程序
2) **接入层**：Nginx(反向代理)+Spring Cloud Gateway(API网关)，统一鉴权和限流
3) **微服务层**：按业务拆分为用户服务、订单服务、商品服务、支付服务等，通过Feign/Ribbon通信
4) **中间件层**：Nacos(注册中心+配置中心)、Sentinel(限流熔断)、SkyWalking(链路追踪)
5) **数据层**：MySQL(主从+读写分离)、Redis(缓存+分布式锁)、Elasticsearch(搜索)、RocketMQ(削峰解耦)
6) **运维层**：Kubernetes容器编排、Prometheus+Grafana监控、ELK日志收集、Jenkins CI/CD

## 2.2 线上CPU飙升到100%怎么排查？
1) `top` 查看进程CPU使用率，找到高CPU的Java进程PID
2) `top -H -p <pid>` 查看进程内线程CPU使用率，找到高CPU线程TID
3) `printf '%x\n' <tid>` 将线程ID转为十六进制
4) `jstack <pid> > jstack.log` 导出线程堆栈
5) `grep <hex_tid> jstack.log -A 30` 查看线程调用栈，定位具体代码
6) **Arthas简化方法**：`thread -n 3` 直接查看CPU占用最高的3个线程及堆栈
常见原因：死循环、大量正则匹配(回溯)、频繁Full GC、JSON序列化大对象

## 2.3 Full GC频繁怎么排查和优化？
**排查步骤**：
1) `jstat -gcutil <pid> 1000 20` 观察各区域使用率和GC频率/耗时
2) `jmap -heap <pid>` 查看堆内存配置和占用
3) `jmap -dump:live,format=b,file=heap.hprof <pid>` 导出堆dump
4) 使用MAT(Eclipse Memory Analyzer)分析dump：查看Dominator Tree(最大对象)、Leak Suspects(泄漏嫌疑)、Histogram(类实例统计)
5) Arthas的`dashboard`查看实时JVM状态，`heapdump`导出dump

**常见原因与优化**：
- 堆内存太小：增大-Xmx和-Xms
- 老年代设置太小：调整-XX:NewRatio
- 大对象直接进老年代：控制对象大小，调整PretenureSizeThreshold
- 内存泄漏：查看dump中类实例数量异常增长(如HashMap/List持续增长)
- GC算法选择：高吞吐选G1GC(-XX:+UseG1GC)，低延迟选ZGC

## 2.4 项目中进行过哪些性能优化？请举例
**案例**：订单列表接口响应从2.5s优化到200ms
1) **发现问题**：Grafana监控发现该接口P99耗时长达2.5s，调用量增大时DB CPU飙高
2) **定位**：SkyWalking追踪发现1次列表查询触发了N+1次子查询(循环查用户信息、订单明细)；Arthas的`trace`发现SQL扫描行数很大
3) **优化措施**：
   - SQL优化：将N+1查询改为1次JOIN查询，减少DB交互次数
   - 索引优化：对order表的查询条件列(user_id, status, create_time)建立联合索引
   - 缓存优化：对不常变的用户信息使用Caffeine本地缓存，减少用户服务调用
   - 分页优化：使用游标分页(基于主键)代替OFFSET分页
4) **结果**：接口响应降到200ms，DB CPU降低50%

## 2.5 MQ消息积压几十万条怎么处理？
**紧急处理**：
1) 检查消费者是否存活：确认消费者没有挂掉或被限流
2) 临时扩容：增加消费实例数量，提升消费能力
3) 批量消费：将单条消费改为批量消费(一次拉取10-100条)，减少网络开销
4) 消费逻辑优化：定位消费慢的环节(如DB操作)，加索引、批处理优化
5) 跳过非关键消息：如日志型消息可暂时丢弃或转存到文件
6) 消息迁移：将积压消息从原Topic迁移到新Topic，由新消费者组处理

**预防措施**：消费者线程池合理配置、消费超时设置、积压告警(超过阈值立即通知)、扩容预案

## 2.6 如何进行事故复盘(RCA)？
**复盘流程**：
1) **事故概述**：XX年XX月XX日，服务A在11:00-11:30不可用，影响用户无法下单，PV影响约10万用户
2) **时间线**：11:00 监控告警→11:02 值班响应→11:05 定位到慢SQL→11:10 执行索引变更→11:20 服务恢复→11:30 验证确认
3) **根因分析**：直接原因：user_id字段缺失索引导致全表扫描。根本原因：上线前DB变更脚本遗漏索引创建，code review未覆盖
4) **改进措施**：短期-补建索引，添加监控。长期-上线流程增加DB变更检查项、SQL审核工具集成、review checklist增加索引检查
5) **5 Whys**：为什么服务不可用？DB CPU 100%。为什么CPU 100%？慢SQL全表扫描。为什么慢SQL？索引缺失。为什么索引缺失？DDL脚本没有建。为什么没有检查出？code review流程没有覆盖DDL

## 2.7 你做过哪些技术选型？选型依据是什么？
**示例**：注册中心选型
- **场景**：微服务25+个，服务实例300+，需要服务注册发现和配置管理
- **候选方案**：Eureka(AP)、Zookeeper(CP)、Nacos(AP/CP+配置中心)、Consul(CP+健康检查)
- **选型依据**：服务发现对可用性要求高于一致性(选AP)；配置管理需要强一致性(需要CP)；团队有Spring Cloud Alibaba背景；需要配置中心功能
- **选择Nacos**：满足服务发现(AP暂存实例)和配置管理(CP持久配置)双需求；与Spring Cloud Alibaba无缝集成；中文社区活跃，文档齐全
- **落地效果**：支撑500+服务实例注册，配置热更新正常，运行稳定

## 2.8 请分享一个你遇到的印象最深的Bug/问题
**案例**：下单接口偶发性超时
- **现象**：高峰期(中午12点)，下单接口偶发超时(占比0.5%)，且超时后订单已创建(数据库有记录)，导致用户重复下单
- **排查过程**：
  1) 查SkyWalking：超时请求在支付回调环节耗时异常(>3s)
  2) 查应用日志：支付回调中获取分布式锁等待超时
  3) 定位代码：下单和支付回调用了同一把分布式锁(订单ID为key)，但下单流程持锁时间过长(包含发短信、写日志等非核心操作)
- **根因**：锁粒度过粗，非核心操作在锁内执行导致锁持有时间长，支付回调获取锁超时
- **解决**：缩小锁的范围，只保护库存扣减和订单状态变更；发短信、通知等操作移出锁外异步执行
- **教训**：分布式锁内只做核心的互斥操作；设置合理的锁超时时间；锁的加解锁必须成对出现

## 2.9 QPS和TPS的区别是什么？
**QPS（Queries Per Second）**：每秒查询数，衡量系统处理请求的能力。关注接口/网关层面，一次请求就是一次查询。适用于API网关、Web服务器、搜索引擎等场景。

**TPS（Transactions Per Second）**：每秒事务数，衡量系统处理业务事务的能力。一个事务可能包含多个请求（如下单包含库存查询、扣减、订单创建、支付等多个步骤）。适用于数据库、支付系统等场景。

| 维度 | QPS | TPS |
|------|-----|-----|
| 全称 | Queries Per Second | Transactions Per Second |
| 粒度 | 单次请求/查询 | 完整业务事务 |
| 关系 | TPS ≈ QPS × 业务复杂度系数 | 一个事务通常包含多个QPS |
| 适用 | API网关、接口压测 | 数据库、支付系统、订单系统 |
| 示例 | 查询用户信息接口 5000 QPS | 下单接口 1000 TPS（内部含3-5次查询） |

回答面试时需说明：我们项目订单接口峰值约X QPS，支付接口约Y TPS。

## 2.10 如何进行接口调优？
接口调优需要多维度、分层梳理：

**1. SQL层面（影响最大）**：
- 添加缺失索引（通过EXPLAIN分析，关注type/rows/Extra）
- 优化慢查询（避免SELECT *、大表Join拆分、深分页改游标分页）
- 读写分离（主库写、从库读）
- 合理使用缓存（Redis缓存热点数据，减轻DB压力）

**2. 代码层面**：
- 合并查询：将N+1次查询改为1次批量查询或JOIN
- 懒加载：非必要数据按需加载
- 异步处理：耗时操作（发短信/写日志）异步化
- 减少对象创建：避免循环中创建大对象、使用对象池

**3. 中间件层面**：
- Redis缓存：热点数据缓存、计算结果缓存
- MQ异步削峰：高并发写场景先入MQ，异步消费处理
- Nginx负载均衡：横向扩展服务节点

**4. 横向扩展**：
- K8s增加服务副本数，通过HPA自动扩缩容
- Docker容器化，弹性伸缩

**回答思路**：先描述遇到的性能问题 → 通过SkyWalking/Arthas定位瓶颈 → 针对性优化 → 对比优化前后指标。
