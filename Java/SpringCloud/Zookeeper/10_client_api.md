
# 1. ZooKeeper 客户端概述

ZooKeeper 提供了官方的 Java 客户端库，是最底层、最直接的交互方式。所有的第三方框架（如 Curator）都构建在此原生 API 之上。

```xml
<!-- Maven 依赖 -->
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.9.2</version>
</dependency>
```

```text
客户端架构概览：
┌──────────────────────────────────────────────────────┐
│                                                       │
│  ZooKeeper（主类，客户端入口）                         │
│    │                                                  │
│    ├── ClientCnxn（网络连接管理）                      │
│    │    · SendThread：发送请求和处理响应               │
│    │    · EventThread：处理 Watch 事件回调             │
│    │                                                  │
│    ├── ZKWatchManager（Watch 管理器）                 │
│    │    · dataWatches                                │
│    │    · childWatches                               │
│    │                                                  │
│    └── HostProvider（地址解析与负载均衡）              │
│         · StaticHostProvider（默认）                  │
│                                                       │
└──────────────────────────────────────────────────────┘
```

# 2. 创建 ZooKeeper 连接

## 2.1 基础连接

```java
import org.apache.zookeeper.*;

public class ZkClientExample {
    private ZooKeeper zooKeeper;
    private final CountDownLatch connectedLatch = new CountDownLatch(1);

    public void connect() throws Exception {
        // ZooKeeper 构造函数是异步的，需要通过 Watcher 获知连接成功
        zooKeeper = new ZooKeeper(
            "127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183", // connectString
            30000,  // sessionTimeout（毫秒）
            new Watcher() {
                @Override
                public void process(WatchedEvent event) {
                    if (event.getState() ==
                        Event.KeeperState.SyncConnected) {
                        connectedLatch.countDown();
                    }
                }
            }
        );
        connectedLatch.await(); // 等待连接建立
        System.out.println("Connected! SessionId: " +
            zooKeeper.getSessionId());
    }
}
```

## 2.2 高级连接配置

```java
// 使用工厂方法创建（ZooKeeper 3.5+）
ZooKeeper zooKeeper = new ZooKeeperBuilder()
    .connectString("host1:2181,host2:2181,host3:2181")
    .sessionTimeout(30000)
    .watcher(watcher)
    .hostProvider(new StaticHostProvider(...))
    .canBeReadOnly(false)
    .build();

// 只读模式连接
ZooKeeper readOnlyZk = new ZooKeeper(
    "host:2181",
    30000,
    watcher,
    true  // canBeReadOnly
);
// 当连接的节点处于只读模式时（如与集群中断），仍可读
```

# 3. CRUD 操作详解

## 3.1 创建节点

```java
// create 重载方法签名
String create(
    String path,       // ZNode 路径
    byte[] data,       // 数据（字节数组）
    List<ACL> acl,     // ACL 列表（可使用 ZooDefs.Ids.OPEN_ACL_UNSAFE）
    CreateMode mode    // 创建模式
) throws KeeperException, InterruptedException;

// 示例：创建持久节点
String path = zooKeeper.create(
    "/config",
    "server.port=8080".getBytes(StandardCharsets.UTF_8),
    ZooDefs.Ids.OPEN_ACL_UNSAFE,
    CreateMode.PERSISTENT
);

// 创建临时节点
zooKeeper.create(
    "/services/order/provider-",
    "192.168.1.1:8080".getBytes(),
    ZooDefs.Ids.OPEN_ACL_UNSAFE,
    CreateMode.EPHEMERAL
);

// 创建临时顺序节点
String seqPath = zooKeeper.create(
    "/locks/lock-",
    "data".getBytes(),
    ZooDefs.Ids.OPEN_ACL_UNSAFE,
    CreateMode.EPHEMERAL_SEQUENTIAL
);
// 返回：/locks/lock-0000000000（自动追加序号）
```

```java
// CreateMode 枚举值
public enum CreateMode {
    PERSISTENT,                    // 持久节点
    PERSISTENT_SEQUENTIAL,         // 持久顺序节点
    EPHEMERAL,                     // 临时节点
    EPHEMERAL_SEQUENTIAL,          // 临时顺序节点
    CONTAINER,                     // 容器节点（3.5+，子节点为空时自动删除）
    PERSISTENT_WITH_TTL,           // 带 TTL 持久节点（3.5+）
    PERSISTENT_SEQUENTIAL_WITH_TTL // 带 TTL 持久顺序节点（3.5+）
}
```

## 3.2 读取数据

```java
// getData — 读取节点数据
Stat stat = new Stat();
byte[] data = zooKeeper.getData(
    "/config",      // 路径
    false,          // watch（是否注册 Watcher）
    stat            // Stat 对象（输出参数，获取元数据）
);
String value = new String(data, StandardCharsets.UTF_8);
System.out.println("Data: " + value);
System.out.println("Version: " + stat.getVersion());
System.out.println("Children: " + stat.getNumChildren());

// 注册 Watcher 版
byte[] data = zooKeeper.getData(
    "/config",
    new Watcher() {
        @Override
        public void process(WatchedEvent event) {
            if (event.getType() ==
                Event.EventType.NodeDataChanged) {
                System.out.println("Config changed!");
                // 需要重新读取数据
            }
        }
    },
    stat
);

// exists — 检查节点是否存在
Stat stat = zooKeeper.exists("/config", false);
if (stat != null) {
    System.out.println("Node exists, created at: " +
        new Date(stat.getCtime()));
}
```

## 3.3 更新数据

```java
// setData — 更新节点数据（支持版本控制）
Stat stat = zooKeeper.setData(
    "/config",
    "server.port=9090".getBytes(StandardCharsets.UTF_8),
    -1  // version = -1 表示不检查版本（强制更新）
);

// 乐观锁更新
Stat currentStat = new Stat();
zooKeeper.getData("/config", false, currentStat);

try {
    Stat newStat = zooKeeper.setData(
        "/config",
        "server.port=9090".getBytes(),
        currentStat.getVersion()  // 指定期望版本
    );
    System.out.println("Update success, new version: " +
        newStat.getVersion());
} catch (KeeperException.BadVersionException e) {
    System.out.println("Version conflict, need retry");
}
```

## 3.4 删除节点

```java
// delete — 删除节点（支持版本控制）
zooKeeper.delete(
    "/config",
    -1  // version = -1 表示不检查版本
);

// 带版本控制的删除
Stat stat = new Stat();
zooKeeper.exists("/config", false);
zooKeeper.delete("/config", stat.getVersion());

// deleteAll — 递归删除（3.5+）
// 原生 API 不支持递归删除，需自行实现或使用 Curator
void deleteRecursive(ZooKeeper zk, String path) throws Exception {
    List<String> children = zk.getChildren(path, false);
    for (String child : children) {
        deleteRecursive(zk, path + "/" + child);
    }
    zk.delete(path, -1);
}

// Curator 方式：
client.delete().deletingChildrenIfNeeded().forPath("/path");
```

## 3.5 获取子节点

```java
// getChildren — 获取子节点列表
List<String> children = zooKeeper.getChildren("/services", false);
for (String child : children) {
    System.out.println("  - " + child);
}

// 带 Watch 的版本
List<String> children = zooKeeper.getChildren(
    "/services",
    new Watcher() {
        @Override
        public void process(WatchedEvent event) {
            if (event.getType() ==
                Event.EventType.NodeChildrenChanged) {
                System.out.println("Children changed!");
            }
        }
    }
);

// 带 Stat 的版本
Stat stat = new Stat();
List<String> children = zooKeeper.getChildren("/services", false, stat);
System.out.println("Num children: " + stat.getNumChildren());
```

# 4. 异步 API

ZooKeeper 的所有同步操作都有对应的异步版本：

```java
// 异步创建节点
zooKeeper.create(
    "/asyncNode",
    "data".getBytes(),
    ZooDefs.Ids.OPEN_ACL_UNSAFE,
    CreateMode.PERSISTENT,
    new AsyncCallback.StringCallback() {
        @Override
        public void processResult(
            int rc,          // 返回码：0 = 成功
            String path,     // 请求的路径
            Object ctx,      // 上下文对象
            String name      // 实际创建的路径（顺序节点时有用）
        ) {
            if (rc == KeeperException.Code.OK.intValue()) {
                System.out.println("Created: " + name);
            } else {
                System.out.println("Failed: " +
                    KeeperException.Code.get(rc));
            }
        }
    },
    null  // ctx：上下文对象，会原样传回回调方法
);

// 异步读取数据
zooKeeper.getData(
    "/config",
    false,
    new AsyncCallback.DataCallback() {
        @Override
        public void processResult(
            int rc, String path, Object ctx,
            byte[] data, Stat stat
        ) {
            // 处理返回数据
        }
    },
    null
);

// 异步检查存在
zooKeeper.exists(
    "/config",
    false,
    new AsyncCallback.StatCallback() {
        @Override
        public void processResult(
            int rc, String path, Object ctx, Stat stat
        ) {
            if (stat != null) { /* 节点存在 */ }
        }
    },
    null
);
```

# 5. Multi 操作（原子批量）

```java
// ZooKeeper 3.4+ 支持原子批量操作
List<Op> ops = Arrays.asList(
    Op.create("/a", "data-a".getBytes(),
        ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT),
    Op.create("/a/b", "data-b".getBytes(),
        ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT),
    Op.setData("/config", "updated".getBytes(), -1)
);

try {
    List<OpResult> results = zooKeeper.multi(ops);
    for (OpResult result : results) {
        if (result instanceof OpResult.CreateResult) {
            String path = ((OpResult.CreateResult) result).getPath();
            System.out.println("Created: " + path);
        }
    }
} catch (KeeperException e) {
    // 整个 multi 操作失败（原子性的体现）
    System.out.println("Multi operation failed: " + e.getMessage());
}

// 异步 Multi
zooKeeper.multi(ops, new AsyncCallback.MultiCallback() {
    @Override
    public void processResult(int rc, String path,
        Object ctx, List<OpResult> results) {
        // 处理结果
    }
}, null);
```

# 6. 异常处理

```java
// ZooKeeper API 可能抛出的异常
public abstract class KeeperException extends Exception {
    public enum Code {
        OK,                       // 成功
        SYSTEMERROR,              // 系统错误
        RUNTIMEINCONSISTENCY,     // 运行时不一致
        DATAINCONSISTENCY,        // 数据不一致
        CONNECTIONLOSS,           // 连接丢失（可重试）
        MARSHALLINGERROR,         // 序列化错误
        UNIMPLEMENTED,            // 未实现
        OPERATIONTIMEOUT,         // 操作超时
        BADARGUMENTS,             // 参数错误
        APIERROR,                 // API 错误
        NONODE,                   // 节点不存在
        NOAUTH,                   // 无权限
        BADVERSION,               // 版本冲突
        NOCHILDRENFOREPHEMERALS,  // 临时节点不能有子节点
        NODEEXISTS,               // 节点已存在
        NOTEMPTY,                 // 节点非空（有子节点）
        SESSIONEXPIRED,           // Session 过期
        INVALIDCALLBACK,          // 无效回调
        INVALIDACL,               // 无效 ACL
        AUTHFAILED,               // 认证失败
        SESSIONMOVED,             // Session 迁移（3.5+）
        NOTREADONLY,              // 非只读模式
        EPHEMERALONLOCALSESSION,  // 本地 Session 临时节点
        REQUESTTIMEOUT,           // 请求超时（3.5+）
    }

    // 可重试的异常判断
    public static boolean isRetryable(int rc) {
        return rc == Code.CONNECTIONLOSS.intValue() ||
               rc == Code.OPERATIONTIMEOUT.intValue() ||
               rc == Code.SESSIONMOVED.intValue() ||
               rc == Code.REQUESTTIMEOUT.intValue();
    }
}
```

```java
// 推荐的异常处理模式
public byte[] getDataWithRetry(ZooKeeper zk, String path, int maxRetries)
        throws Exception {
    for (int i = 0; i < maxRetries; i++) {
        try {
            return zk.getData(path, false, null);
        } catch (KeeperException.ConnectionLossException e) {
            // 连接丢失，可重试（idempotent 操作）
            if (i == maxRetries - 1) throw e;
            Thread.sleep(1000 * (i + 1)); // 指数退避
        } catch (KeeperException.SessionExpiredException e) {
            // Session 过期，不可重试（需要重新创建 session）
            throw e;
        }
    }
    throw new RuntimeException("Max retries exceeded");
}
```

# 7. ZooKeeper 客户端内部线程模型

```text
ZooKeeper 客户端内部线程：

┌──────────────────────────────────────────────────────────┐
│                                                           │
│  SendThread（发送线程）：                                  │
│    · 负责所有 RPC 请求的发送和响应接收                     │
│    · 维护与服务端的 TCP 连接（NIO）                        │
│    · 发送心跳 PING 保活 Session                           │
│    · 处理连接断开和重连                                   │
│    · 单线程模型，所有请求顺序发送                          │
│                                                           │
│  EventThread（事件线程）：                                 │
│    · 处理服务端推送的 Watch 事件                          │
│    · 调用客户端注册的 Watcher 回调                        │
│    · 执行 ConnectionStateListener 回调                   │
│    · 单线程模型，保证回调的顺序性                          │
│                                                           │
│  主线程（调用者线程）：                                    │
│    · 调用 ZooKeeper API 的同步方法                        │
│    · 同步方法内部阻塞等待 SendThread 完成                 │
│                                                           │
│  重要提示：                                               │
│    · 不要在 Watcher 回调（EventThread）中执行耗时操作     │
│    · 耗时操作会阻塞所有后续事件处理                       │
│    · 应将耗时逻辑放入独立线程池                           │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

# 8. 最佳实践与常见陷阱

## 8.1 连接管理

```java
// ✓ 正确：使用 CountDownLatch 等待连接
CountDownLatch latch = new CountDownLatch(1);
ZooKeeper zk = new ZooKeeper(
    "host:2181", 30000, event -> {
        if (event.getState() == SyncConnected) {
            latch.countDown();
        }
    });
latch.await(); // 阻塞直到连接成功

// ✗ 错误：创建后直接使用（连接尚未建立）
ZooKeeper zk = new ZooKeeper("host:2181", 30000, watcher);
zk.getData("/path", false, null); // 可能抛 ConnectionLoss
```

## 8.2 资源释放

```java
// ✓ 正确：在 finally 中关闭连接
ZooKeeper zk = null;
try {
    zk = new ZooKeeper("host:2181", 30000, watcher);
    // ... 使用 zk
} finally {
    if (zk != null) {
        zk.close(); // 释放资源
    }
}

// try-with-resources（ZooKeeper 实现了 AutoCloseable）
try (ZooKeeper zk = new ZooKeeper("host:2181", 30000, watcher)) {
    // ... 使用 zk
} // 自动调用 close()
```

## 8.3 数据序列化

```java
// ✓ 推荐：使用明确的编码
String data = "hello world";
byte[] bytes = data.getBytes(StandardCharsets.UTF_8);

// ✗ 避免：使用默认编码（平台相关）
byte[] bytes = data.getBytes();

// 大对象序列化
MyObject obj = new MyObject();
byte[] bytes = serializeToJson(obj); // 使用 JSON/Protobuf 等
if (bytes.length > 1024 * 1024) {    // 检查 1MB 限制
    throw new IllegalStateException("Data too large");
}
```

## 8.4 不要在生产中直接用原生 API

```text
原生 API 的局限性：
┌──────────────────────────────────────────────────────┐
│  1. 一次性 Watch 需要手动重新注册                     │
│  2. 连接断开后需要手动重连                             │
│  3. Session 过期后需要重建所有状态                     │
│  4. 缺少高级抽象（锁、选举、缓存等）                   │
│  5. 异常处理复杂，易出错                               │
│                                                       │
│  结论：生产环境应使用 Curator 框架                     │
│        原生 API 仅适合理解底层原理或极简场景           │
└──────────────────────────────────────────────────────┘
```
