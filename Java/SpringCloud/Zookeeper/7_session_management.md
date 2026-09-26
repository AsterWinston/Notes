
# 1. Session 概述

Session（会话）是 ZooKeeper 客户端与服务端之间的**逻辑连接**。客户端与集群中任意一个节点建立 TCP 连接后，会创建一个 Session。Session 是 ZooKeeper 许多核心功能的基础：

```text
Session 在 ZooKeeper 中的核心作用：
+---------------------------------------------------------------+
|                                                                |
|  临时节点归属：临时节点与 Session 绑定                           |
|  Watch 管理：Watch 注册在 Session 级别                          |
|  请求顺序保证：同一 Session 内的操作 FIFO 有序                   |
|  权限上下文：客户端认证信息附着在 Session 上                     |
|  连接重连：TCP 断开后可通过 sessionId 重新建立                   |
|                                                                |
+---------------------------------------------------------------+
```

# 2. Session 创建流程

```text
Client                            Server (任意节点)
  |                                         |
  | 1. 建立 TCP 连接                         |
  +--------------------------------------->|
  |                                         |
  | 2. 发送 ConnectRequest                  |
  |    protocolVersion                      |
  |    lastZxidSeen                         |
  |    timeOut (客户端期望的超时时间)         |
  |    sessionId (新连接为 0)                |
  |    passwd (session 密码)                |
  +--------------------------------------->|
  |                                         |
  |                               3. 服务端处理：
  |                                  若 sessionId == 0：
  |                                   创建新 Session
  |                                   生成唯一的 sessionId
  |                                   生成 session 密码
  |                                   计算协商的 sessionTimeout
  |                                  若 sessionId != 0：
  |                                   重连已有 Session
  |                                   验证密码
  |                                   重置超时计时器
  |                                         |
  | 4. 返回 ConnectResponse                 |
  |    sessionId                            |
  |    passwd                               |
  |    negotiatedTimeout                    |
  |<----------------------------------------|
  |                                         |
  | 5. 进入 CONNECTED 状态                   |
  |    开始正常通信                          |
```

# 3. Session ID 生成

```java
// SessionTrackerImpl 中的 sessionId 生成逻辑（简化）
public class SessionTrackerImpl {
    private long nextSessionId;

    public long createSession(int sessionTimeout) {
        long sessionId = initializeNextSessionId();
        return sessionId;
    }

    // sessionId 格式：
    // 高 8 位：serverId（用于标识创建 Session 的节点）
    // 低 56 位：自增序号
    //
    // 为什么需要 serverId？
    //  避免不同节点生成相同 sessionId
    //  支持 Session 迁移：知道原属节点可查询 Session 状态
}
```

# 4. Session Timeout 机制

## 4.1 协商超时时间

客户端提议的 timeout 与服务端限制之间的协商：

```text
timeout 协商规则：
+------------------------------------------------------------------+
|                                                                   |
|  negotiatedTimeout = clamp(                                        |
|      客户端提议的 timeout,                                         |
|      minSessionTimeout,   // 默认 2 * tickTime = 4s              |
|      maxSessionTimeout    // 默认 20 * tickTime = 40s             |
|  )                                                                |
|                                                                   |
|  示例：                                                           |
|    客户端请求 timeout = 30s                                       |
|    min = 4s, max = 40s                                           |
|    协商结果：30s（在范围内）                                      |
|                                                                   |
|    客户端请求 timeout = 50s                                       |
|    协商结果：maxSessionTimeout = 40s（超上限截断）                |
|                                                                   |
|    客户端请求 timeout = 1s                                        |
|    协商结果：minSessionTimeout = 4s（低于下限截断）               |
|                                                                   |
+------------------------------------------------------------------+
```

## 4.2 心跳与保活

```text
心跳机制：
+------------------------------------------------------------------------+
|                                                                        |
|  Client                         Server                                  |
|    |                              |                                     |
|    +--- PING -------------------->| 更新 session 活跃时间                |
|    |                              |                                     |
|    |      每 sessionTimeout/3 发送一次 PING                              |
|    |      例如 timeout=30s --> 每 10s 发一次心跳                         |
|    |                              |                                     |
|    +--- PING -------------------->|                                     |
|    |                              |                                     |
|    +--- getData（附带自动心跳）---->| 普通请求也会更新活跃时间            |
|    |                              |                                     |
|    |           网络中断            |                                     |
|    |                              |                                     |
|    |                              | sessionTimeout 后                    |
|    |                              | 判定 Session Expired                 |
|    |                              |                                     |
|                                                                        |
|  关键机制：                                                              |
|  任何请求（CRUD）都可以作为隐式心跳，延长 session 活跃时间               |
|  心跳间隔 = sessionTimeout / 3，保证在超时前至少有 2 次机会               |
|  客户端无需额外实现心跳逻辑，zookeeper 客户端库自动完成                    |
|                                                                        |
+------------------------------------------------------------------------+
```

# 5. 服务端 Session 管理

## 5.1 分桶管理

```java
// SessionTrackerImpl 使用分桶（Bucketing）管理 Session
public class SessionTrackerImpl {
    // 分桶存储：key 为过期时间点，value 为该时间点过期的 session 集合
    private final ConcurrentHashMap<Long, SessionSet> sessionSets;

    // 每个 tickTime 检查一次下一个需要过期的桶
    private volatile long nextExpirationTime;
}
```

```text
分桶机制示意图：

时间线（每个刻度 = tickTime = 2s）：
------------------------------------------------------------------>
  0s    2s    4s    6s    8s    10s   12s   14s   16s   18s

  Server 在 tickTime 到来时，检查当前时间点对应的桶：
  +--------------------------------------------------------------+
  |                                                               |
  |  bucket[10s] = {sessionA, sessionB}    这两个 session         |
  |  bucket[12s] = {sessionC}              将在对应时间点          |
  |  bucket[14s] = {sessionD, sessionE}    过期                    |
  |  ...                                                          |
  |                                                               |
  |  当 sessionA 在 6s 时收到心跳：                                |
  |    从 bucket[10s] 移除 sessionA                               |
  |    重新计算过期时间：6s + 30s = 36s                            |
  |    将 sessionA 放入 bucket[36s]                               |
  |                                                               |
  |  Server 进程检查：每次 tickTime 到来时                         |
  |    for (session in bucket[currentTime]) {                    |
  |        session.expire();                                      |
  |      // 清理临时节点 + 清理 Watch                              |
  |    }                                                          |
  |                                                               |
  +--------------------------------------------------------------+
```

## 5.2 分桶过期算法优势

| 优势 | 说明 |
|------|------|
| **O(1) 检测** | 每次 tick 只需检查当前桶，无需遍历所有 session |
| **避免精度误差** | 心跳时间精确到 tickTime 级别即可，无需毫秒精度 |
| **避免扫表** | 传统定时器扫描所有 session 的时间复杂度 O(n)，分桶 O(k) |

# 6. Session 过期处理

```text
Session 过期后的清理流程：

1. 清理临时节点
   +------------------------------------------------------------+
   |  服务端遍历 ephemerals Map                                  |
   |    找到该 sessionId 下所有临时节点                          |
   |    逐个删除（触发 Watcher 通知）                            |
   |                                                             |
   |  例如：                                                     |
   |    ephemerals.get(sessionId)                                |
   |    ["/services/order/node1", "/locks/resource-A/_c_xxx"]   |
   |    --> 删除这两个节点                                        |
   +------------------------------------------------------------+

2. 清理 Watch
   +------------------------------------------------------------+
   |  服务端遍历 watcherTable                                    |
   |    移除该 session 注册的所有 Watcher                       |
   |                                                             |
   |  ZooKeeper 3.6+ 优化：                                      |
   |    使用 Netty 连接可以直接在连接断开时清理                  |
   |    "Who Watches the Watch" 机制防止内存泄漏                |
   +------------------------------------------------------------+

3. 通知客户端
   +------------------------------------------------------------+
   |  如果客户端在此期间重新连接：                                |
   |    服务端返回 SessionExpired 事件                          |
   |    客户端必须重新创建 session                               |
   |    临时节点已丢失，需要重新注册                              |
   +------------------------------------------------------------+
```

# 7. Session 重连与迁移

```text
Session 重连流程：

初始：Client <--> Server1

网络中断：
  Client  xxxxxx  Server1

Client 重连到 Server2：
  携带 sessionId 和 session 密码
  Server2 检查 session 是否仍然有效

Session 迁移：
  1. Server2 向 Leader 发送 REVALIDATE 请求
     询问 Leader：该 session 是否仍然有效？

  2. Leader 检查 SessionTracker
     如果 session 尚未过期 --> 返回有效
     同时 Leader 延长该 session 的超时

  3. Server2 接管 session
     客户端与 Server2 建立逻辑连接
     临时节点等所有关联关系保持不变

关键保证：
  Session 迁移期间，临时节点不会被删除
  客户端透明地切换到新 Server
  重连窗口 = sessionTimeout
  如果超时后才重连，session 已过期，临时节点已删除
```

# 8. 客户端 Session 状态机

```text
ZooKeeper 客户端状态转换：

           +----------+
           |  CLOSED   | <-- session 关闭或过期
           +----------+
                ^
                | close()
      +---------+----------+
      |                    |
      v                    |
+------------+       +-----+------+
| CONNECTING |------>| CONNECTED  |
|            |       |            |
+------+-----+       +------+-----+
       |                    |
       | 网络中断             |
       |                    |
       v                    v
+------------+       +--------------+
|  CLOSED     |       | RECONNECTING |
|  (无法重连) |       |              |
+------------+       +------+-------+
                            |
                    重连成功 |        重连失败/过期
                            |              |
                            v              v
                     +----------+  +--------------+
                     |CONNECTED |  |EXPIRED(AUTH) |
                     +----------+  +--------------+
```

```java
// ZooKeeper.States 定义
public enum States {
    CONNECTING,       // 正在建立连接
    CONNECTED,        // 已连接
    CONNECTEDREADONLY, // 已连接（只读模式）
    CLOSED,           // 已关闭
    AUTH_FAILED,      // 认证失败
    NOT_CONNECTED     // 未连接
}
```

# 9. Session 超时与连接限制

```properties
# zoo.cfg 中的连接限制配置

# 单个 IP 地址最大连接数（默认 60）
maxClientCnxns=60

# 最大 Session 超时时间（ms），默认 20 * tickTime
maxSessionTimeout=40000

# 最小 Session 超时时间（ms），默认 2 * tickTime
minSessionTimeout=4000

# 最大客户端连接数（全局，3.5+）
maxCnxns=0  # 0 表示不限制
```

```text
超时时间选择建议：
+------------------------------------------------------------------+
|                                                                   |
|  短超时（5s ~ 10s）：                                             |
|    适用：对故障检测敏感的场景（如金融交易）                       |
|    风险：GC 停顿或网络抖动容易导致误判                           |
|                                                                   |
|  中等超时（15s ~ 30s）：                                          |
|    适用：大多数微服务场景（默认推荐）                              |
|    平衡了故障检测速度和容错能力                                   |
|                                                                   |
|  长超时（60s ~ 120s）：                                           |
|    适用：网络不稳定或应用启动耗时长的场景                         |
|    风险：故障检测慢，影响可用性                                   |
|                                                                   |
|  特别注意：超时时间要大于应用 Full GC 的最大停顿时间              |
|                                                                   |
+------------------------------------------------------------------+
```

# 10. Session 容错与最佳实践

```text
应用层容错机制：

1. ConnectionLoss 处理
   当客户端抛出 ConnectionLossException：
    自动重连（ZooKeeper Client 内置）
    重新注册 Watch（Curator 自动处理）
    重新创建临时节点

2. SessionExpired 处理
   当收到 SessionExpired 事件：
    所有临时节点已丢失
    所有 Watch 已清除
    需要创建新 Session
    重新注册所有数据和 Watch

3. Curator 框架的自动重连
   Curator 的 ConnectionStateListener：
   LOST --> 创建新 Session --> RECONNECTED
    重新创建临时节点（EphemeralNode 自动处理）
    重新注册 Watch（NodeCache 等自动处理）
    分布式锁自动重新竞争（InterProcessMutex）

4. Session 超时监控
   监控 Session 过期次数
   监控重连次数和成功率
   监控临时节点残留（Session 泄露）
   配置告警：Session 过期率超过阈值
```
