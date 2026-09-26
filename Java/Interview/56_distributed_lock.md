# 1. 知识点
- **分布式锁定义**: 在分布式系统中，协调多个进程/线程对共享资源的互斥访问，同一时刻只有一个客户端持有锁
- **分布式锁要求**: 互斥性(任意时刻只有一个客户端持有锁)、防死锁(持有者崩溃后锁能自动释放)、可重入(同一客户端可多次获取)、高性能、高可用
- **数据库实现分布式锁**: 利用数据库唯一索引约束。INSERT一条锁记录成功即获取锁，DELETE删除即释放锁。简单但性能差，数据库是单点
- **数据库乐观锁**: 使用version字段，更新时`UPDATE table SET value=new, version=version+1 WHERE id=? AND version=?`，根据影响行数判断是否获取锁成功
- **数据库悲观锁**: 使用`SELECT ... FOR UPDATE`对行加排他锁，事务结束释放。性能差，容易死锁，不适合高并发
- **Redis分布式锁**: 使用`SET resource_name random_value NX PX expire_time`原子命令(set if not exist + 设置过期时间)
- **Redis SET NX EX原理**: NX表示key不存在时才设置成功(获取锁)，PX设置过期时间防止死锁。两步原子执行避免setnx后expire前宕机的死锁问题
- **Redis锁的value作用**: 使用随机值(UUID)作为锁的value，解锁时判断value是否匹配，防止误解锁(自己持有的锁过期后别人才获取，此时不能删别人的锁)
- **Redis解锁Lua脚本**: 先GET判断value是否匹配，匹配才DEL。Lua脚本保证原子性(GET+DEL非原子操作会导致并发问题)
- **Redisson框架**: Java Redis客户端，提供完整的分布式锁实现，包括可重入锁、公平锁、联锁、红锁、读写锁、信号量等
- **Redisson看门狗(Watchdog)**: 自动续期机制，后台线程每10秒检查锁是否仍被当前线程持有，是则将过期时间重置为30秒，防止业务执行超时锁自动释放
- **Redisson可重入锁原理**: Redis Hash结构存储，key为锁名，field为客户端连接ID(UUID+线程ID)，value为重入计数。lock加1，unlock减1，归零时删除
- **Redisson公平锁**: 多个客户端按请求时间的先后顺序排队获取锁，基于ZSet实现等待队列，防止线程饥饿现象
- **RedLock(红锁)**: Redis官方提出的多节点分布式锁算法。在N个(奇数)独立Redis节点上依次尝试获取锁，超过半数(N/2+1)成功且总耗时小于锁有效时间，才算获取锁成功
- **RedLock争议**: Martin Kleppmann等人认为RedLock基于时间假设不安全(GC停顿、时钟跳跃)，不适用于需要强一致性的场景；Redis作者Antirez反驳该观点
- **ZooKeeper分布式锁**: 利用临时顺序节点实现。所有客户端在同一路径下创建临时顺序节点，序号最小的节点获得锁，其他节点监听前一个节点的删除事件(watcher)
- **ZooKeeper锁释放**: 客户端主动删除节点释放锁，或客户端断开连接(宕机)时临时节点自动删除，避免死锁
- **ZooKeeper锁优点**: 强一致性(CP)，天然防死锁(临时节点)，可监听释放事件(避免轮询)
- **ZooKeeper锁缺点**: 性能低于Redis(需要创建和删除节点)，依赖ZooKeeper集群，吞吐量受限于ZK
- **Curator框架**: Apache Curator是ZooKeeper客户端库，提供各种分布式锁实现(InterProcessMutex可重入锁、InterProcessSemaphoreMutex不可重入锁、InterProcessReadWriteLock读写锁)
- **etcd分布式锁**: 基于etcd的Lease租约机制，类似ZK的临时节点。客户端创建key绑定Lease，定期续约，Lease过期key自动删除(释放锁)
- **Redisson vs ZK vs 数据库锁对比**: Redisson性能最高(AP)，ZK强一致性(CP)但性能低，数据库性能最差不推荐。大多数场景推荐Redisson
- **可重入锁实现**: 锁持有者再次请求锁时成功获取，需记录持有者标识和重入次数。Redis用Hash(count+clientId)，ZK用节点数据记录(clientId+count)
- **自旋锁**: 获取锁失败时不立即阻塞，而是循环等待(自旋)尝试重新获取。缺点：浪费CPU，适合锁占用时间极短的场景

# 2. 面试题

## 2.1 Redis实现分布式锁的SET NX EX为什么必须是原子操作？
使用`SET lock_key value NX EX 30`是原子操作。若分两步执行(`SETNX lock_key value` + `EXPIRE lock_key 30`)，在SETNX成功后、EXPIRE执行前服务器宕机或进程崩溃，锁没有过期时间会导致死锁，其他客户端永远无法获取该锁。原子操作保证"设置锁"和"设置过期"同时成功或同时失败。

## 2.2 Redis分布式锁的Lua脚本解锁为什么有必要？
解锁时需要：1)检查锁的value是否与自己的value匹配(防止解锁别人的锁) 2)匹配才删除。这两个操作必须是原子的。若先用GET判断再DEL，GET返回自己的value，此时锁恰巧过期，另一个客户端获取锁(set了新value)，自己的DEL就会误删别人的锁。Lua脚本在Redis服务端原子执行，保证检查与删除不被打断：
```
if redis.call('get', KEYS[1]) == ARGV[1] then
    return redis.call('del', KEYS[1])
else
    return 0
end
```

## 2.3 Redisson的看门狗(Watchdog)机制是如何工作的？
Redisson获取锁时若不指定leaseTime(过期时间)，默认启用以看门狗机制。锁默认过期时间30秒，看门狗是一个后台定时任务，每30/3=10秒检查当前线程是否还持有锁，如果是则将过期时间重置为30秒。看门狗在锁被释放或客户端实例关闭时自动停止。这解决了"业务执行时间超过锁过期时间导致锁提前释放"的问题。如果指定了leaseTime，看门狗不生效，到期自动释放。

## 2.4 什么是RedLock算法？为什么有争议？
RedLock是Redis官方提出的算法，在N个(建议5个)独立Redis节点上：
1) 客户端获取当前时间戳
2) 依次向N个Redis节点请求获取锁(SET NX PX)
3) 统计成功数量，若超过半数(N/2+1)，且总耗时 < 锁有效时间，获取成功
4) 实际有效时间 = 锁有效时间 - 总耗时
5) 失败则向所有节点发送释放请求

争议点：分布式专家Martin Kleppmann认为RedLock依赖时间同步假设，GC停顿可能导致锁超时误解锁；时钟跳跃可能导致多个客户端同时获得锁。RedLock不适合严格的互斥场景，建议使用ZooKeeper或etcd(CP系统)或Fencing Token(递增token)补偿。

## 2.5 ZooKeeper分布式锁的原理和优缺点？
原理：多个客户端在ZK的同一个路径下创建临时顺序节点(EPHEMERAL_SEQUENTIAL)，ZK保证节点序号全局递增唯一。客户端获取所有子节点，判断自己是否是最小序号，是则获得锁；否则对前一个节点设置Watcher监听。前一个节点删除时(客户端释放或宕机)，监听者收到通知，再次检查自己是否成为最小序号。

优点：强一致性(CP)、防死锁(临时节点、客户端宕机自动释放)、非阻塞(监听通知而非轮询)、可重入(节点数据记录持有者)。缺点：性能比Redis低(需创建/删除ZNode)、吞吐量受ZK集群性能限制、ZK集群自身也有性能瓶颈。

## 2.6 数据库如何实现分布式锁？有哪些方式？
1) **唯一索引法**：创建锁表(lock_name唯一索引)，`INSERT INTO lock_table(lock_name, holder) VALUES('order_lock', 'client1')`成功即获得锁，`DELETE WHERE lock_name='order_lock' AND holder='client1'`释放锁。需要定时清理过期锁。
2) **乐观锁**：`UPDATE table SET count=count-1, version=version+1 WHERE id=? AND version=?`，根据影响行数判断。适合并发冲突低的场景。
3) **悲观锁**：`SELECT * FROM lock_table WHERE lock_name=? FOR UPDATE`行级排他锁，事务结束释放。适合并发冲突高的场景但性能差。

## 2.7 如何选择合适的分布式锁？
| 场景 | 推荐方案 | 理由 |
|------|---------|------|
| 高并发、性能优先 | Redisson(Redis) | 单机Redis QPS可达10万+，看门狗自动续期 |
| 强一致性要求 | ZooKeeper/etcd | CP系统，Leader选举型场景 |
| 简单、低并发 | 数据库唯一索引 | 实现简单，无需引入中间件 |
| 极高安全级别 | ZK+Redisson混合 | 双重锁保证 |
| 临时锁、信号量 | Redisson(RSemaphore) | 丰富的分布式对象支持 |

一般推荐Redisson作为首选，CP敏感场景用ZK/etcd。

## 2.8 分布式锁的可重入性如何实现？
可重入锁允许同一线程多次获取同一把锁。Redisson实现：使用Redis Hash结构，key是锁名，field是"连接ID:线程ID"，value是重入计数。第一次lock创建Hash，value=1；同一线程再次lock，value+1；unlock时value-1，归零才真正删除锁。ZooKeeper实现：创建临时顺序节点，节点数据记录holder信息(线程ID+重入计数)，同一线程再次请求锁时在数据中增加计数。

## 2.9 如何防止分布式锁死锁？
1) **设置过期时间**：Redis SET NX EX设置自动过期；ZK临时节点连接断开自动删除
2) **自动续期**：Redisson看门狗对业务执行时间不确定的场景自动续期
3) **超时释放**：获取锁设置最大持有时间，超时强制释放(业务方处理超时逻辑)
4) **释放校验**：解锁前校验锁的value/holder，只释放自己持有的锁
5) **监控告警**：锁持有时间监控，长时间未释放触发告警人工介入
