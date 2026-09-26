
# 1. 分布式锁概述

在分布式系统中，当多个进程需要互斥访问共享资源时，传统的单机锁（synchronized / ReentrantLock）无法满足需求，需要**分布式锁**来保证跨 JVM 的互斥性。

```text
分布式锁的需求场景：
+----------------------------------------------------------------------+
|                                                                       |
|  1. 库存扣减：多个订单服务实例同时扣减同一商品库存                    |
|  2. 定时任务：多实例下确保同一时刻只有一个实例执行任务                |
|  3. 幂等控制：防止用户重复提交                                       |
|  4. 资源分配：分配全局唯一的资源（如订单号段）                        |
|                                                                       |
+----------------------------------------------------------------------+
```

# 2. ZK 分布式锁的实现原理

## 2.1 基础方案：临时顺序节点

ZooKeeper 实现分布式锁的核心思路是利用**临时顺序节点**：

```text
ZK 分布式锁原理：

/locks/resource-A (持久节点)
  /_c_lock-0000000000  ← Client1 创建，序号最小，获得锁
  /_c_lock-0000000001  ← Client2 创建，Watch 上一个节点
  /_c_lock-0000000002  ← Client3 创建，Watch 上一个节点

释放锁时：
  · 正常释放：Client1 主动删除 _c_lock-0000000000
  · 异常释放：Client1 断开连接，临时节点自动删除

获得锁的判断：
  创建临时顺序节点后，获取所有子节点，如果自己是最小序号则获得锁

等待锁：
  如果自己不是最小序号，则 Watch 前一个节点（避免惊群效应）
```

## 2.2 避免惊群效应（Herd Effect）

```text
错误方案（惊群效应）：
  Client 创建临时节点后，Watch 父节点的 Children 变化
  -> 锁释放时，所有等待的 Client 同时收到通知
  -> 所有 Client 同时去检查自己是否获得锁
  -> 大量无效竞争，浪费资源

正确方案（链式 Watch）：
  Client 只 Watch 自己前一个序号的节点
  -> 锁释放时，只有紧邻的下一个 Client 收到通知
  -> 只有一个 Client 被唤醒去竞争锁
  -> 零惊群效应
```

# 3. Curator 分布式锁 API

Curator 提供了完善且生产级别的分布式锁实现：

## 3.1 InterProcessMutex（可重入互斥锁）

```java
// InterProcessMutex — 基于 ZK 的可重入分布式互斥锁
@Service
public class InventoryService {
    private final CuratorFramework client;

    public void deductStock(String productId, int quantity)
            throws Exception {
        String lockPath = "/locks/inventory/" + productId;
        InterProcessMutex lock = new InterProcessMutex(client, lockPath);

        // 尝试获取锁（最多等待 10 秒）
        if (lock.acquire(10, TimeUnit.SECONDS)) {
            try {
                // 临界区：扣减库存
                int current = getStock(productId);
                if (current >= quantity) {
                    updateStock(productId, current - quantity);
                } else {
                    throw new InsufficientStockException();
                }
            } finally {
                lock.release();  // 释放锁
            }
        } else {
            throw new LockTimeoutException("Failed to acquire lock");
        }
    }
}
```

## 3.2 InterProcessReadWriteLock（读写锁）

```java
// 分布式读写锁 — 读读不互斥，读写互斥
InterProcessReadWriteLock rwLock = new InterProcessReadWriteLock(
    client, "/locks/config-rw");

// 读锁（允许并发读）
InterProcessMutex readLock = rwLock.readLock();
readLock.acquire();
try {
    String config = readConfiguration();
} finally {
    readLock.release();
}

// 写锁（互斥）
InterProcessMutex writeLock = rwLock.writeLock();
writeLock.acquire();
try {
    updateConfiguration(newConfig);
} finally {
    writeLock.release();
}
```

## 3.3 InterProcessSemaphoreMutex（不可重入互斥锁）

```java
// 不可重入互斥锁 — 同一线程不能重复获取
InterProcessSemaphoreMutex semaphore = new InterProcessSemaphoreMutex(
    client, "/locks/semaphore");

semaphore.acquire();
try {
    // 注意：此时不能再次 acquire()，否则死锁
    doWork();
} finally {
    semaphore.release();
}
```

## 3.4 InterProcessMultiLock（多锁容器）

```java
// 同时获取多个锁（原子操作）
InterProcessMutex lock1 = new InterProcessMutex(client, "/locks/a");
InterProcessMutex lock2 = new InterProcessMutex(client, "/locks/b");

InterProcessMultiLock multiLock = new InterProcessMultiLock(
    Arrays.asList(lock1, lock2));

// 同时获取所有锁（全部获取成功才返回）
multiLock.acquire(10, TimeUnit.SECONDS);
try {
    // 同时持有 lock1 和 lock2
    doWork();
} finally {
    multiLock.release();  // 释放所有锁
}
```

# 4. InterProcessMutex 源码分析

```java
// Curator InterProcessMutex 核心源码分析（简化版）
public class InterProcessMutex implements InterProcessLock {
    private final CuratorFramework client;
    private final String path;
    private final LockInternals internals;

    // acquire 核心流程
    public boolean acquire(long time, TimeUnit unit) throws Exception {
        // 1. 检查当前线程是否已持有锁（可重入）
        LockData lockData = threadData.get();
        if (lockData != null) {
            // 可重入：增加计数
            lockData.lockCount.incrementAndGet();
            return true;
        }

        // 2. 在 /locks/xxx/ 下创建临时顺序节点
        String lockPath = internals.attemptLock(time, unit);
        if (lockPath != null) {
            // 3. 成功获取锁，记录到 threadData
            LockData newLockData = new LockData(
                Thread.currentThread(), lockPath);
            threadData.set(newLockData);
            return true;
        }
        return false;
    }

    // release 核心流程
    public void release() throws Exception {
        LockData lockData = threadData.get();
        if (lockData == null) {
            throw new IllegalMonitorStateException("Not holding lock");
        }

        int newCount = lockData.lockCount.decrementAndGet();
        if (newCount == 0) {
            // 完全释放：删除临时节点
            client.delete().guaranteed().forPath(lockData.lockPath);
            threadData.remove();
        }
    }
}

// LockInternals.attemptLock 核心流程
class LockInternals {
    String attemptLock(long time, TimeUnit unit) throws Exception {
        // 1. 创建临时顺序节点
        String ourPath = driver.createsTheLock(client, path);

        // 2. 循环尝试获取锁
        while (true) {
            // 2a. 获取所有子节点，按序号排序
            List<String> children = getSortedChildren();

            // 2b. 取出序号最小的节点的名部分
            String sequenceNodeName = ourPath.substring(
                ourPath.lastIndexOf('/') + 1);

            // 2c. 判断是否自己就是最小的
            int ourIndex = children.indexOf(sequenceNodeName);
            if (ourIndex < 0) {
                throw new NoNodeException("Node not found");
            }

            // 2d. 如果自己是最小的 --> 获得锁
            if (ourIndex == 0) {
                return ourPath;
            }

            // 2e. 否则，Watch 前一个节点并等待
            String previousNode = children.get(ourIndex - 1);
            String pathToWatch = path + "/" + previousNode;

            // 使用 CountDownLatch 等待 Watch 通知
            CountDownLatch latch = new CountDownLatch(1);
            Watcher watcher = event -> latch.countDown();

            if (client.getData().usingWatcher(watcher)
                    .forPath(pathToWatch) != null) {
                // 前一个节点存在，等待它释放
                latch.await(time, unit);
            }
            // 重新进入循环检查
        }
    }
}
```

# 5. 分布式锁的可靠性保障

## 5.1 防死锁

```text
ZK 分布式锁防死锁机制：
+----------------------------------------------------------------------+
|                                                                       |
|  1. 临时节点自动清理                                                  |
|     持有锁的 Client 崩溃后，Session 超时，临时节点自动删除            |
|     下一个等待的 Client 自动获得锁                                    |
|                                                                       |
|  2. 可重入性                                                          |
|     同一线程可多次获取同一把锁，通过锁计数保证释放正确                 |
|                                                                       |
|  3. 锁路径隔离                                                        |
|     每个业务资源使用独立的 lock path，不同资源互不影响                 |
|                                                                       |
+----------------------------------------------------------------------+
```

## 5.2 容错处理

```java
// 锁的容错使用模式
public <T> T executeWithLock(String lockPath,
                              LockedTask<T> task) throws Exception {
    InterProcessMutex lock = new InterProcessMutex(client, lockPath);
    int retryCount = 3;

    for (int i = 0; i < retryCount; i++) {
        try {
            if (lock.acquire(10, TimeUnit.SECONDS)) {
                try {
                    return task.execute();
                } finally {
                    lock.release();
                }
            }
        } catch (Exception e) {
            if (e instanceof ConnectionLossException && i < retryCount - 1) {
                // 连接丢失，但临时节点可能仍然存在
                // 等待 Session 过期后重试
                Thread.sleep(1000 * (long) Math.pow(2, i));
                continue;
            }
            throw e;
        }
    }
    throw new LockAcquireException("Failed after " + retryCount + " retries");
}

@FunctionalInterface
interface LockedTask<T> {
    T execute() throws Exception;
}
```

# 6. 分布式锁对比分析

| 方案 | 原理 | 优点 | 缺点 | 适用场景 |
|------|------|------|------|---------|
| **ZK 临时顺序节点** | 序号最小者获得锁 | 自动释放、无惊群、可重入 | 写性能中等 | 通用场景 |
| **Redis SETNX** | SET if Not eXists | 高性能、简单 | 锁过期问题、续期复杂 | 高并发、短锁 |
| **Redisson** | Redis + Lua 脚本 | WatchDog 自动续期 | Redis 单点风险 | Redis 集群环境 |
| **etcd** | Raft + Lease | 强一致、自动续期 | 生态不如 ZK 成熟 | Kubernetes 环境 |
| **数据库悲观锁** | SELECT FOR UPDATE | 简单、无需额外组件 | 性能差、无自动释放 | 遗留系统 |

# 7. 性能优化

```text
ZK 分布式锁性能优化建议：
+----------------------------------------------------------------------+
|                                                                       |
|  1. 锁粒度控制                                                        |
|     ✓ 按资源 ID 加锁：/locks/inventory/{productId}                   |
|     ✗ 全局一把大锁：/locks/global-lock                               |
|                                                                       |
|  2. 缩短锁持有时间                                                    |
|     锁内只做必要的互斥操作                                            |
|     I/O 操作、外部调用放在锁外                                        |
|                                                                       |
|  3. 合理设置等待超时                                                  |
|     避免无限等待导致线程耗尽                                          |
|     配合业务降级策略                                                  |
|                                                                       |
|  4. 连接池复用                                                        |
|     CuratorFramework 是线程安全的，全局使用一个实例                    |
|     避免每个加锁操作创建新连接                                        |
|                                                                       |
+----------------------------------------------------------------------+
```

```java
// 错误示例：锁粒度过粗
@Lock("/locks/global-order-lock")  // 所有订单操作竞争同一把锁
public void createOrder(OrderDTO dto) { ... }

// 正确示例：细粒度锁
public void createOrder(OrderDTO dto) {
    String lockPath = "/locks/order/" + dto.getUserId();
    InterProcessMutex lock = new InterProcessMutex(client, lockPath);
    // 按用户 ID 隔离，不同用户的请求不互相阻塞
}
```

# 8. 监控与运维

```text
分布式锁的监控指标：
+----------------------------------------------------------------------+
|                                                                       |
|  1. 锁获取成功率：acquire 成功次数 / 总尝试次数                       |
|  2. 锁平均等待时间：从开始等待到获得锁的时间                           |
|  3. 锁平均持有时间：从获得锁到释放的时间                               |
|  4. 锁超时失败率：因超时未获得锁的占比                                 |
|  5. 锁竞争度：同时等待同一把锁的线程数                                 |
|  6. 锁路径残留：因 Session 未正常关闭导致的临时节点残留               |
|                                                                       |
+----------------------------------------------------------------------+
```

```java
// 锁使用指标埋点示例
@Component
public class MonitoredLock {
    private final MeterRegistry meterRegistry;
    private final CuratorFramework client;

    public <T> T executeWithMetrics(String lockPath,
                                    LockedTask<T> task) throws Exception {
        InterProcessMutex lock = new InterProcessMutex(client, lockPath);
        Timer.Sample sample = Timer.start(meterRegistry);

        try {
            if (lock.acquire(10, TimeUnit.SECONDS)) {
                Counter.builder("lock.acquire.success")
                    .tag("path", lockPath)
                    .register(meterRegistry).increment();
                try {
                    return task.execute();
                } finally {
                    lock.release();
                    sample.stop(Timer.builder("lock.hold.time")
                        .tag("path", lockPath)
                        .register(meterRegistry));
                }
            } else {
                Counter.builder("lock.acquire.timeout")
                    .tag("path", lockPath)
                    .register(meterRegistry).increment();
                throw new LockTimeoutException();
            }
        } catch (Exception e) {
            Counter.builder("lock.acquire.failure")
                .tag("path", lockPath)
                .register(meterRegistry).increment();
            throw e;
        }
    }
}
```
