
# 1. 架构设计最佳实践

## 1.1 命名空间规划

```
推荐命名空间划分方案:

┌─────────────────────────────────────────────────────┐
│  Namespace           用途                           │
│                                                      │
│  public (default)    ← 保留，用于默认场景           │
│  dev                 ← 开发环境                      │
│  test                ← 测试环境                      │
│  staging             ← 预发环境                      │
│  prod                ← 生产环境                      │
└─────────────────────────────────────────────────────┘

进阶方案（多团队/多业务线）:

┌─────────────────────────────────────────────────────┐
│  Namespace           用途                           │
│                                                      │
│  order-system-dev    ← 订单系统-开发                 │
│  order-system-prod   ← 订单系统-生产                 │
│  user-system-dev     ← 用户系统-开发                 │
│  user-system-prod    ← 用户系统-生产                 │
└─────────────────────────────────────────────────────┘
```

## 1.2 服务命名规范

```
服务名: {业务域}-{服务职责}-{可选标识}

✅ 正确:
  order-service          # 订单服务
  user-auth-service      # 用户认证服务
  payment-gateway        # 支付网关

❌ 错误:
  service1               # 无意义
  com.example.order      # 包含包名
  Order-Service          # 包含大写
  order_service          # 使用下划线
```

## 1.3 配置分层策略

```yaml
# bootstrap.yml 推荐配置结构
spring:
  application:
    name: order-service
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
        namespace: ${DEPLOY_NAMESPACE}
        file-extension: yaml
        
        # 第一层: 全局共享基础设施配置
        shared-configs[0]:
          data-id: global-infra.yaml
          group: INFRA_GROUP
          
        # 第二层: 全局共享框架配置
        shared-configs[1]:
          data-id: global-framework.yaml
          group: FRAMEWORK_GROUP
          
        # 第三层: 业务域共享配置
        shared-configs[2]:
          data-id: order-domain-common.yaml
          group: BUSINESS_GROUP
          
        # 第四层: 应用自身配置（自动加载，优先级最高）
        # ${prefix}-${profile}.${file-extension}
```

## 1.4 集群部署最佳配置

```
生产环境推荐拓扑:

                   Internet
                       │
                       ▼
              ┌─────────────────┐
              │    WAF / SLB    │
              └────────┬────────┘
                       │
              ┌────────▼────────┐
              │   Nginx / HAProxy│  (七层负载均衡)
              └────────┬────────┘
                       │
             ┌─────────┼─────────┐
             ▼         ▼         ▼
        ┌────────┐┌────────┐┌────────┐
        │Nacos-1 ││Nacos-2 ││Nacos-3 │  (Nacos 集群)
        └───┬────┘└───┬────┘└───┬────┘
            └─────────┼─────────┘
                      │
              ┌───────▼───────┐
              │  MySQL 主从   │  (共享存储)
              │  Master+Slave│
              └───────────────┘
```

# 2. 配置管理最佳实践

## 2.1 敏感配置处理

```yaml
# 生产环境中，敏感配置不直接写在 Nacos 中
# 方案一: 使用环境变量覆盖
spring:
  datasource:
    username: ${DB_USERNAME:default_user}
    password: ${DB_PASSWORD:}

# 方案二: 使用加密框架（见第17章）
spring:
  datasource:
    password: ENC(XyZ9aBcDeFgHiJkLmN...)

# 方案三: 使用 K8s Secret（K8s 环境）
# 通过环境变量注入 Pod，不在 Nacos 暴露
```

## 2.2 配置变更流程

```
配置变更安全流程:

1. 先在 Staging 环境验证
   └──→ 确保配置语法正确，功能正常

2. Beta 灰度发布（可选）
   └──→ 指定 1-2 台生产机器进行 Beta 验证

3. 全量发布
   └──→ 选定业务低峰期发布

4. 发布后监控
   └──→ 观察应用日志、监控指标

5. 异常回滚
   └──→ 通过历史版本一键回滚
```

## 2.3 配置清理策略

```sql
-- 定期清理历史配置（保留最近 30 天）
DELETE FROM his_config_info 
WHERE gmt_modified < DATE_SUB(NOW(), INTERVAL 30 DAY);

-- 清理无效配置（无关联的应用已下线）
-- 需要人工确认
SELECT data_id, group_id, tenant_id, app_name, gmt_modified
FROM config_info 
WHERE gmt_modified < DATE_SUB(NOW(), INTERVAL 90 DAY)
ORDER BY gmt_modified;
```

# 3. 性能调优

## 3.1 Nacos 服务端调优

```properties
# 服务端 application.properties 优化

# ===== 服务发现调优 =====
# 服务缓存刷新间隔（ms）
nacos.naming.service.cache.refresh.interval=1000
# 服务元数据过期时间（ms）
nacos.naming.meta.expire=60000
# 服务自动清理间隔（ms）
nacos.naming.clean.empty-service.interval=60000

# ===== 配置管理调优 =====
# 配置缓存大小
nacos.config.cache.size=10000
# 长轮询保留时间（ms）
nacos.config.longpoll.timeout=28000

# ===== 数据库连接池调优 =====
db.pool.config.connectionTimeout=30000
db.pool.config.maximumPoolSize=50
db.pool.config.minimumIdle=10
db.pool.config.idleTimeout=600000
```

## 3.2 客户端调优

```yaml
spring:
  cloud:
    nacos:
      discovery:
        # 服务列表刷新间隔（ms）
        watch-delay: 30000
        # 心跳间隔（ms），不建议修改
        heart-beat-interval: 5000
        # 心跳超时（ms）
        heart-beat-timeout: 15000
        # 权重
        weight: 10
      config:
        # 配置超时（ms）
        timeout: 5000
        # 长轮询超时（ms）
        config-long-poll-timeout: 30000
        # 配置重试次数
        config-retry-time: 3
```

# 4. 常见问题排错

## 4.1 服务注册失败

**现象**: 服务启动后，Nacos 控制台看不到服务实例

**排查步骤**:

```bash
# 1. 检查 Nacos 服务端是否正常
curl http://127.0.0.1:8848/nacos/v1/console/health/readiness

# 2. 检查客户端日志
grep -i "register" application.log
# 常见错误:
# - java.net.ConnectException: Connection refused
#   → Nacos 服务端未启动或端口被占用
# - 403 Forbidden
#   → 未配置鉴权或 token 过期
# - namespace not found
#   → 命名空间 ID 不存在

# 3. 检查网络连通性
telnet 127.0.0.1 8848

# 4. 检查 Nacos 服务端日志
tail -f /opt/nacos/logs/nacos-naming.log | grep ERROR
```

## 4.2 配置不刷新

**现象**: 在 Nacos 控制台修改了配置后，应用没有实时生效

**排查步骤**:

```bash
# 1. 确认 @RefreshScope 注解是否正确使用
# 检查: @Value 注入的配置是否在 @RefreshScope Bean 中

# 2. 确认配置 ID 是否匹配
# DataId = ${spring.application.name}-${spring.profiles.active}.${spring.cloud.nacos.config.file-extension}
# 示例: order-service-dev.yaml

# 3. 查看客户端日志
grep -i "refresh" application.log
grep -i "config change" application.log

# 4. 检查是否因为配置内容未变化导致不触发刷新
# Nacos 通过 MD5 判断配置是否变化

# 5. 手动触发刷新（通过 Actuator）
curl -X POST http://127.0.0.1:8080/actuator/refresh
```

## 4.3 客户端连接数过多

**现象**: Nacos 2.x 服务端报错 `Too many connections`

**原因**: gRPC 长连接模式下，每个客户端维持一个连接

**解决方案**:

```properties
# 服务端调大连接数限制
nacos.remote.server.grpc.sdk.max-inbound-message-size=10485760
nacos.remote.server.grpc.cluster.max-inbound-message-size=10485760

# 服务端增大线程池
nacos.remote.server.grpc.sdk.thread-pool.core-size=200
nacos.remote.server.grpc.sdk.thread-pool.max-size=500
```

```bash
# 调整系统文件描述符限制
ulimit -n 655350
```

## 4.4 心跳超时大量实例被剔除

**现象**: Nacos 控制台大量实例被标记为不健康或被剔除

**排查**:

```bash
# 1. 检查网络是否有波动
# 查看心跳超时日志
grep "expired" /opt/nacos/logs/nacos-naming.log

# 2. 检查是否因 GC 导致心跳暂停
# 查看 GC 日志
grep "Full GC" /opt/nacos/logs/gc.log

# 3. 临时提高保护阈值
# Nacos 控制台 → 服务详情 → 编辑服务 → 保护阈值设置为 0.5
```

## 4.5 集群脑裂

**现象**: 集群中有多个 Leader 或无法达成共识

**解决方案**:

```bash
# 1. 检查网络连通性
ping {other-node-ip}

# 2. 检查 cluster.conf 配置
cat /opt/nacos/conf/cluster.conf
# 确保所有节点的 cluster.conf 一致

# 3. 检查端口
# 8848: HTTP
# 7848: gRPC Client
# 9848: gRPC Server
# 9849: Raft
netstat -tlnp | grep -E "8848|7848|9848|9849"

# 4. 查看集群日志
grep -i "leader\|election\|raft" /opt/nacos/logs/nacos-cluster.log

# 5. 如果脑裂严重，重建集群
# - 停止所有节点
# - 清空 MySQL 中的 raft 相关表
# - 重新启动节点
```

## 4.6 MySQL 连接池耗尽

**现象**: Nacos 日志报错 `Failed to obtain JDBC Connection`

**排查**:

```bash
# 1. 检查 MySQL 连接数
mysql> SHOW PROCESSLIST;
mysql> SHOW STATUS LIKE 'Threads_connected';

# 2. 增大连接池
# application.properties
db.pool.config.maximumPoolSize=100

# 3. 增大 MySQL 最大连接数
# my.cnf
max_connections = 1000

# 4. 检查是否有慢查询
mysql> SHOW FULL PROCESSLIST;
mysql> SELECT * FROM mysql.slow_log ORDER BY start_time DESC LIMIT 10;
```

# 5. 运维操作 Checklist

## 5.1 日常维护任务

| 频率 | 任务 | 命令/操作 |
|------|------|----------|
| 每日 | 检查 Nacos 节点状态 | `curl /nacos/v1/core/cluster/nodes` |
| 每日 | 检查日志是否有异常 | `grep ERROR /opt/nacos/logs/nacos.log` |
| 每周 | 检查磁盘使用率 | `df -h /opt/nacos` |
| 每周 | 备份 MySQL 数据 | `mysqldump ...` |
| 每月 | 清理历史配置数据 | SQL 清理 |
| 每月 | 检查并升级 SDK 版本 | 查看 GitHub Release |
| 每季 | 全链路压测 | 验证 Nacos 性能 |
| 每季 | 更新 Token 密钥 | 滚动更新密钥 |

## 5.2 升级操作步骤

```
Nacos 版本升级 SOP:

1. 备份
   ├── 备份 MySQL 数据库
   ├── 备份 conf/ 目录
   └── 备份集群配置

2. 灰度升级（先升级一个节点）
   ├── 下载新版本，保留原 conf/
   ├── 停止该节点服务
   ├── 启动新版本
   └── 验证功能（注册、发现、配置、控制台）

3. 验证通过后，滚动升级其余节点
   ├── 逐一停止 → 升级 → 启动
   └── 每次间隔 5 分钟（等待数据同步）

4. 全量验证
   ├── 集群状态
   ├── 服务注册/发现
   └── 配置推送

5. 回滚方案（如有问题）
   └── 恢复原版本文件 → 重启
```

# 6. 生产环境配置模板

## 6.1 application.properties 生产模板

```properties
# ==================== 基础配置 ====================
server.port=8848
server.servlet.contextPath=/nacos

# ==================== 数据库配置 ====================
spring.datasource.platform=mysql
db.num=1
db.url.0=jdbc:mysql://mysql-master:3306/nacos?characterEncoding=utf8&connectTimeout=5000&socketTimeout=10000&autoReconnect=true&useSSL=true&requireSSL=true
db.user.0=nacos_user
db.password.0=${MYSQL_PASSWORD}

# ==================== 鉴权配置 ====================
nacos.core.auth.enabled=true
nacos.core.auth.system.type=nacos
nacos.core.auth.plugin.nacos.token.secret.key=${TOKEN_SECRET_KEY}
nacos.core.auth.plugin.nacos.token.expire.seconds=18000
nacos.core.auth.server.identity.key=${SERVER_IDENTITY_KEY}
nacos.core.auth.server.identity.value=${SERVER_IDENTITY_VALUE}
nacos.core.auth.enable.userAgentAuthWhite=false

# ==================== 集群配置 ====================
nacos.inetutils.ip-address=${POD_IP}
nacos.inetutils.prefer-hostname-over-ip=false

# ==================== 性能配置 ====================
db.pool.config.connectionTimeout=30000
db.pool.config.maximumPoolSize=50
db.pool.config.minimumIdle=10
db.pool.config.idleTimeout=600000
db.pool.config.maxLifetime=1800000

# ==================== 日志配置 ====================
nacos.naming.log.level=INFO
nacos.config.log.level=INFO
logging.file.path=/data/nacos/logs
logging.level.com.alibaba.nacos=INFO

# ==================== Prometheus ====================
management.endpoints.web.exposure.include=*
management.metrics.export.prometheus.enabled=true
```

## 6.2 Spring Cloud 应用 bootstrap.yml 模板

```yaml
spring:
  application:
    name: ${APPLICATION_NAME}
  profiles:
    active: ${PROFILE:dev}
  cloud:
    nacos:
      discovery:
        server-addr: ${NACOS_SERVER_ADDR}
        namespace: ${NACOS_NAMESPACE}
        group: DEFAULT_GROUP
        cluster-name: ${CLUSTER_NAME:DEFAULT}
        weight: ${INSTANCE_WEIGHT:10}
        metadata:
          version: ${APP_VERSION:1.0.0}
          region: ${REGION:default}
        ip: ${POD_IP:}
      config:
        server-addr: ${NACOS_SERVER_ADDR}
        namespace: ${NACOS_NAMESPACE}
        group: DEFAULT_GROUP
        file-extension: yaml
        timeout: 5000
        shared-configs:
          - data-id: global-infra.yaml
            group: INFRA_GROUP
            refresh: true
          - data-id: global-framework.yaml
            group: FRAMEWORK_GROUP
            refresh: true

# Actuator
management:
  endpoints:
    web:
      exposure:
        include: health,info,refresh
  endpoint:
    health:
      show-details: always
```

# 7. 应急处理

## 7.1 全集群宕机恢复

```bash
# 1. 确认 MySQL 正常
mysql -h mysql_host -u root -p -e "SELECT 1"

# 2. 启动第一个 Nacos 节点
cd /opt/nacos/bin && sh startup.sh

# 3. 等待该节点完全启动（Leader 选举完成）
sleep 30
curl http://127.0.0.1:8848/nacos/v1/core/cluster/self

# 4. 启动其余节点
# (在其他节点上执行)
cd /opt/nacos/bin && sh startup.sh

# 5. 验证集群状态
curl http://127.0.0.1:8848/nacos/v1/core/cluster/nodes

# 6. 客户端自动重连后恢复正常
# (客户端有本地缓存，应用不会中断)
```

## 7.2 配置误删除恢复

```bash
# 1. 从历史版本恢复
# Nacos 控制台 → 配置管理 → 历史版本 → 点击回滚

# 2. 从 MySQL 直接恢复
mysql> SELECT content FROM his_config_info 
    -> WHERE data_id = 'order-service-dev.yaml' 
    -> AND group_id = 'DEFAULT_GROUP'
    -> ORDER BY gmt_create DESC LIMIT 1;

# 3. 从备份恢复
# 使用上一节的数据备份脚本恢复

# 4. 从本地快照恢复（客户端侧）
# 文件路径: ${user.home}/nacos/config/fixed-{server}_nacos/snapshot/
```
