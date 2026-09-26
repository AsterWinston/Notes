
# 1. Watch 概述

Watch（观察者）机制是 ZooKeeper 实现**分布式协调**的核心功能。它允许客户端在 ZNode 上注册监听器，当 ZNode 发生变化时，服务端主动向客户端推送事件通知。

```text
Watch 机制的设计哲学：

    "推" 模型替代 "拉" 模型
┌──────────────────────────────────────────────┐
│                                               │
│  传统轮询方式：                                │
│    Client ──不断──► Server ──相同数据──►     │
│    浪费带宽和 CPU                              │
│                                               │
│  Watch 事件驱动方式：                          │
│    Client ──注册──► Server ──变化时推送──►   │
│    按需通知，高效实时                          │
│                                               │
└──────────────────────────────────────────────┘
```

# 2. Watch 的核心特性

| 特性 | 说明 | 设计意图 |
|------|------|---------|
| **一次性触发（One-time Trigger）** | Watch 被触发后自动删除 | 简化服务端状态管理，避免内存无限增长 |
| **有序性** | 客户端看到的 Watch 事件顺序与更新顺序一致 | 保证客户端获取一致的视图 |
| **轻量级** | Watch 通知仅告知"发生了变化"，不包含变更后的数据 | 减少网络开销，客户端需自己重新读取 |
| **异步通知** | Watch 事件通过独立的线程发送 | 不阻塞服务端主处理流程 |
| **客户端保存** | Watch 数据保存在客户端，服务端重启后 Watch 丢失 | 避免服务端需要持久化和管理大量 Watch 状态 |

# 3. Watch 的类型

## 3.1 按触发条件分类

| 类型 | 触发条件 | 注册方式 | 说明 |
|------|---------|---------|------|
| `NodeCreated` | 被监听的 ZNode 被创建 | `exists()` + watch | 节点不存在时注册 |
| `NodeDeleted` | 被监听的 ZNode 被删除 | `exists()` + watch 或 `getData()` + watch | — |
| `NodeDataChanged` | ZNode 数据发生变化 | `getData()` + watch | setData 触发 |
| `NodeChildrenChanged` | 子节点列表变化（增/删子节点） | `getChildren()` + watch | 子节点自己的数据变化不触发 |

```text
Watch 类型与操作的对应关系：

操作                      触发的事件类型
────                      ──────────────
create("/a", data)  →  NodeCreated（对 /a 上的 exists watch）
                        NodeChildrenChanged（对 / 上的 getChildren watch）

delete("/a")        →  NodeDeleted（对 /a 上的 exists/getData watch）
                        NodeChildrenChanged（对 / 上的 getChildren watch）

setData("/a", data) →  NodeDataChanged（对 /a 上的 getData watch）

create("/a/b", d)   →  NodeChildrenChanged（对 /a 上的 getChildren watch）
                        （不会触发对 /a 的 NodeDataChanged）
```

## 3.2 按注册方式分类

```java
// === 标准 Watch（客户端库内置） ===
// exists — 节点存在性监听
zk.exists("/path", new Watcher() {
    public void process(WatchedEvent event) { ... }
});

// getData — 数据变化监听
zk.getData("/path", new Watcher() {...}, stat);

// getChildren — 子节点变化监听
zk.getChildren("/path", new Watcher() {...});

// === 持久化 Watch（ZooKeeper 3.6+） ===
// 标准 Watch 是一次性的，持久化 Watch 触发后不自动删除
zk.addWatch("/path", new Watcher() {...},
    AddWatchMode.PERSISTENT);          // 监听所有事件
zk.addWatch("/path", new Watcher() {...},
    AddWatchMode.PERSISTENT_RECURSIVE); // 递归监听所有子节点
```

# 4. WatchedEvent 结构

```java
// 客户端收到的事件对象
public class WatchedEvent {
    private final KeeperState keeperState;  // 连接状态
    private final EventType eventType;      // 事件类型
    private final String path;              // 触发事件的路径
}

// KeeperState 枚举
public enum KeeperState {
    SyncConnected,     // 正常连接
    Disconnected,      // 连接断开
    Expired,           // Session 过期
    AuthFailed,        // 认证失败
    ConnectedReadOnly, // 只读连接
    SaslAuthenticated, // SASL 认证成功
    Closed             // 连接关闭
}

// EventType 枚举
public enum EventType {
    None,                // 无事件（连接状态变化时使用）
    NodeCreated,         // 节点创建
    NodeDeleted,         // 节点删除
    NodeDataChanged,     // 节点数据变化
    NodeChildrenChanged  // 子节点变化
}
```

# 5. Watch 原理深入

## 5.1 服务端 Watch 存储

```java
// 服务端 WatchManager 核心数据结构
public class WatchManager {
    // 路径到 Watcher 的映射
    // key: ZNode 路径
    // value: 该路径上注册的 Watcher 集合
    private final Map<String, Set<Watcher>> watchTable =
        new HashMap<>();

    // Watcher 到路径的映射（用于 Session 清理时快速查找）
    // key: Watcher 对象
    // value: 该 Watcher 监听的路径集合
    private final Map<Watcher, Set<String>> watch2Paths =
        new HashMap<>();
}
```

```text
Watch 触发流程（服务端视角）：

1. 数据变更发生（如 setData 成功提交）
          │
2. 触发 WatchManager.triggerWatch(path, EventType)
          │
3. 在 watchTable 中查找 path 对应的 Watcher 集合
          │
4. 遍历 Watcher 集合，为每个 Watcher：
   ┌──────────────────────────────────────────────────┐
   │  a. 将 WatchedEvent 放入客户端的发送队列          │
   │  b. 从 watchTable 中移除该 Watcher（一次性触发）  │
   │  c. 从 watch2Paths 中移除该映射关系               │
   └──────────────────────────────────────────────────┘
          │
5. 客户端的 NIO 线程异步发送事件通知
```

## 5.2 客户端 Watch 处理

```java
// 客户端 Watch 相关核心类
public class ZooKeeper {
    // 客户端本地 Watch 管理器
    private final ZKWatchManager watchManager;

    // 默认 Watcher（构造函数传入）
    private final Watcher defaultWatcher;

    class ZKWatchManager implements ClientWatchManager {
        // 存储每个路径的 data watches
        private final Map<String, Set<Watcher>> dataWatches;

        // 存储每个路径的 child watches
        private final Map<String, Set<Watcher>> childWatches;

        // 存储 Persistent watch 信息
        private final Map<String, Set<Watcher>> persistentWatches;
    }
}
```

```text
客户端 Watch 处理流程：

1. 客户端接收到服务端发来的 WatchedEvent
          │
2. ClientCnxn.EventThread 线程处理事件
          │
3. 根据 KeeperState 和 EventType 决定处理方式：
   ┌──────────────────────────────────────────────────┐
   │                                                   │
   │  若 KeeperState == SyncConnected：               │
   │    → 正常状态的事件通知                           │
   │    → 从本地 ZKWatchManager 取出注册的 Watcher    │
   │    → 调用 watcher.process(event)                 │
   │                                                   │
   │  若 KeeperState == Disconnected：                │
   │    → 连接断开                                     │
   │    → 调用 defaultWatcher.process(event)          │
   │    → 启动重连                                     │
   │                                                   │
   │  若 KeeperState == Expired：                     │
   │    → Session 过期                                 │
   │    → 调用 defaultWatcher.process(event)          │
   │    → 清理所有 Watch                               │
   │    → 需要重新创建 Session                        │
   │                                                   │
   └──────────────────────────────────────────────────┘
```

# 6. Watch 的局限与解决方案

## 6.1 一次性触发的缺陷

```text
问题：一次性触发意味着客户端需要在每次收到通知后重新注册

    传统 Watch 循环：
    ┌──────────────────────────────────────────────┐
    │                                               │
    │  while (true) {                               │
    │    data = zk.getData("/config", watcher, st); │
    │    process(data);                             │
    │    // ... 等待事件触发                        │
    │  }                                            │
    │                                               │
    │  问题：                                       │
    │    1. 在 process(data) 和重新注册之间有空窗期 │
    │    2. 这段时间内的数据变更会被丢失             │
    │    3. 代码复杂，容易出错                       │
    │                                               │
    └──────────────────────────────────────────────┘

解决方案：
  1. 使用 Curator 的 NodeCache（自动重新注册 Watch）
  2. 使用 Curator 的 TreeCache（递归监听整棵树）
  3. 使用 ZooKeeper 3.6+ 的持久化 Watch
```

## 6.2 Watch 通知不包含变更数据

```text
问题：客户端收到通知后必须重新读取数据

    示例：
    ┌──────────────────────────────────────────────┐
    │                                               │
    │  1. Client 注册 Watch 监听 /config           │
    │  2. Server 更新 /config 的值                 │
    │  3. Client 收到 NodeDataChanged 通知         │
    │  4. Client 需要再次 getData("/config")       │
    │                                               │
    │  为什么不在事件中带数据？                       │
    │    · 减少网络传输负载                          │
    │    · 避免事件阶段数据又变了导致的歧义          │
    │    · 让客户端按需拉取最新数据                  │
    │                                               │
    └──────────────────────────────────────────────┘
```

# 7. Curator 对 Watch 的封装

Curator 框架对 ZooKeeper 原生 Watch 机制做了完善的高级封装，解决了原生 API 的痛点：

## 7.1 NodeCache

```java
// NodeCache：监听单个 ZNode 的数据变化
CuratorFramework client = CuratorFrameworkFactory.newClient(/*...*/);
client.start();

NodeCache nodeCache = new NodeCache(client, "/config");
nodeCache.getListenable().addListener(new NodeCacheListener() {
    @Override
    public void nodeChanged() throws Exception {
        ChildData currentData = nodeCache.getCurrentData();
        if (currentData != null) {
            String value = new String(currentData.getData());
            System.out.println("New config value: " + value);
        }
    }
});
nodeCache.start(); // 启动后自动注册 Watch + 自动重新注册
```

## 7.2 PathChildrenCache

```java
// PathChildrenCache：监听子节点的增删改
PathChildrenCache cache = new PathChildrenCache(client, "/services", true);
cache.getListenable().addListener(new PathChildrenCacheListener() {
    @Override
    public void childEvent(CuratorFramework client,
                           PathChildrenCacheEvent event) {
        switch (event.getType()) {
            case CHILD_ADDED:
                System.out.println("Node added: " + event.getData().getPath());
                break;
            case CHILD_REMOVED:
                System.out.println("Node removed: " + event.getData().getPath());
                break;
            case CHILD_UPDATED:
                System.out.println("Node updated: " + event.getData().getPath());
                break;
        }
    }
});
cache.start(PathChildrenCache.StartMode.BUILD_INITIAL_CACHE);
```

## 7.3 TreeCache

```java
// TreeCache：递归监听整个子树
TreeCache treeCache = new TreeCache(client, "/app");
treeCache.getListenable().addListener(new TreeCacheListener() {
    @Override
    public void childEvent(CuratorFramework client,
                           TreeCacheEvent event) {
        System.out.println("Event: " + event.getType() +
            " Path: " + (event.getData() != null ?
                event.getData().getPath() : "null"));
    }
});
treeCache.start();
// 自动处理所有子节点的 Watch 注册和重新注册
```

## 7.4 Curator Watch 机制对比

| 特性 | 原生 Watch | NodeCache | PathChildrenCache | TreeCache |
|------|-----------|-----------|------------------|-----------|
| 监听范围 | 单节点单次 | 单节点数据 | 子节点列表 | 递归整棵树 |
| 自动重注册 | ✗ | ✓ | ✓ | ✓ |
| 获取当前数据 | 需手动 get | ✓ (getCurrentData) | ✓ (getCurrentData) | ✓ |
| 事件类型 | 4 种 | 1 种（数据变化） | 3 种（增/删/改） | 5 种 |
| 持久化监听 | ✗（3.6+ 支持） | ✓ | ✓ | ✓ |
| 推荐场景 | 简单一次性监听 | 配置监听 | 服务发现 | 全量配置/状态 |

# 8. ZooKeeper 3.6+ 持久化 Watch

```java
// ZooKeeper 3.6+ 新增持久化 Watch API
ZooKeeper zk = new ZooKeeper("host:2181", 30000, watcher);

// PERSISTENT：不递归，触发后不删除
zk.addWatch("/config", watcher, AddWatchMode.PERSISTENT);

// PERSISTENT_RECURSIVE：递归监听所有子节点，触发后不删除
zk.addWatch("/services", watcher, AddWatchMode.PERSISTENT_RECURSIVE);
```

```text
持久化 Watch 与传统 Watch 对比：

┌──────────────────┬────────────────────┬──────────────────────┐
│ 特性             │  传统 Watch         │  持久化 Watch         │
├──────────────────┼────────────────────┼──────────────────────┤
│ 触发次数         │  一次性触发         │  持续触发             │
│ 自动重注册       │  需要手动           │  自动                 │
│ 递归监听         │  不支持             │  支持 (RECURSIVE)    │
│ 服务端开销       │  较低               │  稍高                 │
│ ZooKeeper 版本   │  所有版本           │  3.6+                │
│ Curator 支持     │  完全支持           │  3.6+ 支持           │
└──────────────────┴────────────────────┴──────────────────────┘
```

# 9. Watch 性能考量

```text
Watch 相关的性能指标：

┌────────────────────────────────────────────────────────┐
│                                                         │
│  1. Watch 数量限制                                      │
│     · ZooKeeper 本身没有硬性限制                        │
│     · 但每个 Watch 占用服务端内存                       │
│     · 建议：单个 ZooKeeper 集群不超过数十万个 Watch     │
│                                                         │
│  2. 风暴问题（Herd Effect）                             │
│     ┌──────────────────────────────────────────────┐   │
│     │                                               │   │
│     │  场景：1000 个客户端同时 Watch /config        │   │
│     │  /config 更新时：                             │   │
│     │    → 服务端需要给 1000 个客户端发送通知       │   │
│     │    → 1000 个客户端几乎同时请求读取最新数据     │   │
│     │    → 引发"惊群效应"，服务端瞬间压力巨大        │   │
│     │                                               │   │
│     │  解决方案：                                   │   │
│     │    · 不要把配置热数据放在同一个节点上          │   │
│     │    · 使用分组策略：/config/v1, /config/v2...  │   │
│     │    · 客户端随机延迟读取（jitter）              │   │
│     │    · 使用 Curator 的群组通知减负              │   │
│     └──────────────────────────────────────────────┘   │
│                                                         │
│  3. Watch 内存占用                                      │
│     · 每个 Watch 约 100-200 字节                       │
│     · 100 万 Watch ≈ 100-200MB 内存                    │
│     · 注意及时清理不再需要的 Watch                      │
│                                                         │
└────────────────────────────────────────────────────────┘
```

# 10. 最佳实践

```text
Watch 使用建议：

1. 尽量使用 Curator 高级 API
   ┌──────────────────────────────────────────────┐
   │  · NodeCache / PathChildrenCache / TreeCache │
   │  · 自动处理重连后的 Watch 重新注册           │
   │  · 避免手动处理一次性 Watch 的复杂度         │
   └──────────────────────────────────────────────┘

2. 避免在 Watch 回调中执行耗时操作
   ┌──────────────────────────────────────────────┐
   │  · Watch 回调在 EventThread 中执行           │
   │  · 耗时操作会阻塞后续事件的处理              │
   │  · 应将耗时逻辑放入独立的线程池              │
   └──────────────────────────────────────────────┘

3. 合理设计 ZNode 粒度
   ┌──────────────────────────────────────────────┐
   │  · 避免过于细粒度导致大量 Watch              │
   │  · 避免过于粗粒度导致通知风暴                │
   │  · 按业务域合理拆分                          │
   └──────────────────────────────────────────────┘

4. 注意重连时的 Watch 重建
   ┌──────────────────────────────────────────────┐
   │  · Session 过期后，所有 Watch 丢失           │
   │  · 重连后需要重新创建 Session 并重新注册 Watch│
   │  · Curator 的 ConnectionStateListener 可感知  │
   └──────────────────────────────────────────────┘

5. 监控 Watch 指标
   ┌──────────────────────────────────────────────┐
   │  · 使用 mntr 命令查看 watch 数量             │
   │  · 监控 watch 总数的增长趋势                  │
   │  · 设置告警阈值                              │
   └──────────────────────────────────────────────┘
```
