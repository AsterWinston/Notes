# 1. Hazelcast 简介

**Hazelcast** 是一个开源的内存数据网格（IMDG），提供了分布式数据结构（如 Map、Queue、Topic 等），天然适合做分布式 Session 存储。

| 特性 | 说明 |
|------|------|
| **嵌入式运行** | 无需独立部署，随应用进程启动 |
| **去中心化** | 无主节点，自动发现与故障转移 |
| **分布式 Map** | 数据自动分片存储，节点间数据均衡 |
| **TTL 支持** | 原生支持 Key 级别 TTL |
| **事件机制** | 原生分布式 Topic，无需额外 Pub/Sub |
| **CP 子系统** | 支持强一致性场景 |

## 1.1 依赖引入

```xml
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-hazelcast</artifactId>
</dependency>
<dependency>
    <groupId>com.hazelcast</groupId>
    <artifactId>hazelcast-spring</artifactId>
</dependency>
```

---

# 2. 架构与数据模型

## 2.1 整体架构

```
应用实例 A (Hazelcast 节点)      应用实例 B (Hazelcast 节点)
┌──────────────────────┐      ┌──────────────────────┐
│  App A               │      │  App B               │
│  ┌────────────────┐  │      │  ┌────────────────┐  │
│  │ Hazelcast Node │◄─┼──────┼─►│ Hazelcast Node │  │
│  │ (存储 Session)  │  │      │  │ (存储 Session)  │  │
│  └────────────────┘  │      │  └────────────────┘  │
└──────────────────────┘      └──────────────────────┘
         │                            │
         └───────── 集群网络 ─────────┘
          (TCP/IP 或 组播发现)
```

## 2.2 数据存储

Session 数据存储在 Hazelcast 的 **分布式 IMap** 中：

```
IMap<String, MapSession>
  ├── Key: sessionId (自动分片)
  ├── Value: MapSession (序列化存储)
  └── TTL: maxInactiveInterval 秒

每个节点持有部分数据（分片），数据在节点间自动迁移和备份。
```

---

# 3. 配置详解

## 3.1 application.yml

```yaml
spring:
  session:
    timeout: 30m
    hazelcast:
      map-name: spring:session:sessions  # IMap 名称
      flush-mode: on_save
      save-mode: on_set_attribute

  hazelcast:
    config: classpath:hazelcast.xml       # 可选：外部配置文件
```

## 3.2 Hazelcast 网络配置

```yaml
# application.yml
hazelcast:
  network:
    join:
      multicast:
        enabled: false                    # 禁用组播（云环境通常不支持）
      tcp-ip:
        enabled: true
        members:
          - 192.168.1.101
          - 192.168.1.102
          - 192.168.1.103
      kubernetes:
        enabled: false                    # K8s 环境可启用
        service-name: my-service
        namespace: default
```

## 3.3 Java 配置

```java
package com.example.config;

import com.hazelcast.config.Config;
import com.hazelcast.config.MapConfig;
import com.hazelcast.core.Hazelcast;
import com.hazelcast.core.HazelcastInstance;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.session.hazelcast.config.annotation.web.http.EnableHazelcastHttpSession;
import org.springframework.session.hazelcast.HazelcastIndexedSessionRepository;

@Configuration
@EnableHazelcastHttpSession(
    maxInactiveIntervalInSeconds = 1800,
    sessionMapName = "spring:session:sessions"
)
public class HazelcastSessionConfig {

    @Bean
    public Config hazelcastConfig() {
        Config config = new Config();

        // 配置 Session Map
        MapConfig sessionMapConfig = new MapConfig("spring:session:sessions");
        sessionMapConfig.setTimeToLiveSeconds(1800);  // 30 分钟 TTL
        sessionMapConfig.setBackupCount(1);            // 每份数据 1 个备份
        config.addMapConfig(sessionMapConfig);

        // 集群发现
        config.getNetworkConfig().getJoin()
            .getMulticastConfig().setEnabled(false);
        config.getNetworkConfig().getJoin()
            .getTcpIpConfig()
            .setEnabled(true)
            .addMember("192.168.1.101")
            .addMember("192.168.1.102");

        return config;
    }

    @Bean
    public HazelcastInstance hazelcastInstance() {
        return Hazelcast.newHazelcastInstance(hazelcastConfig());
    }
}
```

---

# 4. HazelcastIndexedSessionRepository

## 4.1 核心实现

```java
public class HazelcastIndexedSessionRepository extends
        AbstractHazelcastIndexedSessionRepository<
                HazelcastIndexedSessionRepository.HazelcastSession> {

    @Override
    public HazelcastSession createSession() {
        HazelcastSession session = new HazelcastSession();
        session.setMaxInactiveInterval(defaultMaxInactiveInterval);
        return session;
    }

    @Override
    public void save(HazelcastSession session) {
        // 存入 IMap
        sessions.put(session.getId(), new MapSession(session),
            session.getMaxInactiveInterval().getSeconds(), TimeUnit.SECONDS);
    }

    @Override
    public HazelcastSession findById(String id) {
        MapSession saved = sessions.get(id);
        if (saved == null) return null;
        if (saved.isExpired()) {
            deleteById(id);
            return null;
        }
        return new HazelcastSession(saved);
    }

    @Override
    public void deleteById(String id) {
        sessions.remove(id);
    }
}
```

---

# 5. 集群与高可用

## 5.1 数据备份

```java
MapConfig sessionMapConfig = new MapConfig("spring:session:sessions");
sessionMapConfig.setBackupCount(1);   // 每个数据备份 1 份（共 2 份）
sessionMapConfig.setAsyncBackupCount(0); // 同步备份（保证一致性）
```

| 配置 | 说明 |
|------|------|
| `backupCount` | 同步备份数量（默认 1） |
| `asyncBackupCount` | 异步备份数量（默认 0） |

## 5.2 近缓存（Near Cache）

为了减少网络开销，Hazelcast 支持近缓存：

```java
MapConfig sessionMapConfig = new MapConfig("spring:session:sessions");

NearCacheConfig nearCacheConfig = new NearCacheConfig();
nearCacheConfig.setTimeToLiveSeconds(60);    // 缓存 60 秒
nearCacheConfig.setMaxSize(10000);            // 最多 1 万个条目
nearCacheConfig.setInvalidateOnChange(true);  // 变更时失效

sessionMapConfig.setNearCacheConfig(nearCacheConfig);
```

**慎用近缓存**：Session 数据需要最新状态，近缓存可能导致读到旧数据。通常不推荐对 Session 启用近缓存。

---

# 6. 序列化

## 6.1 默认序列化

Hazelcast 默认使用 **Java 序列化**。可以通过配置改为其他序列化方式：

```java
config.getSerializationConfig()
    .getSerializerConfigs()
    .add(new SerializerConfig()
        .setTypeClass(MapSession.class)
        .setClass(JacksonSerializer.class));
```

## 6.2 使用 Kryo 序列化

```java
config.getSerializationConfig()
    .getSerializerConfigs()
    .add(new SerializerConfig()
        .setTypeClassName("org.springframework.session.MapSession")
        .setClassName("com.hazelcast.nio.serialization.kryo.KryoSerializer"));
```

---

# 7. 适用场景与限制

## 7.1 适用场景

| 场景 | 说明 |
|------|------|
| **不想引入外部存储** | Hazelcast 嵌入式部署，不依赖 Redis/DB |
| **对内存数据一致性要求高** | Hazelcast 的 CP 子系统支持强一致性 |
| **应用集群相对稳定** | 节点数变化不频繁（节点加入/离开有数据迁移成本） |
| **中小规模集群** | 10-50 个节点规模最佳 |

## 7.2 限制

| 限制 | 说明 |
|------|------|
| **数据在内存** | 需要充足的堆外/堆内存 |
| **节点退出** | 节点退出时需要 repartition 数据，有短暂性能影响 |
| **版本兼容** | 不同 Hazelcast 版本不兼容，升级需同步所有节点 |
| **调试困难** | 嵌入式运行，日志分散在应用日志中 |

## 7.3 Hazelcast vs Redis vs JDBC

| 维度 | Hazelcast | Redis | JDBC |
|------|-----------|-------|------|
| **部署方式** | 嵌入式 | 独立服务 | 独立服务 |
| **运维复杂度** | 低（自动） | 中（Redis 集群运维） | 低（共用 DB） |
| **性能** | 高（内存 + 近缓存） | 高（内存） | 中（磁盘 I/O） |
| **扩展性** | 中（节点数有上限） | 高（集群分片） | 中（读写分离） |
| **一致性** | 中/强（可配置） | 弱（最终一致） | 强（ACID） |
| **学习成本** | 中 | 低 | 低 |
