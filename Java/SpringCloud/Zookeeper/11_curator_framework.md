
# 1. Curator 概述

Apache Curator 是 Netflix 开源的 ZooKeeper 客户端框架，现在是 Apache 顶级项目。它解决了原生 ZooKeeper API 的诸多痛点，提供了高级抽象、自动重连、连接状态管理以及丰富的分布式协调 Recipe（食谱/模式）。

```text
Curator vs 原生 API：
┌──────────────────────────────────────────────────────────┐
│                                                           │
│  原生 ZooKeeper API：                                     │
│    · 一次性 Watch（需手动重注册）                         │
│    · 无自动重连                                           │
│    · 异常处理复杂                                         │
│    · 无高级分布式原语                                     │
│                                                           │
│  Curator Framework：                                      │
│    · 自动重连 + 状态监听                                  │
│    · 持久化 Watcher（NodeCache 等）                       │
│    · 丰富的分布式 Recipe（锁/选举/队列/屏障等）           │
│    · Fluent API（链式调用风格）                            │
│    · 连接池管理（LeaderLatch 等）                         │
│    · 重试策略可配置                                       │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

```xml
<!-- Maven 依赖 -->
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-framework</artifactId>
    <version>5.6.0</version>
</dependency>

<!-- Recipes（分布式协调模式） -->
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-recipes</artifactId>
    <version>5.6.0</version>
</dependency>

<!-- 与 ZK 3.5+ 兼容的依赖 -->
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-client</artifactId>
    <version>5.6.0</version>
</dependency>
```

# 2. CuratorFramework 基础

## 2.1 创建客户端

```java
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.retry.ExponentialBackoffRetry;

// 方式一：Fluent Builder（推荐）
CuratorFramework client = CuratorFrameworkFactory.builder()
    .connectString("host1:2181,host2:2181,host3:2181")
    .sessionTimeoutMs(30000)       // Session 超时
    .connectionTimeoutMs(15000)   // 连接超时
    .retryPolicy(new ExponentialBackoffRetry(
        1000,  // baseSleepTimeMs：基础睡眠时间
        3     // maxRetries：最大重试次数
    ))
    .namespace("myapp")           // 命名空间（所有路径自动加 /myapp 前缀）
    .canBeReadOnly(false)
    .build();

// 方式二：静态工厂方法
CuratorFramework client = CuratorFrameworkFactory.newClient(
    "host1:2181,host2:2181,host3:2181",
    30000,  // sessionTimeout
    15000,  // connectionTimeout
    new ExponentialBackoffRetry(1000, 3)
);

// 启动客户端（必须调用）
client.start();

// 在使用完毕后关闭
client.close();
```

## 2.2 重试策略

Curator 提供了多种重试策略：

| 策略 | 说明 | 适用场景 |
|------|------|---------|
| `ExponentialBackoffRetry` | 指数退避：1s → 2s → 4s → ... | 推荐，生产首选 |
| `RetryNTimes` | 固定次数重试：N 次 | 确定重试次数的场景 |
| `RetryOneTime` | 仅重试一次 | 轻量场景 |
| `RetryUntilElapsed` | 在指定时间内持续重试 | 网络临时性问题 |
| `SleepingRetry` | 固定间隔重试 | 简单场景 |
| `BoundedExponentialBackoffRetry` | 有上限的指数退避 | 避免退避时间过长 |

```java
// ExponentialBackoffRetry — 指数退避
new ExponentialBackoffRetry(
    1000,   // baseSleepTimeMs：基础等待时间 1s
    3,      // maxRetries：最大重试 3 次
    5000    // maxSleepMs：最大等待时间上限（可选）
);
// 重试间隔：1s → 2s → 4s → (最多 5s)

// RetryNTimes — 固定次数
new RetryNTimes(5, 1000);  // 重试 5 次，每次间隔 1s

// RetryUntilElapsed — 时间窗口内持续重试
new RetryUntilElapsed(30000, 1000); // 30s 内，每次间隔 1s
```

## 2.3 Fluent API 操作

```java
// Curator 提供链式 API（Fluent API）风格操作

// === 创建节点 ===
client.create()
    .creatingParentsIfNeeded()          // 自动创建父节点
    .withMode(CreateMode.PERSISTENT)    // 节点类型
    .withACL(ZooDefs.Ids.OPEN_ACL_UNSAFE)
    .forPath("/config/db", "jdbc:mysql://localhost/db".getBytes());

// withProtection() — 创建临时顺序节点时防止"孤儿节点"
client.create()
    .withMode(CreateMode.EPHEMERAL_SEQUENTIAL)
    .withProtection()  // 加保护模式（GUID 前缀）
    .forPath("/locks/lock-", data);

// === 读取数据 ===
byte[] data = client.getData()
    .forPath("/config/db");

// 读取并获取 Stat
Stat stat = new Stat();
byte[] data = client.getData()
    .storingStatIn(stat)
    .forPath("/config/db");

// 读取带 Watch（使用 NodeCache 更好）
byte[] data = client.getData()
    .watched()
    .forPath("/config/db");

// 检查是否存在
Stat stat = client.checkExists()
    .forPath("/config/db");

// === 更新数据 ===
Stat newStat = client.setData()
    .withVersion(stat.getVersion())  // 乐观锁
    .forPath("/config/db", "new value".getBytes());

// === 删除节点 ===
// 删除单节点
client.delete()
    .withVersion(stat.getVersion())
    .forPath("/config/db");

// 递归删除（删除节点及其所有子节点）
client.delete()
    .deletingChildrenIfNeeded()
    .forPath("/config");

// 静默删除（不抛异常即使节点不存在）
client.delete()
    .quietly()
    .forPath("/config/db");

// 保证删除（重试直到成功或 Session 过期）
client.delete()
    .guaranteed()
    .forPath("/important-ephemeral-node");

// === 获取子节点 ===
List<String> children = client.getChildren()
    .forPath("/services");
```

# 3. 连接状态监听

```java
// ConnectionStateListener — 监听连接状态变化
client.getConnectionStateListenable().addListener(
    new ConnectionStateListener() {
        @Override
        public void stateChanged(CuratorFramework client,
                                 ConnectionState newState) {
            switch (newState) {
                case CONNECTED:
                    // 首次连接成功或重连成功
                    System.out.println("Connected to ZooKeeper");
                    break;
                case RECONNECTED:
                    // 断开后重新连接成功
                    // 注意：临时节点在此期间可能丢失
                    System.out.println("Reconnected");
                    break;
                case LOST:
                    // Session 过期，所有临时节点和 Watch 已丢失
                    // 需要重新创建所有状态
                    System.out.println("Session lost, reinitialize!");
                    break;
                case SUSPENDED:
                    // 连接暂时挂起（未断开，但无法通信）
                    System.out.println("Connection suspended");
                    break;
                case READ_ONLY:
                    // 只读模式
                    break;
            }
        }
    });
```

# 4. 高级 Recipe（分布式协调原语）

Curator Recipes 模块提供了丰富的分布式协调模式：

## 4.1 分布式锁

```java
// InterProcessMutex — 可重入互斥锁
InterProcessMutex lock = new InterProcessMutex(
    client,
    "/locks/resource-a"
);

if (lock.acquire(5, TimeUnit.SECONDS)) {
    try {
        // 临界区代码
        doWork();
    } finally {
        lock.release();
    }
}

// InterProcessReadWriteLock — 读写锁
InterProcessReadWriteLock rwLock = new InterProcessReadWriteLock(
    client, "/locks/rw-resource"
);
InterProcessMutex readLock = rwLock.readLock();
InterProcessMutex writeLock = rwLock.writeLock();

// InterProcessSemaphoreMutex — 不可重入互斥锁
InterProcessSemaphoreMutex semaphore = new InterProcessSemaphoreMutex(
    client, "/locks/semaphore"
);
```

## 4.2 Leader 选举

```java
// LeaderLatch — 简单 Leader 选举
List<LeaderLatch> latches = new ArrayList<>();
for (int i = 0; i < 3; i++) {
    LeaderLatch latch = new LeaderLatch(
        client,
        "/election/leader"
    );
    latch.addListener(new LeaderLatchListener() {
        @Override
        public void isLeader() {
            System.out.println("I am the leader now!");
        }

        @Override
        public void notLeader() {
            System.out.println("I am not the leader");
        }
    });
    latch.start();
    latches.add(latch);
}
// LeaderLatch 会保持连接直到 close()，期间 Leader 断开后自动切换

// LeaderSelector — 带任务执行的 Leader 选举
LeaderSelector selector = new LeaderSelector(
    client,
    "/election/leader-task",
    new LeaderSelectorListenerAdapter() {
        @Override
        public void takeLeadership(CuratorFramework client)
                throws Exception {
            // 只有 Leader 执行此方法
            System.out.println("I am the leader, doing work...");
            // 任务完成后 release，触发重新选举
        }
    }
);
selector.autoRequeue();  // 任务完成后自动重新参与选举
selector.start();
```

## 4.3 分布式 Barrier

```java
// DistributedBarrier — 分布式栅栏
DistributedBarrier barrier = new DistributedBarrier(
    client, "/barriers/task"
);

// 所有参与者调用
barrier.setBarrier();   // 设置栅栏
barrier.waitOnBarrier(); // 等待所有参与者就绪

// 主控方调用
barrier.removeBarrier(); // 解除栅栏，所有等待方继续

// DistributedDoubleBarrier — 双栅栏（进入和离开都同步）
DistributedDoubleBarrier doubleBarrier = new DistributedDoubleBarrier(
    client, "/barriers/double", 3  // memberQty：参与成员数
);
doubleBarrier.enter();  // 等待所有成员进入
// 所有成员一起执行任务
doubleBarrier.leave();  // 等待所有成员完成
```

## 4.4 分布式计数器

```java
// DistributedAtomicLong — 原子长整型
DistributedAtomicLong atomicLong = new DistributedAtomicLong(
    client, "/counters/order-id",
    new RetryNTimes(3, 1000)
);

AtomicValue<Long> value = atomicLong.increment();
if (value.succeeded()) {
    System.out.println("New value: " + value.postValue());
}

// DistributedAtomicInteger — 原子整型
DistributedAtomicInteger atomicInt = new DistributedAtomicInteger(
    client, "/counters/task", retryPolicy
);

// SharedCount — 共享计数器（基于版本号乐观锁）
SharedCount sharedCount = new SharedCount(
    client, "/counters/shared", 0  // 初始值
);
sharedCount.start();
sharedCount.addListener(new SharedCountListener() {
    @Override
    public void countHasChanged(SharedCountReader reader, int newCount) {
        System.out.println("Count changed to: " + newCount);
    }
});
sharedCount.trySetCount(sharedCount.getVersionedValue(), 5);
```

## 4.5 分布式队列

```java
// DistributedQueue — 分布式队列
DistributedQueue<String> queue = QueueBuilder.builder(
    client,
    null,                        // consumer
    new QueueSerializer<String>() {
        @Override
        public byte[] serialize(String item) {
            return item.getBytes();
        }
        @Override
        public String deserialize(byte[] bytes) {
            return new String(bytes);
        }
    },
    "/queues/tasks"              // queuePath
).buildQueue();
queue.start();

// 生产者
queue.put("task-1");
queue.put("task-2");

// 消费者
String item;
while ((item = queue.poll(5, TimeUnit.SECONDS)) != null) {
    processItem(item);
}
```

# 5. Curator Cache（高级 Watch 封装）

## 5.1 NodeCache

```java
// NodeCache — 监听单个节点数据变化
NodeCache nodeCache = new NodeCache(client, "/config/db");
nodeCache.start(true); // true = 启动时立即加载数据

nodeCache.getListenable().addListener(new NodeCacheListener() {
    @Override
    public void nodeChanged() throws Exception {
        ChildData data = nodeCache.getCurrentData();
        if (data != null) {
            String value = new String(data.getData());
            System.out.println("Config updated: " + value);
        } else {
            System.out.println("Config node deleted");
        }
    }
});

// 获取当前数据
ChildData current = nodeCache.getCurrentData();
if (current != null) {
    String path = current.getPath();
    byte[] data = current.getData();
    Stat stat = current.getStat();
}

// 关闭
nodeCache.close();
```

## 5.2 PathChildrenCache

```java
// PathChildrenCache — 监听子节点变化
PathChildrenCache cache = new PathChildrenCache(
    client,
    "/services",
    true  // cacheData：是否缓存子节点数据
);

// 启动模式
cache.start(PathChildrenCache.StartMode.BUILD_INITIAL_CACHE);
// StartMode:
//   NORMAL — 异步初始化缓存
//   BUILD_INITIAL_CACHE — 同步初始化缓存（启动后回调前已构建）
//   POST_INITIALIZED_EVENT — 初始化完成后发送 INITIALIZED 事件

cache.getListenable().addListener(new PathChildrenCacheListener() {
    @Override
    public void childEvent(CuratorFramework client,
                           PathChildrenCacheEvent event) {
        ChildData data = event.getData();
        switch (event.getType()) {
            case CHILD_ADDED:
                System.out.println("Service added: " +
                    data.getPath() + " = " +
                    new String(data.getData()));
                break;
            case CHILD_REMOVED:
                System.out.println("Service removed: " +
                    data.getPath());
                break;
            case CHILD_UPDATED:
                System.out.println("Service updated: " +
                    data.getPath() + " = " +
                    new String(data.getData()));
                break;
            case CONNECTION_SUSPENDED:
            case CONNECTION_RECONNECTED:
                // 自动恢复
                break;
        }
    }
});

cache.close();
```

## 5.3 TreeCache

```java
// TreeCache — 递归监听整个子树（功能最强大）
TreeCache treeCache = TreeCache.newBuilder(client, "/app")
    .setCacheData(true)                 // 缓存数据
    .setMaxDepth(5)                    // 最大深度
    .setExecutor(executorService)       // 自定义线程池
    .build();

treeCache.getListenable().addListener(new TreeCacheListener() {
    @Override
    public void childEvent(CuratorFramework client,
                           TreeCacheEvent event) {
        switch (event.getType()) {
            case NODE_ADDED:
                System.out.println("Added: " +
                    event.getData().getPath());
                break;
            case NODE_REMOVED:
                System.out.println("Removed: " +
                    event.getData().getPath());
                break;
            case NODE_UPDATED:
                System.out.println("Updated: " +
                    event.getData().getPath());
                break;
            case CONNECTION_LOST:
                System.out.println("Connection lost, reconnecting...");
                break;
            case CONNECTION_RECONNECTED:
                System.out.println("Reconnected");
                break;
            case INITIALIZED:
                System.out.println("Cache initialized");
                break;
        }
    }
});

treeCache.start();

// 获取当前状态
Map<String, ChildData> currentChildren = treeCache.getCurrentChildren("/app");
for (Map.Entry<String, ChildData> entry : currentChildren.entrySet()) {
    System.out.println(entry.getKey() + " = " +
        new String(entry.getValue().getData()));
}

treeCache.close();
```

# 6. Curator Cache 对比

| 特性 | NodeCache | PathChildrenCache | TreeCache |
|------|-----------|------------------|-----------|
| **监听范围** | 单个节点数据 | 子节点列表 | 递归整棵子树 |
| **缓存数据** | 是 | 可选 | 可选 |
| **事件类型** | 数据变化/删除 | 子节点增/删/改 | 5 种完整事件 |
| **递归** | 否 | 否 | 是（可设深度） |
| **自动重连** | 是 | 是 | 是 |
| **内存占用** | 低 | 低-中 | 中-高 |
| **推荐场景** | 配置项监听 | 服务列表发现 | 全量配置/状态树 |

# 7. 高级特性

## 7.1 Curator Transaction（事务）

```java
// 类似 multi 操作，原子批量执行
Collection<CuratorTransactionResult> results =
    client.inTransaction()
        .create().forPath("/a", "data-a".getBytes())
        .and()
        .create().forPath("/a/b", "data-b".getBytes())
        .and()
        .setData().forPath("/config", "updated".getBytes())
        .and()
        .commit();
```

## 7.2 Service Discovery

```java
// Curator 提供的服务发现扩展
ServiceDiscovery<InstanceDetails> discovery = ServiceDiscoveryBuilder
    .builder(InstanceDetails.class)
    .client(client)
    .basePath("/services")
    .serializer(new JacksonInstanceSerializer<>(InstanceDetails.class))
    .thisInstance(serviceInstance)
    .build();

// 注册服务
discovery.start();

// 查询服务
Collection<ServiceInstance<InstanceDetails>> instances =
    discovery.queryForInstances("order-service");

// 监听服务变化
ServiceCache<InstanceDetails> cache = discovery
    .serviceCacheBuilder()
    .name("order-service")
    .build();
cache.start();
cache.addListener(new ServiceCacheListener() {
    @Override
    public void cacheChanged() {
        System.out.println("Service instances changed");
    }
});
```

## 7.3 Curator Async（异步操作）

```java
// Curator 支持异步操作（使用 CompletableFuture）
client.create()
    .withMode(CreateMode.PERSISTENT)
    .inBackground()  // 异步执行
    .forPath("/async-node", "data".getBytes());

// 带回调的异步操作
CountDownLatch latch = new CountDownLatch(1);
client.setData()
    .inBackground(new BackgroundCallback() {
        @Override
        public void processResult(CuratorFramework client,
                                  CuratorEvent event) {
            switch (event.getType()) {
                case SET_DATA:
                    System.out.println("Set data result: " +
                        event.getResultCode());
                    break;
            }
            latch.countDown();
        }
    })
    .forPath("/config", "new-value".getBytes());
latch.await();
```

# 8. Curator 最佳实践

```text
最佳实践总结：

1. 使用 Curator 而非原生 API
   ┌──────────────────────────────────────────────────┐
   │  · 除非有特殊需求，否则始终使用 Curator           │
   │  · 自动处理重连、Watch 重注册、Session 恢复       │
   └──────────────────────────────────────────────────┘

2. 合理配置重试策略
   ┌──────────────────────────────────────────────────┐
   │  · 生产环境推荐 ExponentialBackoffRetry          │
   │  · baseSleepTimeMs 不宜过小（1000ms 起步）        │
   │  · maxRetries 不宜过大（3-5 次）                  │
   └──────────────────────────────────────────────────┘

3. 使用 namespace 隔离
   ┌──────────────────────────────────────────────────┐
   │  · .namespace("myapp")                           │
   │  · 所有路径自动加前缀：/myapp/config              │
   │  · 多应用共享同一个 ZK 集群时避免冲突             │
   └──────────────────────────────────────────────────┘

4. 监听连接状态
   ┌──────────────────────────────────────────────────┐
   │  · 注册 ConnectionStateListener                  │
   │  · LOST 状态下重建所有必要的状态                  │
   │  · Curator Cache 自动处理 RECONNECTED            │
   └──────────────────────────────────────────────────┘

5. 使用合适的 Cache
   ┌──────────────────────────────────────────────────┐
   │  · 单值：NodeCache                               │
   │  · 列表：PathChildrenCache                       │
   │  · 全树：TreeCache                               │
   │  · 避免在 RUNNING 状态前就使用缓存数据            │
   └──────────────────────────────────────────────────┘

6. 资源清理
   ┌──────────────────────────────────────────────────┐
   │  · 使用 try-with-resources 或 finally 关闭连接   │
   │  · 关闭 Cache、Lock、LeaderLatch 等资源          │
   │  · 使用 Spring 容器管理 CuratorFramework 生命周期 │
   └──────────────────────────────────────────────────┘
```
