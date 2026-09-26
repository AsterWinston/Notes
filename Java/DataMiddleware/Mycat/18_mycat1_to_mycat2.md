
# 1. 架构差异

| 维度 | Mycat 1.6.x | Mycat 2.x |
|------|------------|-----------|
| **I/O 模型** | BIO（每个连接一个线程） | NIO（Vert.x 事件驱动） |
| **SQL 解析** | Druid 解析器 | Apache Calcite |
| **配置文件** | schema.xml + server.xml + rule.xml（XML） | datasources.json + schemas.json + clusters.json + users.json（JSON） |
| **读写分离** | 在 schema.xml 中配置 dataHost | 在 clusters.json 中独立配置 |
| **分片算法** | 在 rule.xml 中定义 | 在 schemas.json 中内置 |
| **事务** | 弱 XA 支持 | 原生 XA 支持 |
| **动态配置** | 不支持，需重启 | 支持热加载 |
| **管理端口** | 9066 | 9066 |
| **心跳** | dataHost 配置 | 数据源级配置 |

# 2. 配置文件映射

## 2.1 schema.xml → schemas.json

**Mycat 1.6 (schema.xml)**：

```xml
<schema name="order_db" checkSQLschema="true" sqlMaxLimit="100">
    <table name="orders" primaryKey="id"
           dataNode="dn0,dn1" rule="mod-long" />
</schema>
<dataNode name="dn0" dataHost="dh1" database="order_db_0" />
<dataNode name="dn1" dataHost="dh2" database="order_db_1" />
```

**Mycat 2.x (schemas.json)**：

```json
{
  "schemas": {
    "order_db": {
      "tables": {
        "orders": {
          "type": "sharding",
          "dataNodes": "dn0.orders,dn1.orders",
          "sharding": {
            "function": {
              "clazz": "io.mycat.router.function.PartitionByMod",
              "properties": { "columnName": "id", "count": 2 }
            }
          }
        }
      }
    }
  }
}
```

## 2.2 dataHost → datasources.json + clusters.json

**Mycat 1.6 (schema.xml)**：

```xml
<dataHost name="dh1" maxCon="100" minCon="10" balance="1"
          writeType="0" dbType="mysql" dbDriver="native">
    <heartbeat>select user()</heartbeat>
    <writeHost host="master1" url="192.168.1.10:3306"
               user="root" password="root" />
    <readHost host="slave1" url="192.168.1.11:3306"
              user="root" password="root" />
</dataHost>
```

**Mycat 2.x (datasources.json + clusters.json)**：

```json
// datasources.json
{
  "prototype": {
    "name": "master1",
    "url": "jdbc:mysql://192.168.1.10:3306/order_db",
    "user": "root", "password": "root",
    "minCon": 10, "maxCon": 100
  },
  "slave1": {
    "name": "slave1",
    "url": "jdbc:mysql://192.168.1.11:3306/order_db",
    "user": "root", "password": "root"
  }
}

// clusters.json
{
  "clusters": {
    "dh1": {
      "clusterType": "MASTER_SLAVE",
      "readBalanceType": "BALANCE_ALL",
      "master": "master1",
      "replicas": ["slave1"]
    }
  }
}
```

# 3. 分片算法迁移

| Mycat 1.6 | Mycat 2.x |
|-----------|-----------|
| `PartitionByMod` | `io.mycat.router.function.PartitionByMod` |
| `PartitionByLong` | `io.mycat.router.function.PartitionByRange` |
| `PartitionByMurmurHash` | `io.mycat.router.function.PartitionByMurmurHash` |
| `PartitionByDate` | `io.mycat.router.function.PartitionByDate` |
| `AutoIncrement` | `SnowflakeSequenceHandler` |

# 4. SQL 兼容性变更

| 功能 | Mycat 1.6 | Mycat 2.x |
|------|----------|-----------|
| JOIN | 有限支持 | ER 表 + 全局表增强 |
| 子查询 | 部分 | 更完善（Calcite 优化） |
| 存储过程 | 部分 | 不支持（设计限制） |
| XA 事务 | 实验性 | 正式支持 |
| ORDER BY + LIMIT | 支持 | 优化（LIMIT 下推） |
| MySQL 8.0 语法 | 不支持 | 支持 |

# 5. 迁移步骤

```
第 1 步：环境准备
  - 部署 Mycat2 实例（新端口）
  - 编写新格式配置文件

第 2 步：配置验证
  - 使用 Mycat2 连接测试库验证
  - 执行常见 SQL 确保路由正确

第 3 步：数据迁移（如分片规则变更）
  - 使用 mysqldump + 脚本重新分布数据
  - 或使用 Canal 增量同步

第 4 步：灰度上线
  - 先切换 1% 流量 → Mycat2
  - 观察无异常 → 逐步切换到 100%
  - 保留 Mycat1.6 作为回退

第 5 步：完全切换
  - 关闭 Mycat1.6
  - 监控 Mycat2 稳定性
```

# 6. 不兼容项

| 项目 | Mycat 1.6 | Mycat 2.x | 应对 |
|------|----------|-----------|------|
| ZK 配置中心 | 内置 | 需配置 | 启用 Nacos/ZK 配置中心 |
| 管理命令 | `show @@datasource` | `/*!mycat:showDataSources*/` | 更新运维脚本 |
| 全局表 | `type="global"` | `"type": "global"` | 配置迁移 |
| 自增 ID | `autoIncrement="true"` | `"autoIncrement": true` | 配置迁移 |
| 日志格式 | 自定义 | 标准化 | 更新日志收集 |
