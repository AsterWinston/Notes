
# 1. 三层配置模型全景

```
┌─────────────────────────────────────────────────────────────┐
│                       Nacos 配置模型                         │
│                                                              │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐     │
│  │  Namespace-1 │   │  Namespace-2 │   │  Namespace-3 │     │
│  │  (Production)│   │  (Staging)   │   │  (Dev)       │     │
│  │              │   │              │   │              │     │
│  │ ┌─────────┐  │   │ ┌─────────┐  │   │ ┌─────────┐  │     │
│  │ │ Group-A │  │   │ │ Group-A │  │   │ │ Group-A │  │     │
│  │ │  ├─ D1  │  │   │ │  ├─ D1  │  │   │ │  ├─ D1  │  │     │
│  │ │  └─ D2  │  │   │ │  └─ D2  │  │   │ │  └─ D2  │  │     │
│  │ └─────────┘  │   │ └─────────┘  │   │ └─────────┘  │     │
│  │ ┌─────────┐  │   │ ┌─────────┐  │   │ ┌─────────┐  │     │
│  │ │ Group-B │  │   │ │ Group-B │  │   │ │ Group-B │  │     │
│  │ │  ├─ D3  │  │   │ │  ├─ D3  │  │   │ │  ├─ D3  │  │     │
│  │ │  └─ D4  │  │   │ │  └─ D4  │  │   │ │  └─ D4  │  │     │
│  │ └─────────┘  │   │ └─────────┘  │   │ └─────────┘  │     │
│  └──────────────┘   └──────────────┘   └──────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

# 2. Namespace（命名空间）深度解析

## 2.1 命名空间的核心价值

Namespace 是 Nacos 中最顶层的隔离单位，不同 Namespace 之间的服务和配置**完全物理隔离**。

| 场景 | Namespace 用法 | 示例 |
|------|---------------|------|
| 环境隔离 | 每个环境一个 Namespace | `dev` / `test` / `staging` / `prod` |
| 业务线隔离 | 不同业务线独立管理 | `order-system` / `user-system` / `payment-system` |
| 租户隔离 | SaaS 多租户场景 | `tenant-a` / `tenant-b` / `tenant-c` |
| 团队隔离 | 不同团队独立配置 | `team-alpha` / `team-beta` |

## 2.2 Namespace 生命周期管理

```java
// NamespaceManager.java
@Component
public class NamespaceManager {
    
    // 创建命名空间
    public void createNamespace(String namespaceId, String name, String description) {
        TenantInfo tenant = new TenantInfo();
        tenant.setTenantId(namespaceId);
        tenant.setTenantName(name);
        tenant.setTenantDesc(description);
        
        persistService.insertTenantInfo(tenant);
    }
    
    // 删除命名空间（级联删除所有配置和服务）
    public void deleteNamespace(String namespaceId) {
        // 1. 删除所有配置
        persistService.removeConfigInfoByTenant(namespaceId);
        
        // 2. 删除所有服务
        serviceManager.removeAllServices(namespaceId);
        
        // 3. 删除命名空间记录
        persistService.removeTenantInfo(namespaceId);
    }
    
    // 获取命名空间列表
    public List<TenantInfo> getAllNamespaces() {
        return persistService.findAllTenantInfo();
    }
}
```

## 2.3 命名空间 ID 生成策略

```
public namespace (系统默认):
  - namespaceId: ""
  - namespaceName: "public"
  - 不显示 namespaceId

自定义 namespace:
  - namespaceId: UUID 自动生成
  - namespaceName: 用户指定
  - console UI 显示 namespaceId
```

# 3. Group（分组）深度解析

## 3.1 Group 分类策略

```yaml
# 按功能维度分组
DEFAULT_GROUP:       # 默认分组，应用主配置
DATABASE_GROUP:      # 数据库相关配置
MIDDLEWARE_GROUP:    # 中间件配置（Redis、MQ、ES）
BUSINESS_GROUP:      # 业务配置（开关、阈值、策略）
SECURITY_GROUP:      # 安全相关配置（密钥、Token）
LOGGING_GROUP:       # 日志配置
MONITORING_GROUP:    # 监控告警配置
```

## 3.2 Group 最佳实践

```yaml
# 微服务架构下的 Group 划分示例
spring:
  cloud:
    nacos:
      config:
        extension-configs:
          # 基础设施配置（最高优先级共享）
          - data-id: infra-config.yaml
            group: INFRA_GROUP
            refresh: true
          # 日志配置
          - data-id: logback-config.yaml
            group: LOGGING_GROUP
            refresh: true
          # 业务开关配置
          - data-id: feature-flags.yaml
            group: BUSINESS_GROUP
            refresh: true
        shared-configs:
          # 全局共享配置
          - data-id: common-config.yaml
            group: DEFAULT_GROUP
            refresh: true
```

# 4. Data ID 详解

## 4.1 Data ID 命名规范

```
Spring Cloud Alibaba 自动生成的 Data ID:

${prefix}${spring.profiles.active}.${file-extension}

示例:
  order-service.yaml          → prefix=order-service
  order-service-dev.yaml      → profile=dev
  order-service-prod.yml       → file-extension=yml
  user-service-test.properties → file-extension=properties
```

## 4.2 Data ID 配置优先级（源码）

```java
// NacosPropertySourceLocator.java
public PropertySource<?> locate(Environment env) {
    CompositePropertySource composite = new CompositePropertySource("NACOS");
    
    // 加载顺序（后面覆盖前面 = 优先级递增）
    // 1. shared-configs（多个按顺序）
    loadSharedConfiguration(composite, env);
    
    // 2. extension-configs（多个按顺序）
    loadExtConfiguration(composite, env);
    
    // 3. 应用自身配置（最高优先级）
    // 3a. 不带 profile 的配置
    loadApplicationConfiguration(composite, dataIdPrefix, env);
    // 3b. 带 profile 的配置
    loadApplicationConfiguration(composite, dataIdWithProfile, env);
    
    return composite;
}
```

## 4.3 多 Profile 配置加载

```yaml
# bootstrap.yml
spring:
  application:
    name: order-service
  profiles:
    active: dev,china   # 激活多个 profile
  cloud:
    nacos:
      config:
        file-extension: yaml
```

当 `spring.profiles.active=dev,china` 时，会尝试加载：

```
1. order-service.yaml            (无 profile)
2. order-service-dev.yaml        (dev profile)
3. order-service-china.yaml      (china profile)
4. order-service-dev.yaml + china  (交叉 profile，2.x+ 支持)
```

# 5. 配置内容格式

## 5.1 支持的配置格式

| 格式类型 | file-extension | 说明 |
|---------|---------------|------|
| Properties | properties | Key=Value 格式 |
| YAML | yaml / yml | 层级结构 |
| JSON | json | JSON 格式 |
| XML | xml | XML 格式 |
| HTML | html | 富文本 |
| TEXT | 无后缀 | 纯文本 |

## 5.2 YAML 配置最佳实践

```yaml
# Nacos 中存储的配置示例
server:
  port: 8080
  servlet:
    context-path: /order

spring:
  datasource:
    url: jdbc:mysql://localhost:3306/order_db
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 3000

  redis:
    host: ${REDIS_HOST:localhost}
    port: 6379
    password: ${REDIS_PASSWORD:}
    lettuce:
      pool:
        max-active: 50
        max-idle: 20

# 自定义业务配置
order:
  timeout: 30000
  retry:
    max-attempts: 3
    backoff: 1000
  features:
    new-payment-flow: false
    gray-release: true
```

# 6. 配置变更推送模型

## 6.1 推送触发机制

```
Nacos 服务端配置变更触发推送的几种方式:

1. 主动推送（gRPC 双向流）
   ConfigController.publish()
   → ConfigChangePublisher.publish()
   → 遍历所有连接的 gRPC Client
   → 向订阅了该配置的 Client 推送 NotifyRequest

2. 长轮询（HTTP Long Polling）
   Client: POST /v1/cs/configs/listener (30s timeout)
   Server: 挂起请求，有变更时立即返回
   Server: 无变更时 29.5s 后返回空

3. 客户端轮询保底
   定时任务每 3s 检查一次本地 MD5 与远端 MD5
```

## 6.2 推送时机

| 操作 | 是否推送 | 说明 |
|------|---------|------|
| 新建配置 | ✅ 是 | 推送完整内容 |
| 修改配置 | ✅ 是 | 推送变更通知，客户端重新拉取 |
| 删除配置 | ✅ 是 | 推送空内容 |
| 批量导入 | ✅ 是 | 逐个推送 |
| 灰度发布 | ✅ 是（灰度 IP） | 仅向灰度 IP 推送 |

# 7. 配置生命周期管理

## 7.1 配置状态机

```
     ┌─────────┐
     │  CREATED │  (新建)
     └────┬─────┘
          │
          ▼
     ┌─────────┐
     │ PUBLISHED│  (已发布/已更新)
     └────┬─────┘
          │
     ┌────┴────┐
     ▼         ▼
┌────────┐ ┌─────────┐
│ BETA   │ │ DELETED │  (灰度 / 已删除)
└────┬───┘ └─────────┘
     │
     ▼
┌────────┐
│PUBLISHED│  (灰度转正)
└────────┘
```

## 7.2 配置版本追踪

Nacos 在数据库中维护完整的配置变更历史：

```sql
-- 查询配置历史
SELECT nid, data_id, group_id, content, md5, op_type, gmt_create
FROM his_config_info
WHERE data_id = 'order-service-dev.yaml'
  AND group_id = 'DEFAULT_GROUP'
ORDER BY gmt_create DESC;
```

# 8. 高级配置特性

## 8.1 配置标签（Tag）

```bash
# 给配置打标签
curl -X POST 'http://127.0.0.1:8848/nacos/v1/cs/configs' \
  -d 'dataId=order-service-dev.yaml&group=DEFAULT_GROUP&content=...&config_tags=order,sensitive'

# 按标签搜索配置
curl -X GET 'http://127.0.0.1:8848/nacos/v1/cs/configs?search=accurate&config_tags=order'
```

## 8.2 配置的 AppName 归属

```bash
# 指定配置归属于某个应用
curl -X POST 'http://127.0.0.1:8848/nacos/v1/cs/configs' \
  -d 'dataId=order-service-dev.yaml&group=DEFAULT_GROUP&content=...&appName=order-system'
```

## 8.3 配置描述与备注

```bash
curl -X POST 'http://127.0.0.1:8848/nacos/v1/cs/configs' \
  -d 'dataId=order-service-dev.yaml&group=DEFAULT_GROUP&content=...&desc=订单服务开发环境配置&type=yaml'
```

# 9. 配置同步与一致性

## 9.1 集群配置同步流程

```
Node-1 (收到配置写入)          Node-2               Node-3
    │                             │                    │
    ├── 写入 MySQL                │                    │
    ├── 写入本地内存 Cache        │                    │
    ├── 写入本地 dump 文件        │                    │
    │                             │                    │
    ├── Raft 日志复制 ────────────>│ 收到日志           │
    │                             ├── 写入本地 MySQL   │
    │                             ├── 写入本地内存     │
    │                             ├── 写入本地 dump    │
    │                             │                    │
    ├── Raft 日志复制 ────────────┼───────────────────>│
    │                             │                    ├── 同上
    │                             │                    │
    ├── 多数派 ACK 后提交          │                    │
    │                             │                    │
    ├── 发布 LocalDataChangeEvent │                    │
    └── 通知所有长轮询客户端       │                    │
```

## 9.2 配置同步数据流

```
    ┌──────────────────┐
    │    Client 请求    │
    └────────┬─────────┘
             │
    ┌────────▼─────────┐
    │   ConfigController│
    │   (REST API)      │
    └────────┬─────────┘
             │
    ┌────────▼─────────┐
    │  ConfigServletInner │
    └────────┬─────────┘
             │
    ┌────────┼──────────────────┐
    ▼        ▼                   ▼
┌──────┐ ┌──────┐         ┌──────────┐
│MySQL │ │Disk  │         │  Raft    │
│DB    │ │Dump  │         │Consistency│
└──────┘ └──────┘         └────┬─────┘
                               │
                        ┌──────▼──────┐
                        │ 其他 Nacos   │
                        │ 节点         │
                        └─────────────┘
```

# 10. 配置故障场景与恢复

| 故障场景 | 服务端表现 | 客户端表现 | 恢复方案 |
|---------|-----------|-----------|---------|
| MySQL 宕机 | CP 写入失败 | 不影响（使用缓存） | 恢复 MySQL，Raft 日志回放 |
| 单节点宕机 | 不影响（集群） | 自动切换 | 重启节点自动恢复 |
| 全集群宕机 | 完全不可用 | 使用本地快照 | 启动集群，从 MySQL + dump 恢复 |
| 网络分区 | 少数派不可用 | 连接多数派的正常 | 网络恢复后 Raft 自动同步 |
| 配置误删除 | N/A | N/A | 从历史版本恢复 |
