# 1. 整体架构：两代产品的两种骨架

Mycat 家族里存在两套完全不同的进程架构：**1.6.x 是 Cobar 血统的手写 NIO Reactor 多线程模型，2.x 是 Netty + 自研 Aloha 框架的反应式管道**。面试与生产语境中说"Mycat 架构"默认指 1.6，本篇以 1.6 为主线、2.x 为对照。

## 1.1 Mycat 1.6：Reactor 多线程模型

Mycat 1.6 的进程骨架是一套**手写的多 Reactor 线程模型**：主 Reactor（NIOAcceptor）只干一件事——accept 新连接；连接建立后被轮询分配给某个子 Reactor（NIOReactor），此后该连接的读写事件都在这个子 Reactor 的 Selector 上处理；请求的解析、路由、改写、执行调度则交给与 Reactor 一一绑定的 **Processor** 线程池。**一条前端连接终身绑定一个 Processor**，这是理解 Mycat 1.6 一切性能与故障现象的钥匙：某个 Processor 被打满（比如一条慢 SQL 做了全分片归并），只会拖慢它名下的连接，而不是全进程卡死——线程模型天然实现了故障隔离。

```text
                        Client (JDBC / mysql 客户端)
                                  │ 8066
                 ┌────────────────▼────────────────┐
                 │        ServerBootstrap          │ ← 进程启动总装
                 │ (配置加载/线程池/心跳/管理线程)      │
                 └────────────────┬────────────────┘
                                  │
                    ┌─────────────▼─────────────┐
                    │        NIOAcceptor        │ 主 Reactor：只 accept
                    │   (NIOReactorPool 轮询分发)  │
                    └──────┬───────────┬─────────┘
              ┌────────────▼───┐  ┌────▼────────────┐
              │ NIOReactor-0   │  │ NIOReactor-1    │ … N 个（= processors）
              │ R读线程/W写线程  │  │ R读线程/W写线程   │   + BO 后台线程
              │ FrontendConn   │  │ FrontendConn    │ ← 连接↔Reactor 固定绑定
              └────────┬───────┘  └────────┬────────┘
              ┌────────▼───────┐  ┌────────▼────────┐
              │  Processor-0   │  │  Processor-1    │ ← SQL 解析/路由/改写/归并
              │ (后端连接池集合) │  │ (每 dataHost 各  │
              └────────┬───────┘  │  一套池)         │
          ┌────────────┼──────────┴─────────────────┘
          ▼            ▼            ▼
      MySQL dn1     MySQL dn2    MySQL dn3     ← 9066 管理端口同样各有一套 Acceptor
```

**线程角色速查**：

| 线程 | 数量 | 职责 | 关键调优参数 |
|------|------|------|-------------|
| Acceptor | 每端口 1 个 | accept 前端连接，轮询分给子 Reactor | `serverPort`/`managerPort` |
| Reactor R/W | 每 Reactor 各 1 个 | Selector 事件轮询、读写 SocketChannel | `processors` |
| BO 后台 | 每 Reactor 1 个 | 写就绪队列、定时任务 | — |
| Processor | 与 Reactor 一一对应 | SQL 解析、路由、改写、归并调度 | `processorBufferPool` |
| 心跳/定时器 | 全局少量 | dataHost 主从探测、序列加载 | `heartbeatTimeout` |

> **注意**：`processors` 参数常被误解为"业务线程数"。它实际是 Reactor/Processor 的对数，默认等于 CPU 核数，一般不建议超过核数 × 2。设得过大只会加剧上下文切换，且每个 Processor 独立持有 DirectByteBuffer 缓冲池，内存随线程数线性上涨——"16 核机器把 processors 调到 32 反而吞吐下降"是社区反复出现的案例。

## 1.2 ServerBootstrap / Processor / Session 三层分工

**ServerBootstrap 是总装车间**：入口 `Bootstrap.main()` 里它按顺序加载 `server.xml`（端口、线程、用户、集群）、`schema.xml`（逻辑库、dataHost、dataNode）、`rule.xml`（分片规则与函数），初始化 NIOReactorPool、ProcessorPool、心跳线程与管理线程，最后把 8066/9066 两个 NIOAcceptor 挂上去。它不处理任何一条 SQL。

**Processor 是干活的大脑**：每个 Processor 聚合了一组功能线程与全部后端连接资源——向后端发 SQL 收结果的执行器、`DataSourceManager`（每个 dataHost 一套物理连接池）、大结果集临时文件与归并服务。**后端连接池是按 Processor 隔离的**：同一个 dataHost 的池在每个 Processor 里各有一份。所以"MySQL 侧最大连接数"要按 `processors × 每池 maxCon × dataHost 数` 估算，这是 Mycat 上线容量规划最容易漏算的一笔账。

**Session 是一条连接的上下文**：`FrontendConnection`（前端会话，持有用户、schema、事务状态、字符集、当前执行中的 SQL）与 `BackendConnection`（后端物理连接，持有真实 MySQL 的 socket、事务状态、缓存的数据包）。一条前端 SQL 执行时，Mycat 从对应 dataNode 的池借出若干后端会话并行下发，等全部响应到齐后归并写回前端。三层关系可以背下来：**Bootstrap 装配一次，Processor 常驻池化，Session 随连接生灭**。

```java
// Mycat 1.6 请求处理链（简化示意）
void handleQuery(FrontendConnection conn, String sql) {
    // 1. 语法初判：RumpParser + Druid 语法树识别语句类型
    // 2. 路由：按 rule.xml 的分片函数算出目标 dataNode 集合
    RouteResultset route = conn.getSchema().route(sql, conn);
    // 3. 读写分离：写→writeHost；读→按 balance 策略选节点
    route.setRunOnSlave(needSlave(route));
    if (route.getNodes().length == 1) {
        send(route.getNodes()[0], sql);              // 4a. 单节点：原样直发
    } else {
        for (RouteResultsetNode node : route.getNodes()) {
            send(node, rewriteSql(sql, node));       // 4b. 多节点：改写后并行下发
        }
        ResultSet merged = mergeService.merge(route); // 5. 归并排序/聚合再计算
        writeBack(conn, merged);                     // 6. 编码成 MySQL 协议包回写
    }
}
```

与 1.6 的"过程式改写"相比，**Mycat 2.x 把第 2~5 步换成了 Calcite 标准流程**——解析成 SqlNode、校验成 RelNode、优化后生成分布式执行计划，聚合与下推的正确性明显更好，这是两代产品对复杂 SQL 支持差距的根源。

## 1.3 前端连接与后端连接

| 连接侧 | 生命周期与要点 |
|--------|---------------|
| Frontend（客户端→Mycat，8066） | server.xml `<user>` 按 schema 白名单授权；超限/认证失败/idleTimeout 直接断开；一个前端连接 ≠ 一个后端连接 |
| Backend（Mycat→MySQL，按 dataNode 分组） | 每 dataHost 的 writeHost/readHost 各有池；事务内按节点绑定一条连接复用；广播查询并行借 N 条、用完即还；空闲被 MySQL wait_timeout 踢掉后靠心跳探活恢复 |

**设计考量**：

- **前后端连接数不成正比**：应用侧 1000 条池化连接打到 Mycat，Mycat 到每个 MySQL 可能只保持几十条——这是 Proxy 方案"收敛后端连接"卖点的由来，也是它必须精确管理借还时机的原因
- **字符集在握手时锁定**：前端 `SET NAMES` 会同步下发给该会话当前持有的后端连接，中途新建的后端连接靠 dataHost 的 `initSQL` 补齐；两边不一致就会出现"同一个查询偶尔乱码"的玄学问题
- **后端连接故障对前端透明重建**：心跳发现 writeHost 挂掉后切换新连接，但正在进行的事务会失败，应用层必须能重试

# 2. Mycat 2.x：反应式管道架构

Mycat 2.x 采用**基于 Netty + Aloha 的反应式管道架构**，请求依次流经 Protocol 解码 → Router 路由 → Calcite 优化器 → Planner → Executor 分发后端 → Merger 结果聚合，最后编码回写客户端。**内核特性对比 1.6**：网络层交给 Netty 的 EventLoop，连接与 EventLoop 固定绑定，等价于 1.6 的"连接绑定 Processor"，但缓冲管理、半包粘包、流控全由 Netty 兜底；EventLoop 中不允许执行阻塞操作，后端 JDBC 类调用提交到独立执行线程，全链路 future 化；DDL 对逻辑表的操作借助 Calcite 可以精确改写并广播到所有分片。

## 2.1 协议层

Mycat（两代一致）对上层完整模拟 MySQL Server 协议：握手阶段返回伪造的 Server 版本信息（1.6 报 5.x，2.x 报 8.0.x），认证凭据来自 server.xml `<user>`（2.x 为 users.json），随后按命令类型分流处理：

| 命令 | 说明 | 处理方式 |
|------|------|---------|
| `COM_QUERY` | 普通查询 | Mycat 解析路由 |
| `COM_STMT_PREPARE` | 预编译 | Mycat 模拟 |
| `COM_STMT_EXECUTE` | 执行预编译 | Mycat 模拟 |
| `COM_PING` | 心跳 | Mycat 直接响应 |
| `COM_QUIT` | 断开 | 释放连接 |
| `COM_INIT_DB` | 切换数据库 | 切换 Schema |

> **注意**：Mycat 的"预编译"是中间件自己拼好参数再发往后端，并非把客户端 PreparedStatement 透传，依赖服务端预编译二进制协议的冷门驱动可能踩坑；MySQL 8.0 默认 `caching_sha2_password` 认证在 1.6 上需要把后端账号改为 `mysql_native_password`，或升级到已适配的 1.6.7.5+ 版本。

## 2.2 核心模块分工

| 模块 | 职责 | 核心实现（1.6 → 2.x） |
|------|------|----------------------|
| **Protocol** | MySQL 协议编解码 | `FrontendConnection` → `MySQLPacketBuilder` |
| **Router** | SQL 路由（确定目标节点） | `ServerSchema`/`EntraRouter` → `DrdsRouter` + Hint |
| **Parser** | SQL 解析 | 自研 RumpParser + Druid → Apache Calcite |
| **Executor** | 分布式执行 | `ResultSetNode`/`MergeNode` → `ExecutionPlan` |
| **Merger** | 结果聚合 | `HeapMerger`/`AggregateMerger` → `ResultSetMerger` |
| **Heartbeat** | 主从状态检测 | `MySQLHeartbeat` → `HeartbeatDetector` |
| **Config** | 配置管理 | `ConfigLoader`（XML）→ `ConfigCenter`（JSON 热加载） |

# 3. 三张配置的分工（1.6）

**Mycat 1.6 的行为完全由 conf 下三个 XML 决定**，职责边界清晰到可以背下来——这是排障的第一性原理：连不上查 server.xml，路由不对查 schema.xml，分片落错节点查 rule.xml。

| 文件 | 管什么 | 关键元素 | 生效方式 |
|------|--------|---------|---------|
| `server.xml` | 系统级：端口、线程、用户、权限 | `<System>`、`<Manager>`、`<User>` | 重启 |
| `schema.xml` | 逻辑库拓扑：schema → dataNode → dataHost → 物理 MySQL 与主从 | `<schema>`、`<dataNode>`、`<dataHost>`、`<writeHost>/<readHost>` | 9066 `reload @@config-all` |
| `rule.xml` | 分片规则：逻辑表按什么算法落哪个 dataNode | `<tableRule>`、`<function>` | `reload @@config-all` |

**三者引用链**：schema.xml 里 `<table>` 的 `dataNode="dn1,dn2"` 给出候选节点集合、`rule="partition-by-mod"` 指向 rule.xml 的 `<tableRule>`；`<dataNode>` 又引用 `<dataHost>`。**一句话概括：schema.xml 告诉"有哪些家"，rule.xml 告诉"按什么规矩进哪家门"**。三张 XML 的完整可粘贴骨架见第 3 篇《快速开始》，这里不重复；只需记住骨架层级：

```text
schema.xml: <schema> ─ <table dataNode rule> → <dataNode dataHost database> → <dataHost> ─ <writeHost> ─ <readHost>
rule.xml:   <tableRule name> ─ <rule: columns + algorithm> → <function name class> ─ <property>
server.xml: <System(端口/线程)> + <Manager(9066 账号)> + <User(8066 账号 → schemas 白名单)>
```

server.xml 中 `<User>` 定义的是业务账号（可访问哪些 schema），`<Manager>` 定义的是 9066 管理账号——**两套账号体系互不相通**，用业务账号连 9066 或管理账号连 8066 都会被拒绝。

> **设计争议**：三配置分工是 Mycat 被吐槽最多的设计之一——同一张表的完整拓扑信息被拆散在 schema.xml 与 rule.xml 两处，`dataNode` 列表和 `count` 必须人工保持一致（`count` 不等于 dataNode 数就会路由越界报错），新手上手极易配错。对比之下 ShardingSphere 用 Groovy/YAML 内联表达式把"规则"与"拓扑"写在一起，Mycat2 的 JSON 配置也做了合并。

# 4. 与 ShardingSphere-Proxy 架构对比

同为 Proxy 路线，两者架构哲学差异值得单独一张表：

| 维度 | Mycat 1.6 | ShardingSphere-Proxy |
|------|-----------|---------------------|
| 网络框架 | 手写 NIO Reactor | Netty |
| SQL 引擎 | 自研解析 + 规则改写 | ANTLR 解析 + 自研路由/改写/归并 |
| 分表支持 | 只分库，逻辑表:物理表 = 1:1 | 分库 + 分表，1:N 灵活映射 |
| 读写分离 | 内置在 schema.xml（balance/switchType） | 独立 DistSQL/规则，支持 MGR/GaussianDB 等 |
| 权限模型 | 用户-schema 级白名单 | 用户-库-表-列级，接入外部权限体系 |
| 治理特性 | 弱（仅连接限流、sqlMaxTimeout） | 影子库压测、数据脱敏、可观测 OpenTracing |
| 元数据一致性 | 本地 XML，多实例各自为政（可配集群模式） | 注册中心（Governance Center）统一推送 |
| 异构数据库 | 仅 MySQL | MySQL/PostgreSQL/openGauss 等 |

**Mycat 多实例的一致性问题**：多个 Mycat 节点各读各的本地 XML，改配置要逐台 reload，且全局序列、切换状态靠 ZooKeeper 集群模式同步；ShardingSphere 从 4.x 起把"配置即状态"收敛到注册中心，这是两者在云原生适配上的代差。**设计考量**：

- **要极简与可控选 Mycat**：三个 XML 一目了然，内存占用低（1GB 级即可跑），适合 DBA 团队直接运维
- **要治理与生态选 ShardingSphere**：影子压测、脱敏、分布式元数据这些能力 Mycat 全缺
- **两者都不能解决"跨分片事务弱"的问题**，Proxy 架构真正的天花板在事务与复杂 SQL，不在网络层

# 5. 配置热加载与运行时管理（对照）

**Mycat 1.6 的 reload 是"全量重建逻辑模型"**：改完 XML 在 9066 执行 `RELOAD @@CONFIG`（schema + rule）或 `RELOAD @@CONFIG ALL`，正在执行中的 SQL 不受影响，但 `processors` 这类 server.xml 参数必须重启。Mycat 2.x 则是改 JSON / 调 Admin API 后由配置中心 diff、增量更新路由表、动态扩缩数据源，仅 server 级参数可能需重启。**运维铁律：reload 前先备份当前生效配置（`SHOW @@CONFIG` 截图存档），改错了能回退**。

# 6. 小结

**Mycat 1.6 的架构可以压缩成三句话**：进程骨架是"Acceptor + N 组 Reactor/Processor"的手写 NIO 模型，连接终身绑定 Processor 实现故障隔离；行为由 server.xml（系统）、schema.xml（拓扑）、rule.xml（路由规则）三张 XML 分工决定；前端连接与后端连接池解耦，Proxy 借此收敛 MySQL 连接数。**2.x 则把地基换成 Netty + Aloha、把 SQL 引擎换成 Calcite**，代价是与 1.6 生态彻底割裂。对比 ShardingSphere-Proxy，Mycat 赢在轻与简，输在分表粒度、元数据治理与异构支持——选型时按"团队要可控还是要治理"决策。
