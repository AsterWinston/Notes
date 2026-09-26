
# 1. 部署架构

```
                    ┌──────────────┐
                    │    Nacos     │  ← 注册中心 / 配置中心
                    │   Cluster    │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ TC Node1 │ │ TC Node2 │ │ TC Node3 │  ← TC Cluster
        └────┬─────┘ └────┬─────┘ └────┬─────┘
             │            │            │
             └────────────┼────────────┘
                          │
                   ┌──────▼──────┐
                   │  MySQL (DB) │  ← 共享存储
                   │  (seata库)  │
                   └─────────────┘
```

# 2. DB 存储模式（生产推荐）

```yaml
# TC Server 1 (tc-node1)
server:
  port: 7091

seata:
  registry:
    type: nacos
    nacos:
      application: seata-server
      server-addr: nacos1:8848,nacos2:8848,nacos3:8848
      namespace: seata
      group: SEATA_GROUP
      cluster: default

  config:
    type: nacos
    nacos:
      server-addr: nacos1:8848,nacos2:8848,nacos3:8848
      namespace: seata
      group: SEATA_GROUP
      data-id: seata.properties

  store:
    mode: db
    db:
      datasource: druid
      db-type: mysql
      driver-class-name: com.mysql.cj.jdbc.Driver
      url: jdbc:mysql://mysql1:3306,mysql2:3306/seata?useUnicode=true
      user: root
      password: root
      min-conn: 10
      max-conn: 100
```

# 3. 数据库表

```sql
CREATE DATABASE seata;
USE seata;

-- 全局事务表
CREATE TABLE `global_table` (
  `xid`                       VARCHAR(128) NOT NULL,
  `transaction_id`            BIGINT,
  `status`                    TINYINT      NOT NULL,
  `application_id`            VARCHAR(32),
  `transaction_service_group` VARCHAR(32),
  `transaction_name`          VARCHAR(128),
  `timeout`                   INT,
  `begin_time`                BIGINT,
  `application_data`          VARCHAR(2000),
  `gmt_create`                DATETIME,
  `gmt_modified`              DATETIME,
  PRIMARY KEY (`xid`),
  KEY `idx_status_gmt_modified` (`status`, `gmt_modified`),
  KEY `idx_transaction_id` (`transaction_id`)
) ENGINE=InnoDB;

-- 分支事务表
CREATE TABLE `branch_table` (
  `branch_id`         BIGINT       NOT NULL,
  `xid`               VARCHAR(128) NOT NULL,
  `transaction_id`    BIGINT,
  `resource_group_id` VARCHAR(32),
  `resource_id`       VARCHAR(256),
  `branch_type`       VARCHAR(8),
  `status`            TINYINT,
  `client_id`         VARCHAR(64),
  `application_data`  VARCHAR(2000),
  `gmt_create`        DATETIME,
  `gmt_modified`      DATETIME,
  PRIMARY KEY (`branch_id`),
  KEY `idx_xid` (`xid`)
) ENGINE=InnoDB;

-- 全局锁表
CREATE TABLE `lock_table` (
  `row_key`        VARCHAR(128) NOT NULL,
  `xid`            VARCHAR(128) NOT NULL,
  `transaction_id` BIGINT,
  `branch_id`      BIGINT       NOT NULL,
  `resource_id`    VARCHAR(256),
  `table_name`     VARCHAR(32),
  `pk`             VARCHAR(36),
  `status`         TINYINT      NOT NULL DEFAULT 0,
  `gmt_create`     DATETIME,
  `gmt_modified`   DATETIME,
  PRIMARY KEY (`row_key`),
  KEY `idx_xid` (`xid`)
) ENGINE=InnoDB;
```

# 4. Client 多 TC 地址配置

```yaml
seata:
  tx-service-group: default_tx_group
  registry:
    type: nacos
    nacos:
      application: seata-server   # TC 在 Nacos 中的服务名
      server-addr: nacos1:8848
      namespace: seata
      group: SEATA_GROUP
```

Client 通过 Nacos 自动发现 TC 节点列表，支持**自动故障转移**。

# 5. 故障转移流程

```
1. Client 连接到 TC Node1
2. TC Node1 宕机
3. Client 检测到连接断开
4. 从 Nacos 获取 TC 节点列表
5. 选择 TC Node2 重连
6. TC Node2 从 DB 恢复未完成的全局事务
7. 继续处理
```

# 6. TC 集群配置（raft 模式）

```yaml
# TC Server 3 节点 Raft 集群
seata:
  server:
    raft:
      group: default
      server-addr: 192.168.1.10:7091,192.168.1.11:7091,192.168.1.12:7091
      snapshot-interval: 600
      apply-batch: 32
      max-append-buffer-size: 262144
      max-replicator-inflight-msgs: 256
      election-timeout-ms: 1000
      disruptor-buffer-size: 16384
```

# 7. 健康检查与监控

```bash
# TC Server 健康检查
curl http://localhost:7091/health
# → {"status":"UP"}

# 查询 TC 集群节点
curl http://localhost:7091/metadata/v1/cluster-info
```

# 8. 容器化部署

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: seata-server
spec:
  serviceName: seata-server
  replicas: 3
  template:
    spec:
      containers:
        - name: seata-server
          image: seataio/seata-server:1.7.0
          ports:
            - containerPort: 7091
            - containerPort: 8091
          env:
            - name: SEATA_PORT
              value: "8091"
            - name: STORE_MODE
              value: "db"
          readinessProbe:
            httpGet:
              path: /health
              port: 7091
            initialDelaySeconds: 30
```
