
# 1. 架构设计最佳实践

## 1.1 ZNode 设计原则

```text
ZNode 设计原则：
+----------------------------------------------------------------------+
|                                                                       |
|  1. 宁可多层级，不要单节点大 JSON                                      |
|     /config/db/url 优于 /config {db:{url:"..."}, redis:{...}}         |
|     每次修改只需要 watch 一个节点，粒度更细                            |
|                                                                       |
|  2. 保持 ZNode 数据精简（< 100KB）                                      |
|     超过 100KB 考虑拆分或外部存储                                      |
|                                                                       |
|  3. 使用临时节点做心跳/注册                                            |
|     服务实例、锁竞争者使用 EPHEMERAL                                   |
|                                                                       |
|  4. 使用顺序节点做队列/锁                                              |
|     利用 ZK 自动编号特性，避免 ID 冲突                                 |
|                                                                       |
|  5. 层次不要超过 10 层                                                 |
|     /a/b/c/d/e/f/g/h/i/j/k → 太深                                    |
+----------------------------------------------------------------------+
```

## 1.2 集群架构

```text
集群设计原则：
+----------------------------------------------------------------------+
|                                                                       |
|  1. 生产环境至少 3 节点（推荐 5）                                      |
|                                                                       |
|  2. 使用 Observer 做读扩展                                              |
|     不影响写性能，线性扩展读吞吐                                        |
|                                                                       |
|  3. 跨区域部署用 Observer，不要用 Follower                             |
|     避免跨区域投票影响写延迟                                            |
|                                                                       |
|  4. 数据和日志分盘                                                     |
|     dataDir 和 dataLogDir 放在不同物理磁盘                              |
|     dataLogDir 使用 SSD                                                |
|                                                                       |
|  5. 节点分布在不同物理机/机架                                          |
|     避免单点故障                                                        |
+----------------------------------------------------------------------+
```

# 2. 客户端使用最佳实践

## 2.1 Curator 优于原生 API

```java
// 始终使用 Curator，避免原生 API 的坑

// ✗ 原生 API（不推荐）
ZooKeeper zk = new ZooKeeper("host:2181", 3000, watcher);
byte[] data = zk.getData("/path", true, stat); // 一次性 Watch！

// ✓ Curator（推荐）
CuratorFramework client = CuratorFrameworkFactory.builder()
    .connectString("host1:2181,host2:2181,host3:2181")
    .sessionTimeoutMs(30000)
    .retryPolicy(new ExponentialBackoffRetry(1000, 3))
    .namespace("myapp")
    .build();
client.start();

// 使用 NodeCache 自动管理 Watch
NodeCache nodeCache = new NodeCache(client, "/path");
nodeCache.start();
nodeCache.getListenable().addListener(() -> {
    ChildData data = nodeCache.getCurrentData();
    // 自动重新注册 Watch
});
```

## 2.2 正确使用 namespace

```java
// 多个应用共享同一个 ZK 集群时，使用 namespace 隔离

// 应用 A
CuratorFramework clientA = CuratorFrameworkFactory.builder()
    .namespace("app-a")
    .build();
// clientA.create().forPath("/config") → 实际路径 /app-a/config

// 应用 B
CuratorFramework clientB = CuratorFrameworkFactory.builder()
    .namespace("app-b")
    .build();
// clientB.create().forPath("/config") → 实际路径 /app-b/config
```

## 2.3 连接状态处理

```java
// 监听连接状态并正确处理
client.getConnectionStateListenable().addListener(
    (client, newState) -> {
    switch (newState) {
        case RECONNECTED:
            // 连接恢复，检查并重建状态
            recoverState();
            break;
        case LOST:
            // Session 过期，临时节点和 Watch 已丢失
            // 需要重新创建所有资源
            reinitialize();
            break;
        case SUSPENDED:
            // 连接暂时挂起，等待恢复
            pauseOperations();
            break;
    }
});
```

## 2.4 资源清理

```java
// 使用 try-with-resources 确保资源清理
try (CuratorFramework client = CuratorFrameworkFactory.newClient(
    "host:2181", 30000, 10000,
    new ExponentialBackoffRetry(1000, 3))) {
    client.start();

    try (NodeCache cache = new NodeCache(client, "/config")) {
        cache.start();
        // 使用 cache...
    } // cache 自动 close

} // client 自动 close
```

# 3. Watch 使用最佳实践

```text
Watch 最佳实践：
+----------------------------------------------------------------------+
|                                                                       |
|  1. 永远不要手动处理一次性 Watch 重注册                               |
|     使用 Curator Cache（NodeCache/PathChildrenCache/TreeCache）      |
|                                                                       |
|  2. 避免 Watch 风暴                                                    |
|     不要把大量客户端对同一个 ZNode 注册 Watch                         |
|     解决方案：打散到多个子节点、客户端随机延迟                         |
|                                                                       |
|  3. Watch 回调中不要执行耗时操作                                       |
|     Watch 回调在 EventThread 单线程中执行                             |
|     耗时操作提交到线程池异步处理                                      |
|                                                                       |
|  4. 及时清理不再需要的 Watch                                          |
|     避免 Watch 泄漏导致内存增长                                       |
|                                                                       |
+----------------------------------------------------------------------+
```

```java
// Watch 回调的正确处理方式
private final ExecutorService watchCallbackPool =
    Executors.newFixedThreadPool(4);

nodeCache.getListenable().addListener(() -> {
    // 在 EventThread 中只做必要的数据提取
    ChildData data = nodeCache.getCurrentData();
    String path = data.getPath();
    byte[] content = data.getData();

    // 耗时操作提交到线程池
    watchCallbackPool.submit(() -> {
        processConfigChange(path, content);
    });
});
```

# 4. 分布式锁最佳实践

```text
分布式锁最佳实践：
+----------------------------------------------------------------------+
|                                                                       |
|  1. 锁粒度尽可能细                                                     |
|     /locks/order/{orderId} 优于 /locks/order                          |
|                                                                       |
|  2. 设置合理的超时时间                                                  |
|     避免死锁导致线程永久阻塞                                           |
|                                                                       |
|  3. 锁持有时间尽量短                                                    |
|     临界区内只做必要的互斥操作                                         |
|                                                                       |
|  4. 使用可重入锁（InterProcessMutex）                                 |
|     避免递归调用时自己阻塞自己                                         |
|                                                                       |
|  5. 不要在锁内开启新的事务或远程调用                                    |
|     这会显著延长锁持有时间                                             |
|                                                                       |
|  6. 锁路径命名规范                                                      |
|     /locks/{resource-type}/{resource-id}                              |
|                                                                       |
+----------------------------------------------------------------------+
```

# 5. 服务发现最佳实践

```yaml
# 服务发现配置最佳实践
spring:
  cloud:
    zookeeper:
      discovery:
        instance-id: ${spring.application.name}:${server.port}:${random.uuid}
        metadata:
          version: ${info.version}
          region: ${REGION:default}
          weight: ${WEIGHT:100}
          management.port: ${management.server.port}
```

```text
服务发现最佳实践：
+----------------------------------------------------------------------+
|                                                                       |
|  1. 客户端缓存实例列表                                                 |
|     不要每次调用都去 ZK 拉取，使用本地缓存 + Watch 更新               |
|                                                                       |
|  2. 添加版本信息到 metadata                                           |
|     便于灰度发布和问题排查                                             |
|                                                                       |
|  3. 设置合理的 Session 超时                                            |
|     15-30s 为生产常用值                                                |
|                                                                       |
|  4. 消费端做好降级                                                    |
|     ZK 不可用时使用本地缓存的服务列表                                  |
|                                                                       |
+----------------------------------------------------------------------+
```

# 6. 配置管理最佳实践

```text
配置管理最佳实践：
+----------------------------------------------------------------------+
|                                                                       |
|  1. 分层配置                                                            |
|     公共层 / 环境层 / 应用层 / 实例层                                   |
|     优先级由低到高                                                     |
|                                                                       |
|  2. 敏感信息不存明文                                                    |
|     使用加密存储或引用 KMS                                             |
|                                                                       |
|  3. 做配置变更审计                                                      |
|     记录谁在什么时间改了什么                                           |
|                                                                       |
|  4. 配置变更后逐步推送                                                  |
|     配置变更 -> Watch 通知 -> @RefreshScope 刷新                       |
|                                                                       |
+----------------------------------------------------------------------+
```

# 7. 反模式（Anti-Patterns）

| 反模式 | 问题 | 正确做法 |
|--------|------|---------|
| **ZK 当数据库用** | ZNode 数据量过大，性能下降 | 只存元数据和配置，大数据用 DB/缓存 |
| **ZK 当消息队列用** | 频繁创建删除顺序节点导致性能瓶颈 | 使用 Kafka/RocketMQ 等专业消息队列 |
| **全局大锁** | 所有操作竞争一把锁，吞吐量极低 | 按业务 ID 细粒度加锁 |
| **手动管理 Watch** | 一次性 Watch 处理不当，事件丢失 | 使用 Curator Cache |
| **ZNode 过深** | 10+ 层嵌套，读取效率低 | 控制在 5 层以内 |
| **偶数节点集群** | 比奇数多一台机器但不增加容错 | 始终使用奇数节点 |
| **sessionTimeout 过短** | GC 停顿导致误判超时 | 确保超时时间 > 最大 GC 停顿 |
| **不设置 namespace** | 多应用数据混乱 | 每个应用使用独立 namespace |
| **同步 API 阻塞** | 主线程长时间阻塞 | 使用 Curator 的 inBackground() |
| **忽略连接状态** | 未处理 LOST 导致状态不一致 | 注册 ConnectionStateListener |

# 8. 生产部署 Checklist

```text
上线前 Checklist：

□ 集群节点 >= 3（奇数）
□ dataLogDir 使用独立 SSD
□ JVM 堆大小合理（4-8GB）
□ 使用 G1GC 并配置 MaxGCPauseMillis < syncLimit * tickTime
□ 操作系统禁用 Swap
□ 文件句柄限制 >= 65536
□ maxClientCnxns 合理设置
□ autopurge 开启自动清理
□ 四字命令白名单配置
□ ACL 限制（禁用 world:anyone:cdrwa）
□ 使用 SSL/TLS（安全要求高的场景）
□ Prometheus 监控 + Grafana 仪表盘
□ 告警规则配置（宕机/延迟/Leader切换等）
□ 备份脚本就绪
□ 灾难恢复方案文档化
□ 与依赖方（Dubbo/Kafka 等）确认 ZK 版本兼容性
```
