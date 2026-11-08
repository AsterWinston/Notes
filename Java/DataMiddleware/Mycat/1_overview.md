# 1. Mycat 是什么

## 1.1 数据库中间件的位置

**数据库中间件是位于应用与数据库之间的透明数据访问层**，解决的是"单库单表撑不住，但又不想换数据库"这一类问题。它不是数据库，不负责存储引擎、事务日志与查询优化器本身的实现；它做的事情是：截获 SQL、理解 SQL、把一条逻辑 SQL 拆解或改写成若干条可以在真实 MySQL 上执行的物理 SQL，再把多个结果集合并后返回。按部署形态划分，这类中间件只有两条技术路线：

| 形态 | 代表产品 | 部署位置 | 应用改造量 | 语言无关性 |
|------|---------|---------|-----------|-----------|
| Client JAR | 淘宝 TDDL、ShardingSphere-JDBC | 嵌入应用进程，在 JDBC 层拦截 | 需引入依赖、配合改造 | 绑定 JVM 语言栈 |
| Server Proxy | Cobar、Mycat、ShardingSphere-Proxy、MySQL Router | 独立进程，对外伪装成 MySQL 服务 | 近乎为零，改个 URL 即可 | 任何语言只要会说 MySQL 协议 |

**Client 路线胜在少一跳网络、天然水平扩展；Proxy 路线胜在语言无关、应用零改造、连接数可以集中治理**。对大量存量 PHP/C++/Java 混合系统、或者不允许改代码的遗留系统而言，Proxy 几乎是唯一选项——这正是 Mycat 选择的路线。代价也很直接：Mycat 本身成为架构中的单点，需要额外部署高可用（keepalived/VIP 或多实例负载均衡），并且多一跳网络延迟。

## 1.2 Mycat 的定位：一个"没有库的库"

**Mycat 官方把自己描述为"大数据分库分表中间件"，社区最常用的说法是"Mycat 不是数据库，是一个代理"**。它对上完整实现了 MySQL Server 协议——握手、认证、COM_QUERY、预编译语句都以假装的"MySQL 5.x/8.0 服务端"身份应答；对下以 MySQL Client 的身份维护到各个后端实例的连接池。应用侧把 `jdbc:mysql://真实库:3306` 改成 `jdbc:mysql://mycat:8066`，驱动、ORM、连接池全都不用动：

```
┌─────────────────────────────────────────────────┐
│                   Application                    │
│      (JDBC/ODBC，连接 Mycat 就像连接 MySQL)        │
└────────────────────┬────────────────────────────┘
                     │ MySQL Protocol (8066)
        ┌────────────▼────────────┐
        │       Mycat 1.6.x       │──── 管理端口 9066
        │  协议解析 / 路由 / 改写   │
        │  结果集归并 / 全局序列     │
        └──┬───────┬───────┬───────┘
           │       │       │
     ┌─────▼──┐ ┌──▼─────┐ ┌▼─────────┐
     │ MySQL  │ │ MySQL  │ │ MySQL    │
     │ dn1 库  │ │ dn2 库  │ │ dn3 库    │
     │ (分片)  │ │ (分片)  │ │ (全局表副本)│
     └────────┘ └────────┘ └──────────┘
```

**Mycat 的核心能力可以压缩成一张表**：

| 能力 | 说明 | 1.6.x 成熟度 |
|------|------|-------------|
| **读写分离** | 写走 writeHost，读按 balance 策略分摊到 readHost | 高，生产主力特性 |
| **分库分表** | 按 rule.xml 的分片函数把表水平拆分到多个 dataNode | 高，但只分库不分表 |
| **分布式事务** | 默认本地事务；1.6.7.4 起支持 XATX 两阶段提交 | 中，生产上普遍绕过 |
| **全局序列** | 数据库号段 / 本地文件 / ZooKeeper 三种主键方案 | 高 |
| **ER 分片与全局表** | 父子表同分片、字典表全节点复制，优化 JOIN | 高 |
| **协议兼容** | MySQL 5.x 协议；8.0 的 caching_sha2_password 认证需绕行 | 中 |

> **注意**：Mycat 1.6 只实现了 MySQL 协议的"常用子集"，存储过程、触发器、`LOAD DATA INFILE`、`LOCK TABLES`、`KILL` 等语句要么不支持要么行为受限，详见第 10 篇。把"MySQL 能做的 Mycat 都能做"当成前提上线，是新手最常见的翻车点。

# 2. 谱系：从阿里 TDDL 到京东 Mycat

## 2.1 三条血脉的来龙去脉

理解 Mycat，必须理解它身上流着两代阿里的血、一代京东的血：

```
2009  淘宝 TDDL (Client JAR 路线，Trantor 分层架构：Matrix/Group/Atom)
        │  闭源，随阿里去 IOE 一路演进
        ▼
2012  阿里开源 Cobar (Server Proxy 路线，NIO Reactor 模型)
        │  不久后停止维护，代码与架构被社区继承
        ▼
2013  Mycat 1.x 诞生 (基于 Cobar 改造增强，社区驱动)
        │  核心开发者加入京东，Mycat 进入京东生产体系
        ▼
2016  Mycat 1.6.x 定型 (1.6.7.x 成为事实上的最终版)
        │
        ▼
2019~  Mycat 2.x 立项重写 (Netty + 自研 Aloha 框架 + Calcite)
```

**TDDL 的贡献是概念而非代码**——它定义了"应用透明分库分表"的整套问题域：分片规则、全局主键、读写分离、扩容迁移，后来所有中间件都在回答 TDDL 最先提出的问题。**Cobar 的贡献是架构**——它是第一个把"用 Java NIO 手写一个 MySQL 协议代理"跑通的开源实现，Mycat 1.x 的 `ServerBootstrap / NIOAcceptor / NIOReactor / Processor` 这套包结构几乎是原样继承。2013 年前后 Cobar 停维，社区在其分库分表基础上加入全局序列、ER 表、多库多表路由等能力，取名 Mycat，寓意致敬 Cobar 的猫科命名传统（Cobar 本身是澳大利亚地名，My+Cat）。

## 2.2 京东与 Mycat2 的改写

**Mycat 与京东的关系是它区别于其他社区项目的关键背景**：1.x 时代核心维护者任职于京东，京东内部用 Mycat 支撑了交易、库存等核心链路的分库分表，1.5~1.6 的许多特性（如按日期分片、心跳优化）直接来自生产需求。2019 年起，同一支团队启动了 Mycat2 的**完全重写**：网络层从自研 NIO Reactor 换成 Netty，并在其上封装了自研异步框架 Aloha；SQL 层引入 Apache Calcite 做解析与优化；配置从一堆 XML 换成 JSON 并支持热加载。

**关键区别**：Mycat2 不是 Mycat1 的升级版，而是推倒重写，架构、配置格式、内核实现完全不同，两套配置不可混用。社区历史上还存在过一版基于 Vert.x 的早期实验性 mycat2（mycat.org.cn 提供的 1.0/1.22 安装包即出自该线），影响力有限；当前讨论"Mycat2"一般指京东团队重写的 Netty + Aloha 版本。

| 维度 | Mycat 1.6.x | Mycat 2.x |
|------|------------|-----------|
| **I/O 模型** | 自研 NIO Reactor（Selector 手写事件循环） | Netty + 自研 Aloha 异步框架 |
| **SQL 解析** | 自研轻量解析 + 借 Druid 语法树 | Apache Calcite（解析+校验+优化） |
| **查询优化** | 规则式改写 | CBO/RBO 优化器 |
| **分布式事务** | 基本靠本地事务；1.6.7.4 起有 XATX | 原生 XA，XA 管理器内置 |
| **配置方式** | server.xml / schema.xml / rule.xml，改配置需 reload 或重启 | JSON 配置，热加载 |
| **分表能力** | 只分库不分表（一逻辑表对应一物理表/分片） | 支持一库多表的逻辑分表 |
| **协议兼容** | MySQL 5.x；8.0 认证插件需降级适配 | 较完整地支持 MySQL 8.0 |
| **社区活跃度** | 基本停更（1.6.7.6 后无实质迭代） | 低速维护 |
| **生产成熟度** | 高（存量巨大） | 中 |

> **行业实践**：2026 年还新开项目直接选 Mycat 1.6 的已经很少，但存量 Mycat 1.6 系统在国内中型互联网公司里数量非常可观——它停更但足够稳定，"能跑就不动"。面试与源码阅读价值集中在 1.6：它是最适合用来学习"NIO 手写协议代理 + SQL 路由改写"的国产开源样本。

# 3. 架构演进：1.x 的 NIO 与 2.x 的 Netty + Aloha

## 3.1 Mycat 1.6.x：自己造 Reactor

Mycat 1.6 诞生时 Netty 尚未一统江湖，Cobar 系代码选择基于 JDK 原生 NIO（`Selector`/`SocketChannel`/`ByteBuffer`）**手写了一套 Reactor 多线程模型**：一个 NIOAcceptor 负责接收前端连接，若干 NIOReactor（每个含 R 读线程、W 写线程、BO 后台线程）负责事件轮询，每个前端连接按轮询固定绑定到一个 Processor 上处理请求。这套模型的优点是零第三方依赖、路径极短；缺点是缓冲池、半包处理、慢客户端流控全部自己实现，代码里大量 `ByteBuffer` 手工切片，可读性和可维护性差，也出现过若干次与 buffer pool 相关的线上 bug。

**1.x 的性能模型是"线程与连接多对一"**：连接一旦分配给某个 Processor，该连接上的请求解析、路由、SQL 改写都在这个线程上同步走完（异步的部分交给回写线程），单 Processor 线程同时承载成百上千连接。这与 Tomcat NIO 思路相似，调优重点在 `processors` 线程数、`processorBufferPool` 与后端连接池大小。

## 3.2 Mycat 2.x：站在 Netty 上重来

Mycat 2.x 把网络层整个换成 Netty，并用自研 **Aloha 框架**在其上封装了一层"异步消息 + 会话状态机"的模型：前端连接、后端连接、心跳都是消息驱动的 actor 式处理，天然适配 Netty 的 EventLoop 与零拷贝 ByteBuf。SQL 层从"够用就解析"升级为交给 Calcite 生成执行计划，跨库聚合、子查询、JOIN 的处理能力明显增强；配置文件合并为 JSON 体系并支持运行时热加载。

```java
// Mycat 1.6 与 2.x 的连接处理思路对比（示意）
// 1.6：自研 Reactor，手写 Selector 事件循环
public class NIOReactor {
    private final Selector selector;      // JDK NIO
    public void run() {
        while (!Thread.interrupted()) {
            selector.select();            // 事件轮询
            for (SelectionKey key : selector.selectedKeys()) {
                if (key.isReadable()) {
                    handler.handle(key);  // 手工读 Buffer、拼半包
                }
            }
        }
    }
}

// 2.x：Netty ChannelHandler + Aloha 异步消息
public class MySQLFrontendHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        // msg 已是解好包的 MySQLPacket，交给 Aloha 会话状态机
        session.dispatch(new QueryMessage((MySQLPacket) msg));
    }
}
```

> **设计争议**：重写是否是正确决策？Mycat 团队的答案是 1.x 的 NIO 地基难以承接 Calcite、热配置和 8.0 协议；批评者的答案是重写造成社区分裂——1.x 的文档、生态、招聘市场与 2.x 互不相认，人力又被 Apache ShardingSphere 吸走，两头都没做到绝对领先。这是中间件项目"架构债 vs 社区连续性"取舍的经典案例。

# 4. 社区现状与竞争格局

## 4.1 一超多强的分库分表中间件市场

**当前国内数据分片中间件的格局是：Apache ShardingSphere 为头号活跃项目，Mycat 守存量，云厂商卖托管产品**。ShardingSphere 走"JDBC + Proxy + Sidecar 三形态"全栈路线，进 Apache 顶级项目后以 DistSQL、异构数据源（PostgreSQL/openGauss）、弹性迁移和近乎每月一个 release 的节奏碾压了 Mycat 的声量；Mycat 的优势仅剩 1.6 的存量部署、更小的内存占用和更简单的心智模型。

| 项目 | 形态 | 维护方/治理 | 差异化 |
|------|------|------------|--------|
| Mycat 1.6 | Proxy | 社区（近似停更） | 轻量、协议代理、学习价值高 |
| Mycat 2.x | Proxy | 京东系团队（低速） | Netty + Calcite、热配置 |
| ShardingSphere-JDBC/Proxy | JAR + Proxy + Sidecar | Apache 顶级项目 | 生态最全、异构库、弹性迁移 |
| DRDS → PolarDB-X | 云服务（内核开源） | 阿里云 | TDDL 血脉的云上形态 |
| Vitess | Proxy | CNCF（YouTube 起源） | 海外云原生 MySQL 分片事实标准 |

## 4.2 适用判断

选不选 Mycat，本质是一道"约束题"而非"能力题"：

- ✓ **遗留系统无法改代码**，但单表已过千万行——Proxy 形态的 Mycat 是最省事的解法之一
- ✓ 团队已有 1.6 生产经验，规模在"十几到几十个分片"量级，业务查询模式固定
- ✓ 以读为主 + 主从架构，读写分离是核心诉求，分库只是顺带
- ✓ 学习目的：吃透 NIO 手写协议代理、SQL 路由改写、结果集归并——1.6 源码是极佳教材
- ✗ 新项目、且团队在 Java 栈内可引入依赖：优先评估 ShardingSphere-JDBC，少一跳网络少一个单点
- ✗ 需要跨异构数据库联邦查询、OLAP 分析：那是 Trino/ClickHouse 的地盘
- ✗ 需要强一致的金融级分布式事务：任何分片中间件的 XA 都不是最优解，应转向专门方案（见第 9 篇）

**设计考量**：

- **看团队而不是看产品**：Mycat 的运维资料、报错信息以中文社区经验为主，踩坑时"搜得到同类问题"本身就是生产力
- **看查询模式**：90% 查询能带上分片键、跨分片聚合少，Mycat 的短板就不会暴露；反之复杂 JOIN/子查询密集的系统，ShardingSphere 的 Calcite 路线或直上分布式 NewSQL 更稳
- **看退出成本**：Proxy 方案换中间件只需应用换连接串，这反而是 Mycat 相对 JAR 方案被低估的优势

# 5. 小结

**Mycat 是脱胎于阿里 Cobar、在京东生产体系里长成的 MySQL 协议代理型分库分表中间件**，1.6.x 用自研 NIO Reactor 支撑了国内最大的一批存量分片系统；2.x 以 Netty + Aloha + Calcite 完全重写但声量有限。理解 Mycat 的三件事：它"不是数据库"，只做 SQL 的解析、路由与归并；它的谱系是 TDDL（概念）→ Cobar（架构）→ Mycat（工程化与生产化）；它在 2026 年的位置是"守存量 + 教学价值"，新项目更多流向 ShardingSphere 或云上 PolarDB-X 一类托管方案。后续各篇以 1.6.7.x 为主线讲机制，以 2.x 作对照讲演进。
