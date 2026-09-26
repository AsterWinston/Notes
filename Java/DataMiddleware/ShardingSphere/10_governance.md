
# 1. 治理中心架构

ShardingSphere 治理中心（Governance Center）提供基于注册中心的集群管理能力。

```
┌──────────────────────────────────────────┐
│           Governance Center               │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ │
│  │ Registry │ │  Config  │ │  Circuit  │ │
│  │  Center  │ │  Center  │ │  Breaker  │ │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ │
│       │            │            │        │
│  ┌────┴────────────┴────────────┴─────┐  │
│  │          Mode (Cluster)            │  │
│  │    ZooKeeper / etcd / Nacos        │  │
│  └────────────────────────────────────┘  │
└──────────────────────────────────────────┘
```

# 2. Mode — 运行模式

## 2.1 Standalone 模式

```yaml
mode:
  type: Standalone
  repository:
    type: JDBC
    props:
      jdbc-url: jdbc:mysql://localhost:3306/shardingsphere
      username: root
      password: root
```

## 2.2 Cluster 模式

```yaml
mode:
  type: Cluster
  repository:
    type: ZooKeeper
    props:
      namespace: governance
      server-lists: zk1:2181,zk2:2181,zk3:2181
      retryIntervalMilliseconds: 500
      timeToLiveSeconds: 60
```

# 3. 注册中心

ShardingSphere 支持多种注册中心：

| 类型 | 适用场景 |
|------|---------|
| ZooKeeper | **推荐**，成熟稳定 |
| etcd | 云原生环境 |
| Nacos | Spring Cloud Alibaba 生态 |
| Consul | HashiCorp 生态 |

```yaml
# Nacos 注册中心
mode:
  type: Cluster
  repository:
    type: Nacos
    props:
      namespace: shardingsphere
      server-addr: nacos1:8848,nacos2:8848,nacos3:8848
```

# 4. 治理能力

## 4.1 熔断

```yaml
props:
  circuit-breaker-enabled: true
```

```sql
-- DistSQL 禁用/启用从库
DISABLE READWRITE_SPLITTING READ ds_slave0 FROM readwrite_ds;
ENABLE READWRITE_SPLITTING READ ds_slave0 FROM readwrite_ds;
```

## 4.2 禁用存储单元

```sql
-- 临时摘除故障数据源
DISABLE STORAGE UNIT ds2;
-- 故障恢复后启用
ENABLE STORAGE UNIT ds2;
```

## 4.3 数据库发现

```yaml
rules:
  - !DB_DISCOVERY
    dataSources:
      ds_mgr:
        dataSourceNames:
          - ds_master
          - ds_slave0
          - ds_slave1
        discoveryHeartbeatName: mgr_heartbeat
        discoveryTypeName: mgr
    discoveryHeartbeats:
      mgr_heartbeat:
        props:
          keep-alive-cron: "0/5 * * * * ?"
    discoveryTypes:
      mgr:
        type: MGR
```

# 5. 权限与认证

```yaml
authority:
  users:
    - user: root@%
      password: root
    - user: readonly@%
      password: readonly
  privilege:
    type: ALL_PRIVILEGES         # ALL_PRIVILEGES / SCHEMA_PRIVILEGES
  authenticator:
    type: MySQL
```

```sql
-- 创建只读用户
CREATE USER readonly@'%' IDENTIFIED BY 'readonly';
GRANT SELECT ON sharding_db.* TO readonly@'%';
```

# 6. 集群元数据持久化

Cluster 模式下，ShardingSphere 将以下信息持久化到注册中心：

```
/shardingsphere/
  ├── metadata/           # 元数据（数据源、表结构）
  ├── rules/              # 规则配置（分片、加密等）
  ├── props/              # 属性配置
  └── nodes/              # 集群节点信息
```

所有 ShardingSphere-Proxy 实例共享同一份注册中心的配置，任一实例通过 DistSQL 修改规则后，其他实例自动同步。

# 7. 节点状态

```sql
-- 查看集群节点
SHOW INSTANCE LIST;

-- 输出示例：
-- +----------------+-----------+-------+
-- | instance_id    | host      | port  |
-- +----------------+-----------+-------+
-- | proxy-1        | 10.0.0.1  | 3307  |
-- | proxy-2        | 10.0.0.2  | 3307  |
-- +----------------+-----------+-------+
```

# 8. 最佳实践

1. **生产必用 Cluster 模式**：Standalone 仅开发/测试使用
2. **注册中心高可用**：ZooKeeper 至少 3 节点
3. **定期备份元数据**：`EXPORT DATABASE CONFIGURATION`
4. **DistSQL 做变更**：Cluster 模式下 DistSQL 会自动同步到所有节点
5. **熔断与告警联动**：从库被禁用时触发告警
