Apache ShardingSphere 不是一个单独的 jar 包，而是**一套分布式数据库生态系统的集合**。它解决的问题很朴素：单库单表撑不住了，但又不至于要换一套 NewSQL 数据库。ShardingSphere 的思路是**在现有关系型数据库之上加一层 SQL 引擎**，把分片、读写分离、加密、影子压测、弹性迁移这些"分布式化"的脏活从业务代码里剥离出去，让应用继续把 MySQL、PostgreSQL 当成一个"逻辑上更大的数据库"来用。本篇先把定位、演进史、产品形态和功能边界讲清楚，后面的架构与用法都建立在这张地图上。

# 1. 定位与演进

## 1.1 从 Sharding-JDBC 到 Apache 顶级项目

**ShardingSphere 的血脉来自 Sharding-JDBC**。2016 年当当网开源了内部分库分表组件 Sharding-JDBC（作者张照轩），特点是纯 Java 客户端、无中心化、直接嵌入应用；2018 年该项目与其后的 Proxy 形态一起以 ShardingSphere 之名进入 **Apache 软件基金会孵化器**，2020 年 4 月正式毕业成为**顶级项目（TLP）**，是国内数据库中间件领域第一个走完孵化流程的 Apache 项目。这段历史有两个实际影响：

- **Java 内核优先**。因为前身是 JDBC 库，所以解析、路由、改写这套内核最早是为 Java 客户端设计的，Proxy 只是把同一套内核包了一层数据库协议。今天依然是 JDBC 功能领先、Proxy 补齐协议能力。
- **SphereEx 主导社区**。核心贡献者成立了商业化公司 SphereEx，功能节奏（如 DistSQL、数据迁移 Pipeline、Proxy CDC）基本由这条线驱动，社区版本与企业版 ShardingSphere-Insight/云版的边界需要在选型时留意。

## 1.2 与 Mycat、Sharding-JDBC 前身、Vitess 的关系

选型讨论里最容易混淆的是这三个名字：

- **Mycat**：独立于 ShardingSphere 的老牌 Proxy 中间件（1.x 基于 Cobar 代码），**只有 Proxy 形态**，配置以本地 XML/JSON 为准，集群一致性靠人工同步。Mycat2 是重写版，走元数据一致性路线，但社区规模和 SQL 解析深度不及 ShardingSphere。
- **Sharding-JDBC**：ShardingSphere 的 JDBC 形态前身，4.x 之前甚至可以单独用这个 artifact。5.x 之后统一收敛为 `shardingsphere-jdbc`，不再作为独立项目存在。
- **Vitess**：YouTube 为 MySQL 水平扩展写的 Proxy 体系，CNCF 毕业项目，**为云原生而生**，自带拓扑管理、查询并行化与 vtgate/vtworker 工具链，但对 SQL 方言只有 MySQL，且运维模型与 Java 技术栈距离较远。

三者共同的上位概念是"**在 MySQL 之上做分布式**"，差别在接入形态、SQL 处理深度和生态归属。

| 维度 | ShardingSphere 5.5.x | Mycat2 | Vitess |
|------|---------------------|--------|--------|
| **接入形态** | JDBC + Proxy + Sidecar（规划中） | Proxy | Proxy（vtgate） |
| **SQL 方言** | MySQL/PostgreSQL/openGauss/SQL Server/Oracle/ClickHouse 等 | MySQL | MySQL |
| **配置管理** | DistSQL 在线变更 + YAML + 元数据仓库 | JSON 文件 | topology + vtctl |
| **分布式事务** | LOCAL / XA / BASE（Seata 集成） | XA | 2PC / VCursor 事务 |
| **弹性伸缩** | Migration Pipeline 在线迁移 | 手工 | 原生 resharding |
| **生态归属** | Apache TLP | 独立社区 | CNCF |
| **Java 亲和度** | 最高（同进程） | 中 | 低 |

> **行业实践**：国内 Java 技术栈的选型分布大致是——**已有 Spring 应用、追求零额外部署成本走 JDBC**；**多语言或 DBA 团队要求集中管控走 Proxy**；Mycat 存量系统通常维持原样，新模块才引 ShardingSphere；Vitess 主要出现在云原生 + 全 MySQL 且团队有较强运维能力的场景。

## 1.3 版本演进主线

| 版本 | 关键变化 |
|------|---------|
| 1.x–3.x | Sharding-JDBC 独立发展，Spring namespace 配置，功能只有分片与读写分离 |
| 4.x | 引入 Proxy，治理中心（Governance）雏形，YAML 配置以 `dataSources` + 规则前缀式写法为主 |
| 5.0（2021） | **配置模型重构**：统一为顶层 `dataSources` + `rules` 列表（`!SHARDING`、`!READWRITE_SPLITTING` 等），引入 Mode（Standalone/Cluster）替代旧治理中心，DistSQL 上线 |
| 5.1–5.3 | 数据迁移 Pipeline 取代 ElasticJob 成为弹性伸缩主力，Proxy 补齐 PostgreSQL/openGauss 协议 |
| 5.4–5.5.x | SQL 联邦查询（Calcite）转正、数据脱敏（Mask）独立成规则、ShardingSphere-Agent 可观测体系成熟、Oracle/SQL Server 方言解析增强 |

> **注意**：4.x 与 5.x 的配置**完全不兼容**，网上大量教程仍写 4.x 的 `sharding.jdbc.datasource.names=ds0,ds1` 与 `actualDataNodes` 挂在 `tables.t_order` 下的旧结构，直接照抄到 5.5.x 会启动失败。5.x 的规则块以 `!` 开头的 YAML 标签（tagged style）表达，读教程时先看它有没有 `rules:` 顶层字段。

# 2. 三形态：JDBC / Proxy / Sidecar

ShardingSphere 5.x 官方定义的产品矩阵是"**同一内核，三种接入形态**"，其中前两种已经成熟可用，Sidecar 仍在推进。

```
                ┌────────────────────────────────────────────┐
                │            ShardingSphere 内核             │
                │  解析 → 路由 → 改写 → 执行 → 归并（五大引擎）│
                └───────┬───────────────┬──────────────┬─────┘
                        │               │              │
             ┌──────────▼──┐    ┌───────▼────────┐  ┌──▼──────────────┐
             │ JDBC（SDK） │    │ Proxy（独立进程）│  │ Sidecar（规划中）│
             │ 应用同进程   │    │ MySQL/PG 协议   │  │ K8s Pod 伴生    │
             └──────┬──────┘    └───────┬────────┘  └──────┬──────────┘
                    │ 直连              │ JDBC 连接池       │ 转发
             ┌──────▼───────────────────▼──────────────────▼────┐
             │           MySQL / PostgreSQL / openGauss ...     │
             └──────────────────────────────────────────────────┘
```

**ShardingSphere-JDBC** 定位是"增强版 JDBC 驱动"，实现了 `DataSource`/`Connection`/`Statement` 等标准接口，对上层 MyBatis、JPA、Spring JDBC 完全透明。它的核心卖点是**零额外网络跳数、零额外部署**，代价是仅限 Java、治理逻辑分散在每个应用进程里、升级要随应用发版。

**ShardingSphere-Proxy** 伪装成一个 MySQL/PostgreSQL Server，任何能用 mysql 客户端连上的语言都能接入；配置集中在服务端，DBA 可通过 DistSQL 直接运维。代价是多一跳网络、Proxy 本身成为关键路径需要高可用部署。

**ShardingSphere-Sidecar** 是 Kubernetes 语境下的形态：Proxy 以伴生容器部署在业务 Pod 内，走 localhost 通信，兼具 Proxy 的语言无关性与接近 JDBC 的时延。官方文档中它长期标注为 *To Be Released*，实践中一般用 **ShardingSphere-Operator** 把 Proxy 部署成独立 Service 或 Sidecar 容器来近似达成，做技术选型时不要把它当成已交付能力。

| 维度 | JDBC | Proxy | Sidecar（规划） |
|------|------|-------|----------------|
| 部署方式 | 依赖嵌入应用 | 独立集群 | Pod 内容器 |
| 语言支持 | 仅 Java | 协议级任意语言 | 协议级任意语言 |
| 网络开销 | 无额外跳 | +1 跳（跨节点） | +0 跳（localhost） |
| 性能 | 最高 | 略低（协议编解码） | 接近 Proxy 上限 |
| 配置管理 | 随应用发布 | DistSQL / YAML 集中 | 集中 |
| 运维复杂度 | 分散在各应用 | 需保 Proxy 高可用 | 依赖 K8s 编排 |
| 典型场景 | 存量 Java 单体/微服务 | 多语言、DBA 集中管控、数据迁移 | 云原生多语言 |

> **设计争议**：JDBC 还是 Proxy 长期有争论。JDBC 派认为多一跳代理是不可接受的稳定性风险（Proxy 挂了全站挂）；Proxy 派认为客户端配置的版本漂移无法治理。5.x 的答案是"**两者共存 + Mode 统一元数据**"：Cluster 模式下 JDBC 与 Proxy 共享注册中心里的同一份规则，配置漂移问题被 Mode 消化掉，争论点收敛为"能否接受多一跳"。

# 3. 功能矩阵

ShardingSphere 的官方能力清单按"数据分片 / 弹性伸缩 / 数据加密 / 数据脱敏 / 隐私计算（实验中） / 异构数据源联邦查询 / 分布式事务 / 链路追踪与指标 / DistSQL / 影子库压测 / 权限与连接治理"组织。下表给出使用视角的能力—场景对应：

| 能力 | 内核机制 | 适用场景 | 对应本篇后续 |
|------|---------|---------|-------------|
| **数据分片** | 分片算法 + 路由引擎 | 单表亿级、写入瓶颈 | 4_sharding |
| **读写分离** | 主从组 + 负载均衡算法 | 读多写少、报表分流 | 5_read_write_splitting |
| **数据加密** | 加解密改写 + 辅助查询列 | 手机号/身份证落库合规 | 6_data_encryption |
| **数据脱敏** | 返回结果改写 | 开发/客服查库看脱敏值 | 5.x 新增 Mask 规则 |
| **影子库** | 影子算法判定流量 | 生产全链路压测 | 7_shadow_database |
| **分布式事务** | LOCAL/XA/BASE 三类型 | 跨分片写一致性 | 8_distributed_transaction |
| **DistSQL** | RDL/RQL/RAL/RUL 四类语法 | 在线运维与变更 | 9_distsql |
| **弹性伸缩** | Migration Pipeline | 在线扩容迁移、异构迁库 | 10_governance |
| **可观测** | Agent + Prometheus/OTel | 指标、链路、慢 SQL | 11_observability |

各能力**可以叠加组合**——这是 ShardingSphere 与多数"只做分片"的中间件最大的差异点：一条 SQL 会先经分片路由，再经读写分离选主从，再经影子判定，最后按事务类型执行。组合关系不是任意自由度的，例如分片键不能选加密列、XA 事务对数据库有协议要求，具体限制各篇会展开。

# 4. 核心设计理念

理解下面五点，基本就能预判 ShardingSphere 各类行为的边界：

1. **SQL 贯穿始终**：一切能力都实现为"解析后的 SQL 上的操作"。Parse 出 AST 后，分片决定路由、加密决定改写、归并决定结果组装，这就是为什么它对"函数包裹分片键""跨库子查询"这类解析不出语义的 SQL 会降级甚至报错。
2. **插件化（SPI）**：分片算法、主键生成器、负载均衡、加密算法、影子判定、元数据仓库、事务管理器，全部是 SPI 接口。官方内置实现只是默认选项，业务可以替换而不碰内核。
3. **数据库增强而非替代**：它不自己存数据，不重造存储引擎，因此**继承了底层数据库的事务与复制语义**；同样地，跨库 JOIN 无法走数据库原生能力，性能天花板由归并引擎决定。
4. **同一内核多端复用**：JDBC、Proxy、未来 Sidecar 共享同一套 Kernel 与 Mode，功能与配置语义保持一致，只是接入协议不同——这避免了 Mycat 时代"客户端版和服务端版规则写法不同"的割裂。
5. **无中心化 + 中心化混合**：JDBC 形态下没有协调者（配置本地生效或从注册中心拉取），Proxy 形态下有集群元数据面。Cluster Mode 是可选而非强制的，中小规模完全可以先用 Standalone 起步。

> **设计考量**：
>
> - **不建议把 ShardingSphere 当成"分布式数据库"平替 NewSQL**：跨分片事务、全局二级索引、分布式聚合下推这些能力它的实现深度有限，数据倾斜和大查询代价高的场景优先考虑 TiDB/OceanBase。
> - **JDBC + Proxy 混用是常态而非异常**：Java 走 JDBC、脚本与第三方系统走 Proxy，同一个 Cluster Mode 保证两边看到同一份规则，这是官方推荐的生产拓扑。

# 5. 小结

| 记忆点 | 一句话 |
|--------|--------|
| 出身 | Sharding-JDBC → 2018 进 Apache 孵化器 → 2020 毕业 TLP，SphereEx 主导 |
| 形态 | JDBC（Java SDK）、Proxy（协议网关）成熟，Sidecar 规划中 |
| 内核 | 解析→路由→改写→执行→归并五阶段 SQL 引擎 |
| 能力 | 分片/读写分离/加密/脱敏/影子/事务/弹性迁移/DistSQL/可观测，可叠加组合 |
| 版本 | 4.x 与 5.x 配置格式不兼容，5.x 用 `rules:` + `!SHARDING` 标签写法 |
| 边界 | 增强既有数据库而非替代，重跨库计算的场景要考虑 NewSQL |

下一篇（2_architecture）拆内核：五大引擎的执行流水线、Facade/Engine 分层和 SPI 插件体系。
