
# 1. 整体架构

Mycat2 采用**反应式管道架构**，每个请求经过一系列处理器：

```
Client Request (MySQL Protocol)
  │
  ▼
┌─────────────────────────────────────────────────────┐
│                  Mycat2 Runtime                       │
│                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│  │ Protocol │→│  Router  │→│ Calcite  │          │
│  │  Layer   │  │  (路由)  │  │ Optimizer│          │
│  └──────────┘  └──────────┘  └────┬─────┘          │
│                                   │                  │
│  ┌──────────┐  ┌──────────┐  ┌───▼──────┐          │
│  │  Result  │←│ Executor │←│  Planner │          │
│  │  Merger  │  │ (执行器) │  │ (计划器) │          │
│  └──────────┘  └──────────┘  └──────────┘          │
│       │              │                               │
└───────┼──────────────┼───────────────────────────────┘
        │              │
        ▼              ▼
    Response    Backend MySQL(s)
```

# 2. Vert.x 反应式内核

Mycat2 基于 Vert.x 实现全异步非阻塞 I/O：

```java
// Vert.x Event Loop 模型
┌──────────────────────────────────────┐
│            Vert.x Instance           │
│                                      │
│  ┌────────┐  ┌────────┐  ┌────────┐ │
│  │Event   │  │Event   │  │Event   │ │
│  │Loop 0  │  │Loop 1  │  │Loop N  │ │  ← 默认 = CPU 核数 × 2
│  └───┬────┘  └───┬────┘  └───┬────┘ │
│      │           │           │       │
│      ▼           ▼           ▼       │
│  Worker Pool (阻塞任务隔离)           │
└──────────────────────────────────────┘
```

**关键特性**：
- 每个 Event Loop 单线程处理所有 I/O 事件（无锁）
- 不允许在 Event Loop 中执行阻塞操作
- 阻塞任务（如后端 JDBC 调用）提交到 Worker Pool

# 3. 协议层

Mycat2 完全模拟 MySQL Server 协议：

```
客户端 JDBC
  │
  ├── 连接握手 (Handshake)
  │     └── Mycat2 返回伪造的 MySQL Server 版本信息
  │
  ├── 认证
  │     └── users.json 配置用户名/密码
  │
  ├── 命令执行
  │     ├── COM_QUERY (普通查询)
  │     ├── COM_STMT_PREPARE (预编译)
  │     └── COM_STMT_EXECUTE (执行预编译)
  │
  └── SET 变量 / SHOW 命令
        └── 部分透明转发，部分需 Mycat2 处理
```

**支持的协议命令**：

| 命令 | 说明 | 处理方式 |
|------|------|---------|
| `COM_QUERY` | 普通查询 | Mycat2 解析路由 |
| `COM_STMT_PREPARE` | 预编译 | Mycat2 模拟 |
| `COM_STMT_EXECUTE` | 执行预编译 | Mycat2 模拟 |
| `COM_PING` | 心跳 | Mycat2 直接响应 |
| `COM_QUIT` | 断开 | 释放连接 |
| `COM_INIT_DB` | 切换数据库 | 切换 Schema |

# 4. 模块化处理器

## 4.1 请求处理管道

```java
// 伪代码：Mycat2 的请求处理链
public class MySQLReactor {

    public void handleQuery(FrontendConnection conn, String sql) {
        // 1. 协议解析 → 提取 SQL
        // 2. 路由 → 确定目标数据节点
        RouteResult route = router.route(sql, conn.getSchema());

        // 3. Calcite 优化 → 生成执行计划
        RelNode optimized = optimizer.optimize(sql, route);

        // 4. 执行 → 分发到后端 MySQL
        List<Future<ResultSet>> futures = executor.execute(optimized, route);

        // 5. 合并 → 聚合多个结果集
        ResultSet merged = merger.merge(futures);

        // 6. 响应 → 写回 MySQL 协议包
        protocol.write(conn, merged);
    }
}
```

## 4.2 核心模块

| 模块 | 职责 | 核心类 |
|------|------|--------|
| **Protocol** | MySQL 协议编解码 | `MySQLPacketBuilder` |
| **Router** | SQL 路由（确定目标节点） | `DrdsRouter`, `HintHandler` |
| **Calcite Adapter** | SQL 解析 + 优化 | `MycatCalciteSupport` |
| **Executor** | 分布式执行 | `ExecutionPlan` |
| **Merger** | 结果聚合 | `ResultSetMerger` |
| **Heartbeat** | 心跳检测 | `HeartbeatDetector` |
| **Config** | 配置管理 | `ConfigCenter` |

# 5. 数据源连接池

```
┌────────────────────────────────────┐
│         Mycat2 连接管理             │
│                                    │
│  Frontend Connections (客户端)      │
│  ├── maxCon: 10000                 │
│  └── 基于 Vert.x 异步连接           │
│                                    │
│  Backend Connections (后端 MySQL)   │
│  ├── 每个数据源独立连接池            │
│  ├── 最小连接数 / 最大连接数         │
│  ├── 连接超时 / 空闲超时            │
│  └── 每个数据节点动态扩缩            │
└────────────────────────────────────┘
```

# 6. 配置文件体系

Mycat2 使用 JSON 格式配置文件，位于 `conf/` 目录：

| 文件 | 作用 | 核心配置 |
|------|------|---------|
| `datasources.json` | 数据源定义 | 连接地址、用户名/密码、连接池 |
| `schemas.json` | 逻辑 Schema + 分片规则 | 表映射、分片算法、全局表 |
| `clusters.json` | 集群定义 | 主从/集群节点、读写分离类型 |
| `users.json` | 用户与权限 | 用户名/密码、可访问 Schema、IP 白名单 |

# 7. 配置热加载

```
Admin API (HTTP)                 Mycat2 Runtime
      │                               │
      │ PUT /config/schemas            │
      │ ───────────────────────────→   │
      │                               │ 1. 校验新配置
      │                               │ 2. 对比差异
      │                               │ 3. 增量更新路由表
      │                               │ 4. 关闭旧数据源
      │                               │ 5. 打开新数据源
      │ ←── { "status": "ok" } ─────  │
      │                              │
```
