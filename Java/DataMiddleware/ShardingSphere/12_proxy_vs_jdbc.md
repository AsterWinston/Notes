
# 1. 架构差异

```
ShardingSphere-JDBC                    ShardingSphere-Proxy
┌──────────────────┐                   ┌──────────┐
│   Application    │                   │   App    │  ← 任意语言
│ ┌──────────────┐ │                   └────┬─────┘
│ │ Sharding-JDBC│ │                        │ MySQL Protocol
│ │   (SDK)      │ │                   ┌────▼─────┐
│ └──────┬───────┘ │                   │  Proxy   │
│        │ JDBC    │                   └────┬─────┘
└────────┼─────────┘                        │ JDBC
         ▼                                  ▼
   ┌──────────┐                       ┌──────────┐
   │  MySQL   │                       │  MySQL   │
   └──────────┘                       └──────────┘

网络：0 跳                               网络：+1 跳
```

# 2. 功能对比

| 维度 | JDBC | Proxy |
|------|------|------|
| **数据分片** | ✓ | ✓ |
| **读写分离** | ✓ | ✓ |
| **数据加密** | ✓ | ✓ |
| **影子库** | ✓ | ✓ |
| **分布式事务** | ✓（内置） | ✓（内置） |
| **DistSQL** | ✗ | ✓ |
| **治理中心** | 可选（注册中心） | 内置 |
| **可观测性** | Agent 插件 | Agent 插件 |
| **异构语言** | ✗（仅 Java） | ✓ |
| **连接池** | 应用自行管理 | Proxy 统一管理 |

# 3. 性能对比

```
基准测试：4 分片，单表 1000 万行，查询 WHERE shardkey = ?

| 模式               | QPS     | AVG RT | P99 RT |
|--------------------|---------|--------|--------|
| 直连 MySQL（无分片）| 30000   | 3ms    | 8ms    |
| ShardingSphere-JDBC | 28000   | 4ms    | 10ms   |
| ShardingSphere-Proxy| 15000   | 7ms    | 18ms   |

JDBC 模式性能损耗约 7%，Proxy 模式约 50%。
Proxy 额外开销主要来自：网络转发 + MySQL 协议编解码。
```

# 4. 运维复杂度对比

| 维度 | JDBC | Proxy |
|------|------|------|
| 部署 | 随应用发布 | 独立部署维护 |
| 配置变更 | 修改 YAML + 重启 | DistSQL 在线变更 |
| 多应用共享 | 各自配置 | 共享 Proxy |
| 连接数管理 | 应用 1:N DB | 应用 M:1 Proxy 1:N DB |
| 升级 | 依赖冲突风险 | 独立平滑升级 |
| 高可用 | 随应用 | 需独立实现（LB + 多实例） |

# 5. 混合部署

```
┌─────────────────────────────────────────┐
│             混合架构                      │
│                                          │
│  Java 应用 → ShardingSphere-JDBC (直连)   │
│  Go 应用   → ShardingSphere-Proxy → DB   │
│  Python    → ShardingSphere-Proxy → DB   │
│                                          │
│  共享同一套 MySQL 集群和分片规则           │
└─────────────────────────────────────────┘
```

**混合部署注意事项**：
- JDBC 和 Proxy 必须使用相同的分片规则
- Cluster 模式下通过注册中心同步配置
- Proxy 使用 DistSQL 变更 → JDBC 应用重启生效（或动态加载）

# 6. 选型决策

```
是否需要支持非 Java 语言？
  ├── 是 → ShardingSphere-Proxy
  └── 否 → 追求极致性能？
            ├── 是 → ShardingSphere-JDBC
            └── 否 → 需要 DistSQL 在线变更？
                      ├── 是 → ShardingSphere-Proxy
                      └── 否 → 任选
```

# 7. 迁移路径

```
阶段 1：单库 → ShardingSphere-JDBC 分片（Java 应用）
阶段 2：需要支持其他语言 → 引入 ShardingSphere-Proxy
阶段 3：Proxy 稳定后，逐步将 Java 应用也迁移至 Proxy（统一治理）
```
