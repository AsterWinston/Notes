
# 1. 层次命名空间

ZooKeeper 的数据模型类似 Unix 文件系统，采用树形层次结构。每个节点称为 **ZNode**（ZooKeeper Data Node）。

```text
ZNode 树结构示例：

                      / (根节点)
                          │
        ┌─────────────────┼─────────────────────┐
        │                 │                      │
    /zookeeper        /services              /config
    (系统保留)             │                      │
                ┌─────────┼─────────┐    ┌───────┼───────┐
                │         │         │    │       │       │
           /order      /user     /pay  /db    /redis  /mq
                │         │
        ┌───────┤    ┌────┼────┐
        │       │    │    │    │
   /providers /consumers  /node1 /node2 /node3
        │
   ┌────┼────┐
   │    │    │
   ip1  ip2  ip3
```

**命名规范：**

- 路径由 `/` 分隔的 Unicode 字符串组成
- 路径必须是**绝对路径**（以 `/` 开头）
- 不支持相对路径、`.` 或 `..`
- 路径中不能包含 `/u0000` 空字符
- ZNode 命名可以使用除 `/u0000` 外的任意 Unicode 字符（但建议使用字母、数字、下划线、连字符）

# 2. ZNode 核心特性

每个 ZNode 有两个组成部分：

| 组成部分 | 说明 | 限制 |
|---------|------|------|
| **Data（数据）** | 节点存储的业务数据（字节数组） | 默认最大 1MB |
| **Stat（状态）** | 节点元数据（版本号、时间戳等） | 系统自动维护 |
| **ACL（权限）** | 访问控制列表 | 决定了谁可以执行什么操作 |
| **Children（子节点）** | 子 ZNode 列表 | 子节点无数量限制 |

```text
ZNode 结构示意：
┌─────────────────────────────────────────┐
│  ZNode: /config/db                      │
│                                          │
│  ┌─────────────────────────────────┐    │
│  │ Data (byte[])                    │    │
│  │ "jdbc:mysql://localhost:3306/db" │    │
│  └─────────────────────────────────┘    │
│                                          │
│  ┌─────────────────────────────────┐    │
│  │ Stat（元数据）                    │    │
│  │ · czxid: 0x100000002             │    │
│  │ · mzxid: 0x100000005             │    │
│  │ · ctime: 2024-01-15 10:30:00    │    │
│  │ · mtime: 2024-01-15 14:20:00    │    │
│  │ · version: 3                     │    │
│  │ · cversion: 0                    │    │
│  │ · aversion: 1                    │    │
│  │ · ephemeralOwner: 0              │    │
│  │ · dataLength: 36                 │    │
│  │ · numChildren: 0                 │    │
│  │ · pzxid: 0x100000002             │    │
│  └─────────────────────────────────┘    │
│                                          │
│  ┌─────────────────────────────────┐    │
│  │ ACL List                        │    │
│  │ [world:anyone:cdrwa]            │    │
│  └─────────────────────────────────┘    │
│                                          │
│  ┌─────────────────────────────────┐    │
│  │ Children（子节点）               │    │
│  │ [] (空)                         │    │
│  └─────────────────────────────────┘    │
└─────────────────────────────────────────┘
```

# 3. ZNode 类型详解

ZooKeeper 有四种 ZNode 类型，通过组合**持久性**和**顺序性**两个维度产生：

| 类型 | 持久性 | 顺序性 | 典型场景 |
|------|--------|--------|---------|
| **PERSISTENT** | 持久 | 无序 | 配置信息、服务地址 |
| **PERSISTENT_SEQUENTIAL** | 持久 | 顺序 | 唯一 ID 生成、分布式队列 |
| **EPHEMERAL** | 临时 | 无序 | 服务注册、选主 |
| **EPHEMERAL_SEQUENTIAL** | 临时 | 顺序 | 分布式锁、Leader 选举 |

## 3.1 持久节点（PERSISTENT）

```text
特性：
· 创建后一直存在，直到被显式删除
· 不依赖创建者的 Session 生命周期
· 适用于存储持久化配置信息

示例：
┌───────────────────────────────────┐
│  Session A 创建 /config/db        │
│  Session A 断开                    │
│  /config/db 依然存在 ✓            │
│  只有 Session B 执行 delete 才删除 │
└───────────────────────────────────┘
```

## 3.2 临时节点（EPHEMERAL）

```text
特性：
· 与创建者的 Session 绑定
· 创建者 Session 结束（断开/过期）时自动删除
· 不能拥有子节点（ZooKeeper 硬性限制）
· 适用于服务注册和心跳检测

Session 生命周期与临时节点的关系：
┌──────────────────────────────────────────────────────┐
│                                                       │
│  Client A                  ZooKeeper                   │
│    │                          │                        │
│    │── create -e /service/a ─►│ /service/a 创建        │
│    │                          │                        │
│    │─ Session 正常断开 ────────│ /service/a 删除        │
│    │                          │                        │
│    │─ Session 超时 ───────────│ /service/a 删除        │
│    │                          │                        │
│    │─ Client 进程崩溃 ────────│ (超时后) /service/a 删除│
│                                                       │
└──────────────────────────────────────────────────────┘

重要限制：
· 临时节点不能有子节点（任何试图在临时节点下创建子节点的操作都会失败）
· 临时节点在转换为持久节点前必须删除重建（不支持直接转换）
```

## 3.3 持久顺序节点（PERSISTENT_SEQUENTIAL）

```text
特性：
· 创建时 ZooKeeper 自动在名称后附加一个单调递增的 10 位数字编号
· 编号填充前导零（如 0000000000）
· 编号由父节点的 cversion 计数器维护
· 适用于需要全局唯一 ID 的场景

示例：
create -s /tasks/task- "data"
→ /tasks/task-0000000000
create -s /tasks/task- "data"
→ /tasks/task-0000000001
create -s /tasks/task- "data"
→ /tasks/task-0000000002
```

## 3.4 临时顺序节点（EPHEMERAL_SEQUENTIAL）

```text
特性：
· 结合临时节点和顺序节点的所有特性
· Session 结束自动删除 + 创建时自动编号
· 是 Curator 分布式锁和 Leader 选举的核心基础设施

典型用法 — 分布式锁：
/locks/resource-A/
    ├── _c_0000000000 ← Client1 创建（序号最小，获得锁）
    ├── _c_0000000001 ← Client2 创建（Watch 前一个节点）
    └── _c_0000000002 ← Client3 创建（Watch 前一个节点）
```

## 3.5 扩展类型（ZooKeeper 3.5+）

ZooKeeper 3.5 引入了 `CreateMode` 的扩展类型，需要在配置中显式开启后才能使用：

```properties
# zoo.cfg 中开启扩展类型
extendedTypesEnabled=true
```

| 扩展类型 | 说明 |
|---------|------|
| **PERSISTENT_WITH_TTL** | 带 TTL 的持久节点，到期后自动删除 |
| **PERSISTENT_SEQUENTIAL_WITH_TTL** | 带 TTL 的持久顺序节点 |

```java
// Java 示例：创建 5 秒后自动删除的节点
client.create()
    .withMode(CreateMode.PERSISTENT_WITH_TTL)
    .withTTL(5000)  // 5 秒
    .forPath("/ttl_node", "data".getBytes());
```

# 4. Stat 结构详解

Stat（状态信息）是 ZNode 的元数据，记录了节点的所有统计信息：

```java
// ZooKeeper 源码中的 Stat 类定义（简化）
public class Stat {
    private long czxid;      // 创建该 ZNode 的事务 ID
    private long mzxid;      // 最后一次修改该 ZNode 的事务 ID
    private long ctime;      // 创建时间（毫秒时间戳）
    private long mtime;      // 最后一次修改时间
    private int version;     // 数据版本号（每次 setData 递增）
    private int cversion;    // 子节点版本号（子节点增删时递增）
    private int aversion;    // ACL 版本号（ACL 变更时递增）
    private long ephemeralOwner; // 临时节点归属的 sessionId（持久节点为 0）
    private int dataLength;  // 数据长度（字节数）
    private int numChildren; // 子节点数量
    private long pzxid;      // 最后一次修改子节点列表的事务 ID
}
```

**字段详解**：

| 字段 | 含义 | 影响的操作 | 示例值 |
|------|------|-----------|--------|
| `czxid` | Create ZXID，创建该节点的事务 ID | 创建节点时设置 | 0x100000001 |
| `mzxid` | Modify ZXID，最后修改节点数据的事务 ID | setData 时更新 | 0x100000005 |
| `ctime` | Create Time，创建时间（Unix 毫秒） | 创建节点时设置 | 1705298400000 |
| `mtime` | Modify Time，最后修改时间 | setData 时更新 | 1705329000000 |
| `version` | 数据版本号，从 0 开始 | setData 时递增 | 3 |
| `cversion` | 子节点列表版本号 | 增删子节点时递增 | 5 |
| `aversion` | ACL 版本号 | setACL 时递增 | 1 |
| `ephemeralOwner` | 临时节点属主 sessionId | 创建时设置；0 表示持久节点 | 0x100012345678900 |
| `dataLength` | 数据内容的字节长度 | 写入数据时更新 | 127 |
| `numChildren` | 直接子节点数量 | 增删子节点时更新 | 3 |
| `pzxid` | 最后修改子节点列表的事务 ID | 增删子节点时更新 | 0x10000000A |

## 4.1 版本号的 CAS 操作

ZooKeeper 使用版本号实现**乐观锁（Optimistic Locking）**，类似数据库的 CAS（Compare-And-Swap）：

```java
// 原生 API 中传入版本号
Stat stat = new Stat();
zk.getData("/config", false, stat);

// 读取当前版本：stat.getVersion()
// 期望版本为当前版本，如果中间被其他客户端修改过则失败
Stat newStat = zk.setData("/config", newData, stat.getVersion());
// 如果 version 不匹配，抛出 KeeperException.BadVersionException
```

```text
CAS 版本控制流程：
┌─────────────────────────────────────────────────────┐
│                                                      │
│  Client A                    Client B                 │
│    │                            │                     │
│    │─ getData → version=1 ──────│── getData → v=1    │
│    │                            │                     │
│    │─ setData(v=1) ✓ ──────────│                     │
│    │  version → 2               │                     │
│    │                            │                     │
│    │                     ┌──────│── setData(v=1) ✗   │
│    │                     │      │   BadVersion!       │
│    │                     │      │                     │
│    │                     ▼      │                     │
│    │                  getData → v=2                   │
│    │                  setData(v=2) ✓                  │
└─────────────────────────────────────────────────────┘
```

# 5. 操作原语（API 基础操作）

ZooKeeper 提供了一套精心设计的、简单的操作原语：

| 操作 | 类型 | 说明 | 幂等性 |
|------|------|------|--------|
| `create` | 写 | 创建 ZNode（可指定模式、ACL） | 否（节点已存在则失败） |
| `delete` | 写 | 删除 ZNode（可指定版本） | 是（指定版本） |
| `exists` | 读 | 检查 ZNode 是否存在（可设置 Watch） | 是 |
| `getData` | 读 | 读取 ZNode 数据（可设置 Watch） | 是 |
| `setData` | 写 | 设置 ZNode 数据（可指定版本） | 是（指定版本） |
| `getChildren` | 读 | 获取子节点列表（可设置 Watch） | 是 |
| `sync` | 读 | 同步操作（等待本地数据同步到 Leader） | 是 |
| `multi` | 写 | 原子地执行多个操作（3.4+ 引入） | — |
| `getAcl` | 读 | 获取节点的 ACL 列表 | 是 |
| `setAcl` | 写 | 设置节点的 ACL 列表（可指定版本） | 是（指定版本） |

## 5.1 同步（sync）操作的意义

由于 ZooKeeper 是最终一致性模型（Follower 可能有小延迟），当客户端从 Follower 读取时，可能读到的是稍微旧的数据。`sync` 操作强制 Follower 追赶 Leader 的最新状态。

```text
sync 操作解决的问题：

Client                      Follower                       Leader
  │                           │                              │
  │──── setData("/a", "v2") ──│───────── forward ──────────►│ ✓ 提交
  │                           │                              │
  │──── getData("/a") ───────►│ 返回 "v1"（尚未同步）         │
  │                           │                              │
  │──── sync("/a") ──────────►│──── sync request ───────────►│
  │                           │ 等待所有 pending 事务完成      │
  │                           │                              │
  │──── getData("/a") ───────►│ 返回 "v2" ✓                  │
```

## 5.2 multi 操作（原子批量操作）

ZooKeeper 3.4.0 引入了 `multi` 操作，允许原子地执行多个操作：

```java
// 原子操作：创建 /a 并在其下创建 /a/b
List<Op> ops = Arrays.asList(
    Op.create("/a", data1, ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT),
    Op.create("/a/b", data2, ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT)
);
List<OpResult> results = zk.multi(ops);
// 要么全部成功，要么全部失败
```

```text
multi 操作保证：
┌────────────────────────────────────────────────────┐
│  · 原子性：所有操作要么全部成功，要么全部失败        │
│  · 隔离性：执行期间不受其他客户端干扰（但无锁）      │
│  · 顺序性：按 ops 列表的顺序依次执行                │
│  · 无回滚：部分成功后整体失败不会回滚已成功的操作    │
│             因此要求 multi 中的每个操作都是幂等可重试 │
└────────────────────────────────────────────────────┘
```

# 6. 数据模型的设计约束

| 约束 | 说明 | 原因 |
|------|------|------|
| 单节点数据不超过 1MB | ZNode 数据上限 1MB（默认，可配置 `jute.maxbuffer`） | 服务端完全在内存操作，避免大对象影响性能 |
| 临时节点不能有子节点 | 创建子节点会直接报错 | 临时节点随 Session 消失，子节点的处理逻辑会极复杂 |
| `/zookeeper` 为系统保留 | 存放 Quota 等内部信息，不建议业务使用 | 保留命名空间 |
| 空节点名不允许 | 不能创建名为空字符串的子节点 | API 约束 |
| 路径长度限制 | 无明确的长度限制，但过深路径影响性能 | 建议不超过 10 层 |

# 7. 数据序列化与 jute

ZooKeeper 使用自研的 **jute** 协议进行数据序列化，它在 ZooKeeper 的各个层面被使用：

```text
jute 的使用范围：
┌────────────────────────────────────────────────────┐
│  · 网络传输：客户端请求/响应的序列化                 │
│  · 事务日志：每条事务记录的持久化格式                │
│  · 快照文件：DataTree 的序列化和反序列化             │
│  · 选举消息：Leader 选举投票消息的序列化             │
└────────────────────────────────────────────────────┘

jute 的主要类型映射：
┌──────────────────────────────────────────────────────┐
│  Java 类型         jute 序列化方式                     │
│  byte[]            → 先写长度（4字节int）+ 再写数据    │
│  String            → UTF-8 编码后序列化                │
│  int/long/bool     → 固定长度大端序                     │
│  List/Map          → 先写长度 + 逐个序列化             │
└──────────────────────────────────────────────────────┘
```

# 8. 内存数据模型源码视角

```java
// org.apache.zookeeper.server.DataTree
public class DataTree {
    // key: 路径（如 "/config/db"），value: DataNode
    private final ConcurrentHashMap<String, DataNode> nodes =
        new ConcurrentHashMap<>();

    // Watch 管理：key 为路径，value 为 Watcher 集合
    private final WatchManager dataWatches;
    private final WatchManager childWatches;

    // 临时节点映射：key 为 sessionId，value 为该 session 创建的临时节点路径集合
    private final Map<Long, HashSet<String>> ephemerals =
        new ConcurrentHashMap<>();
}

// DataNode：内存中的 ZNode 表示
public class DataNode implements Record {
    byte[] data;           // 节点数据
    Long acl;              // ACL 引用（指向 aclMap 的 key）
    public StatPersisted stat; // 持久化部分的状态信息
    private Set<String> children = new HashSet<>(); // 子节点名称集合
}
```

```text
DataTree 内部结构示意：

nodes (ConcurrentHashMap<路径, DataNode>):
┌────────────────────────────────────────────────────┐
│  "/"          → DataNode {data:null, children:{"zk","svc"}} │
│  "/zookeeper" → DataNode {data:null, children:{"quota"}}   │
│  "/services"  → DataNode {data:null, children:{"order"}}   │
│  "/services/order" → DataNode {data:"192.168.1.1:8080"}    │
│  ...                                                  │
└────────────────────────────────────────────────────┘

ephemerals (Map<sessionId, Set<path>>):
┌────────────────────────────────────────────────────┐
│  0x10001234 → {"/services/order/node1", "/lock/001"}│
│  0x10005678 → {"/services/user/node2"}              │
│  ...                                                 │
└────────────────────────────────────────────────────┘
```

# 9. 数据模型最佳实践

## 9.1 ZNode 大小控制

```text
√ 推荐：
  · 配置项：KB 级别
  · 服务地址：几百字节
  · 简单的状态值：几十字节

✗ 避免：
  · 存储大文件内容
  · 存储 JSON 配置（超过 100KB）
  · 存储序列化对象

替代方案：
  · 大配置 → 数据库 + ZooKeeper 仅存版本号和地址
  · 大文件 → HDFS/对象存储 + ZooKeeper 存路径
```

## 9.2 ZNode 数量控制

```text
√ 推荐：
  · 带层级的合理组织（如 /services/order, /services/user）
  · 顺序节点用于分布式锁（用完即删）
  · 总数控制在 10 万级别以下

✗ 避免：
  · 百万级别的节点数量（读操作变慢、Leader 重启/选举后数据同步耗时增加）
  · 过深的树结构（超过 10 层）
```

## 9.3 命名规范建议

```text
推荐命名：
  /config/db/mysql-url
  /services/order-service/providers
  /locks/stock-lock
  /election/master

避免命名：
  /Config/db.url          ← 混用大小写
  /config/db/mysql%20url  ← 包含特殊字符
  /config/db/…            ← 过度嵌套
```
