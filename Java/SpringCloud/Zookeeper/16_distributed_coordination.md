
# 1. 概述

除了分布式锁，ZooKeeper 还可用于构建各类**分布式协调原语**（Coordination Primitives）。Curator Recipes 模块提供了丰富的开箱即用实现。

```text
ZK 分布式协调原语全景：
+----------------------------------------------------------------------+
|                                                                       |
|  分布式锁         分布式选举        分布式屏障                        |
|  InterProcessMutex LeaderLatch      DistributedBarrier                |
|  InterProcessRWLock LeaderSelector  DistributedDoubleBarrier           |
|  InterProcessSemaphore                                                |
|                                                                       |
|  分布式计数器      分布式队列        分布式缓存                        |
|  DistributedAtomicLong  DistributedQueue     NodeCache                |
|  DistributedAtomicInt   DistributedPriorityQueue PathChildrenCache     |
|  SharedCount            DistributedDelayQueue  TreeCache              |
|                                                                       |
+----------------------------------------------------------------------+
```

# 2. Leader 选举

## 2.1 LeaderLatch

```java
// LeaderLatch：简单、持续的 Leader 选举
// 所有参与者竞争同一个路径，只有一个是 Leader
public class SchedulerCluster {
    private final LeaderLatch leaderLatch;

    @PostConstruct
    public void init() throws Exception {
        leaderLatch = new LeaderLatch(client, "/election/scheduler");

        leaderLatch.addListener(new LeaderLatchListener() {
            @Override
            public void isLeader() {
                System.out.println("I am the leader, starting scheduler");
                startScheduledTasks();
            }

            @Override
            public void notLeader() {
                System.out.println("I am not the leader, stopping scheduler");
                stopScheduledTasks();
            }
        });
        leaderLatch.start();
    }

    @PreDestroy
    public void destroy() throws Exception {
        leaderLatch.close();
    }

    // 检查当前节点是否是 Leader
    public boolean isLeader() {
        return leaderLatch.hasLeadership();
    }

    // 获取当前 Leader 的 ID
    public String getLeaderId() throws Exception {
        return leaderLatch.getLeader().getId();
    }

    // 获取所有参与者
    public List<Participant> getParticipants() throws Exception {
        return leaderLatch.getParticipants();
    }
}
```

```text
LeaderLatch 内部实现：
+----------------------------------------------------------------------+
|                                                                       |
|  底层使用临时顺序节点：                                               |
|  /election/scheduler                                                  |
|    /_c_0000000000  ← Leader（序号最小）                               |
|    /_c_0000000001  ← 等待者                                           |
|    /_c_0000000002  ← 等待者                                           |
|                                                                       |
|  Leader 断开连接 -> 临时节点删除 -> 下一个获得 Leader                  |
|  与分布式锁的链式 Watch 不同，所有等待者 Watch 父节点                 |
|                                                                       |
+----------------------------------------------------------------------+
```

## 2.2 LeaderSelector

```java
// LeaderSelector：带任务执行的 Leader 选举
// Leader 执行 takeLeadership 方法，执行完后 release 触发重新选举
public class TaskExecutor {
    private final LeaderSelector leaderSelector;

    public void start() {
        leaderSelector = new LeaderSelector(
            client,
            "/election/task-executor",
            new LeaderSelectorListenerAdapter() {
                @Override
                public void takeLeadership(CuratorFramework client)
                        throws Exception {
                    System.out.println("Taking leadership, processing tasks...");

                    // 只有 Leader 执行此方法
                    while (true) {
                        Task task = fetchTaskFromQueue();
                        if (task != null) {
                            processTask(task);
                        } else {
                            // 无任务时短暂休眠，避免忙等
                            Thread.sleep(1000);
                        }
                    }
                    // 方法退出后释放 leadership，触发重新选举
                }
            }
        );

        // 自动重新参与选举（takeLeadership 退出后）
        leaderSelector.autoRequeue();
        leaderSelector.start();
    }

    public void stop() {
        leaderSelector.close();
    }
}
```

# 3. 分布式 Barrier（栅栏）

## 3.1 DistributedBarrier

```java
// DistributedBarrier：所有参与者到达后才一起继续
public class ParallelTaskRunner {
    private final CuratorFramework client;

    public void runTask(String barrierPath, int participantCount)
            throws Exception {
        DistributedBarrier barrier = new DistributedBarrier(
            client, barrierPath);

        // 主控方：设置栅栏
        barrier.setBarrier();

        // 启动多个参与者线程（每个线程代表一个分布式节点）
        CountDownLatch startLatch = new CountDownLatch(1);
        for (int i = 0; i < participantCount; i++) {
            String nodeId = "node-" + i;
            new Thread(() -> {
                try {
                    System.out.println(nodeId + " ready, waiting...");
                    // 参与者：等待其他参与者就绪
                    barrier.waitOnBarrier();
                    System.out.println(nodeId + " start working!");
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }).start();
        }

        // 等待所有参与者就绪后，移除栅栏
        Thread.sleep(3000);  // 等待参与者注册
        System.out.println("All participants ready, releasing barrier");
        barrier.removeBarrier();
    }
}
```

## 3.2 DistributedDoubleBarrier

```java
// DistributedDoubleBarrier：进入和退出两个阶段都同步
public class BatchProcessor {
    public void processBatch(int memberQty) throws Exception {
        DistributedDoubleBarrier doubleBarrier =
            new DistributedDoubleBarrier(
                client,
                "/barriers/batch",
                memberQty  // 需要多少成员
            );

        // 阶段 1：等待所有成员进入
        System.out.println("Waiting for all members to enter...");
        doubleBarrier.enter();
        System.out.println("All members entered, doing work...");

        // 执行批处理任务
        doBatchWork();

        // 阶段 2：等待所有成员完成
        System.out.println("Work done, waiting for all to leave...");
        doubleBarrier.leave();
        System.out.println("All members left");
    }
}
```

# 4. 分布式计数器

## 4.1 SharedCount

```java
// SharedCount：基于版本号的分布式计数器
// 使用乐观锁（CAS）更新，适合读多写少的场景
public class SharedCounterExample {
    public void runSharedCount() throws Exception {
        SharedCount sharedCount = new SharedCount(client, "/counters/order", 0);
        sharedCount.start();

        // 监听计数变化
        sharedCount.addListener(new SharedCountListener() {
            @Override
            public void countHasChanged(SharedCountReader reader,
                                        int newCount) {
                System.out.println("Count changed to: " + newCount);
            }

            @Override
            public void stateChanged(CuratorFramework client,
                                     ConnectionState newState) {
                // 连接状态变化处理
            }
        });

        // 更新计数（CAS 循环重试）
        while (true) {
            VersionedValue<Integer> current = sharedCount.getVersionedValue();
            int newValue = current.getValue() + 1;
            if (sharedCount.trySetCount(current, newValue)) {
                System.out.println("Updated to: " + newValue);
                break;
            }
            // CAS 失败，自动重试
        }

        sharedCount.close();
    }
}
```

## 4.2 DistributedAtomicLong

```java
// DistributedAtomicLong：原子长整型操作
// 内部使用乐观锁 + 重试机制
public class IdGenerator {
    private final DistributedAtomicLong atomicLong;

    public IdGenerator(CuratorFramework client) {
        this.atomicLong = new DistributedAtomicLong(
            client,
            "/counters/order-id",
            new RetryNTimes(5, 100)  // 最多重试 5 次
        );
    }

    // 生成下一个 ID
    public Long nextId() throws Exception {
        AtomicValue<Long> result = atomicLong.increment();
        if (result.succeeded()) {
            return result.postValue();
        }
        throw new RuntimeException("Failed to generate ID: " +
            result.getStats());
    }

    // 批量获取 ID
    public LongRange nextIdRange(int count) throws Exception {
        for (int i = 0; i < 3; i++) {
            AtomicValue<Long> current = atomicLong.get();
            if (!current.succeeded()) continue;

            long startId = current.postValue();
            long endId = startId + count - 1;

            if (atomicLong.compareAndSet(startId, endId + 1).succeeded()) {
                return new LongRange(startId, endId);
            }
        }
        throw new RuntimeException("Failed to acquire ID range");
    }
}
```

# 5. 分布式队列

## 5.1 DistributedQueue

```java
// DistributedQueue：基于 ZK 的 FIFO 分布式队列
// 使用持久顺序节点实现
public class TaskQueueExample {
    private DistributedQueue<String> queue;

    public void init(CuratorFramework client) throws Exception {
        QueueBuilder<String> builder = QueueBuilder.builder(
            client,
            consumer,           // 消费者回调
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
            "/queues/tasks"     // 队列路径
        );

        queue = builder.buildQueue();
        queue.start();
    }

    // 生产者：放入任务
    public void produce(String task) throws Exception {
        queue.put(task);
        System.out.println("Produced: " + task);
    }

    // 消费者：取出任务
    private final QueueConsumer<String> consumer =
        new QueueConsumer<String>() {
        @Override
        public void consumeMessage(String message) throws Exception {
            System.out.println("Consumed: " + message);
            processTask(message);
        }

        @Override
        public void stateChanged(CuratorFramework client,
                                 ConnectionState newState) {
            System.out.println("Queue consumer state: " + newState);
        }
    };
}
```

## 5.2 SimpleDistributedQueue

```java
// SimpleDistributedQueue：轻量级分布式队列
// 适合简单场景，不支持消费者回调
SimpleDistributedQueue queue = new SimpleDistributedQueue(
    client, "/queues/simple");

// 生产者
queue.offer("task-data".getBytes());

// 消费者
byte[] data = queue.take();   // 阻塞获取
byte[] data = queue.poll(5, TimeUnit.SECONDS);  // 超时获取
```

# 6. 分布式信号量

```java
// InterProcessSemaphoreV2：分布式信号量（控制并发数）
public class RateLimiter {
    private final InterProcessSemaphoreV2 semaphore;

    public RateLimiter(CuratorFramework client, int maxLeases) {
        // 创建信号量，最多允许 maxLeases 个许可同时持有
        this.semaphore = new InterProcessSemaphoreV2(
            client,
            "/semaphores/api-calls",
            maxLeases  // 最大许可数
        );
    }

    // 获取许可（阻塞等待）
    public Lease acquire() throws Exception {
        return semaphore.acquire();
    }

    // 获取多个许可
    public Collection<Lease> acquire(int qty) throws Exception {
        return semaphore.acquire(qty);
    }

    // 使用示例
    public void rateLimitedCall() throws Exception {
        Lease lease = semaphore.acquire(5, TimeUnit.SECONDS);
        if (lease != null) {
            try {
                doApiCall();
            } finally {
                semaphore.returnLease(lease);
            }
        } else {
            throw new RateLimitExceededException();
        }
    }
}
```

# 7. 分布式缓存（Cache）

Curator 提供的 Cache 实现详见 [11_curator_framework.md](./11_curator_framework.md)，这里做简要对照：

| Cache 类型 | 作用 | 关键 API |
|-----------|------|---------|
| **NodeCache** | 缓存单个节点的数据和状态 | `getCurrentData()`, `getListenable()` |
| **PathChildrenCache** | 缓存子节点列表及其数据 | `getCurrentData()`, `getListenable()` |
| **TreeCache** | 递归缓存整个子树 | `getCurrentChildren()`, `getCurrentData()` |

# 8. 分布式 ID 生成器

```java
// 基于 ZK 顺序节点的全局唯一 ID 生成
public class ZkIdGenerator {
    private final CuratorFramework client;
    private static final String ID_PATH = "/id-generators";

    public long generateId(String namespace) throws Exception {
        String path = ID_PATH + "/" + namespace + "/id-";

        // 使用持久顺序节点生成唯一 ID
        String createdPath = client.create()
            .creatingParentsIfNeeded()
            .withMode(CreateMode.PERSISTENT_SEQUENTIAL)
            .forPath(path);

        // 提取序号
        String seqPart = createdPath.substring(
            createdPath.lastIndexOf('-') + 1);

        // 清理旧节点（可选，避免节点数量增长）
        if (Long.parseLong(seqPart) % 100 == 0) {
            client.delete().forPath(createdPath);
        }

        return Long.parseLong(seqPart);
    }
}
```

# 9. 最佳实践总结

```text
分布式协调原语使用建议：

1. 选型决策
   +----------------------------------------------------------------+
   |  需要互斥访问共享资源 --> InterProcessMutex                     |
   |  需要读多写少的互斥 --> InterProcessReadWriteLock                |
   |  需要选出一个执行者 --> LeaderLatch / LeaderSelector             |
   |  需要多节点同步 --> DistributedBarrier / DoubleBarrier           |
   |  需要全局计数器 --> SharedCount / DistributedAtomicLong          |
   |  需要任务队列 --> DistributedQueue                               |
   |  需要限流 --> InterProcessSemaphoreV2                            |
   +----------------------------------------------------------------+

2. 资源清理
   +----------------------------------------------------------------+
   |  确保在应用关闭时调用 close() 清理资源                          |
   |  LeaderLatch.close()、LeaderSelector.close()                    |
   |  DistributedQueue.close()                                       |
   |  使用 Spring 的 @PreDestroy 或 ShutdownHook                      |
   +----------------------------------------------------------------+

3. 路径规范
   +----------------------------------------------------------------+
   |  使用统一的路径前缀 + namespace 隔离                            |
   |  /election/{service-name}                                       |
   |  /locks/{resource-type}/{resource-id}                           |
   |  /queues/{queue-name}                                           |
   |  /counters/{counter-name}                                       |
   +----------------------------------------------------------------+

4. 异常处理
   +----------------------------------------------------------------+
   |  所有协调原语都可能抛出异常                                    |
   |  特别是 CONNECTIONLOSS 和 SESSIONEXPIRED                        |
   |  使用 Curator 的自动重连机制                                    |
   |  监听 ConnectionState 变化并做相应处理                          |
   +----------------------------------------------------------------+
```
