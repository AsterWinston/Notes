
# 1. 配置层次

Seata 的配置分为两层：

```
┌──────────────────────────────────┐
│         TC Server 配置            │
│   (seata.properties / Nacos)     │
│   注册中心、配置中心、存储模式      │
│   服务端口、事务日志保留策略        │
└──────────────┬───────────────────┘
               │
┌──────────────▼───────────────────┐
│         Client 配置               │
│   (application.yml / registry.conf)│
│   事务分组、TM/RM 参数、超时、重试  │
└──────────────────────────────────┘
```

# 2. TC Server 配置

## 2.1 application.yml

```yaml
server:
  port: 7091

spring:
  application:
    name: seata-server

seata:
  config:
    type: nacos
    nacos:
      server-addr: 127.0.0.1:8848
      namespace: seata
      group: SEATA_GROUP
      data-id: seata.properties

  registry:
    type: nacos
    nacos:
      application: seata-server
      server-addr: 127.0.0.1:8848
      namespace: seata
      group: SEATA_GROUP

  store:
    mode: db
    db:
      datasource: druid
      db-type: mysql
      driver-class-name: com.mysql.cj.jdbc.Driver
      url: jdbc:mysql://127.0.0.1:3306/seata?useUnicode=true
      user: root
      password: root
      min-conn: 10
      max-conn: 100
```

## 2.2 seata.properties（Nacos 中）

```properties
# ========== 事务分组映射 ==========
service.vgroupMapping.default_tx_group=default

# ========== 存储 ==========
store.mode=db
store.db.datasource=druid
store.db.dbType=mysql
store.db.driverClassName=com.mysql.cj.jdbc.Driver
store.db.url=jdbc:mysql://127.0.0.1:3306/seata?useUnicode=true
store.db.user=root
store.db.password=root

# ========== 服务端事务管理 ==========
server.undo.logSaveDays=7
server.undo.logDeletePeriod=86400000
server.maxCommitRetryTimeout=120000
server.maxRollbackRetryTimeout=120000

# ========== 服务端通信 ==========
service.enableDegrade=false
service.disableGlobalTransaction=false
```

# 3. Client 配置

## 3.1 Spring Boot 配置项

```yaml
seata:
  # 事务分组（映射到 TC 集群）
  tx-service-group: default_tx_group

  # 注册中心
  registry:
    type: nacos
    nacos:
      server-addr: localhost:8848
      namespace: seata
      application: seata-server
      group: SEATA_GROUP

  # 配置中心
  config:
    type: nacos
    nacos:
      server-addr: localhost:8848
      namespace: seata
      group: SEATA_GROUP
      data-id: seata.properties

  # 数据源代理模式
  data-source-proxy-mode: AT    # AT / XA

  # 客户端 TM 配置
  client:
    tm:
      commit-retry-count: 5
      rollback-retry-count: 5
      default-global-transaction-timeout: 300000

    # 客户端 RM 配置
    rm:
      async-commit-buffer-limit: 10000
      report-retry-count: 5
      table-meta-check-enable: true
      report-success-enable: false
      saga-branch-register-enable: false

    # 全局锁（AT 模式）
    lock:
      retry-interval: 10       # 获取全局锁重试间隔(ms)
      retry-times: 30           # 获取全局锁重试次数
      retry-policy-branch-rollback-on-conflict: true

    # undo_log 配置
    undo:
      data-validation: true    # 镜像数据校验
      log-serialization: jackson
      log-table: undo_log
      only-care-update-columns: true  # 仅记录变更列
```

# 4. 支持的后端类型

## 4.1 配置中心

| 类型 | 配置值 | 特点 |
|------|--------|------|
| Nacos | `nacos` | **推荐**，阿里出品，功能完善 |
| Apollo | `apollo` | 携程出品，可视化配置管理 |
| ZooKeeper | `zk` | 分布式协调 |
| Consul | `consul` | HashiCorp 出品 |
| Etcd | `etcd3` | CoreOS 出品 |
| File | `file` | 本地文件，仅开发用 |

## 4.2 注册中心

| 类型 | 用途 |
|------|------|
| Nacos | **推荐**，同时作为注册和配置中心 |
| Eureka | Netflix 服务发现 |
| ZooKeeper | 分布式协调 |
| Redis | 轻量场景 |
| File | 本地配置，仅开发 |

## 4.3 存储模式

| 模式 | 配置值 | 特点 | 适用场景 |
|------|--------|------|---------|
| DB | `db` | MySQL/Oracle/PG 存储 | **生产推荐** |
| File | `file` | 本地文件存储 | 开发测试 |
| Redis | `redis` | Redis 存储 | 中等规模 |

# 5. 事务分组映射

```yaml
# Client 端
seata:
  tx-service-group: my_tx_group    # 事务分组名

# TC Server 端（seata.properties）
# 将事务分组映射到 TC 集群名
service.vgroupMapping.my_tx_group=cluster_a
service.vgroupMapping.order_tx_group=cluster_b
```

**事务分组的作用**：
1. 逻辑隔离：不同业务使用不同 TC 集群
2. 灰度发布：新版本服务使用新分组，指向灰度 TC
3. 多机房：每个机房一个 TC 集群，按分组路由

# 6. 多环境配置

```yaml
# application-dev.yml
seata:
  tx-service-group: default_tx_group
  registry:
    nacos:
      server-addr: localhost:8848
      namespace: dev

---
# application-prod.yml
seata:
  tx-service-group: prod_tx_group
  registry:
    nacos:
      server-addr: nacos.prod:8848
      namespace: production
```

# 7. 全局事务超时配置层次

```
1. @GlobalTransactional(timeoutMills = 60000)    ← 最高优先级
2. seata.client.tm.default-global-transaction-timeout
3. TC Server: server.maxCommitRetryTimeout
4. Dubbo/Feign: connectTimeout + readTimeout
```

**超时应满足**：`Dubbo/Feign timeout > Seata 事务超时 > 业务执行时间`

# 8. 常用 JVM 参数

```bash
# 指定配置文件位置（非 Spring Boot 项目）
-Dseata.config.type=file
-Dseata.config.file.registry.conf=/path/to/registry.conf

# 指定事务分组
-Dseata.tx-service-group=default_tx_group

# 强制开启日志
-Dseata.log.level=DEBUG

# 跳过全局事务（调试用）
-Dseata.disable.global.transaction=true
```
