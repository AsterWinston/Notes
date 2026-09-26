
# 1. 一致性模型选择

Nacos 是目前唯一一个可以在服务发现和配置管理场景中根据不同需求**灵活切换 CAP 模式**的注册中心。

```
          Nacos 一致性模型

  ┌─────────────────────────────────┐
  │         Nacos Server            │
  │                                 │
  │  ┌──────────┐  ┌────────────┐  │
  │  │  临时实例 │  │  持久实例   │  │
  │  │  (AP)    │  │  (CP)      │  │
  │  └─────┬────┘  └─────┬──────┘  │
  │        │              │         │
  │        ▼              ▼         │
  │  ┌──────────┐  ┌────────────┐  │
  │  │  Distro  │  │   Raft     │  │
  │  │  协议    │  │   协议      │  │
  │  └──────────┘  └────────────┘  │
  │                                 │
  └─────────────────────────────────┘
```

# 2. Distro 协议（AP 模式）

## 2.1 协议概述

Distro 协议是阿里巴巴自研的 AP（可用性 + 分区容错）协议，专为临时实例的服务发现场景设计。

**核心设计目标**：

1. **高可用**：每个节点都可以处理读写请求
2. **最终一致性**：数据异步同步，允许短暂不一致
3. **去中心化**：没有 Leader 节点，所有节点对等
4. **水平扩展**：新节点加入后自动均衡数据

## 2.2 Distro 数据分片策略

```
                     Distro 数据分片

  ┌──────────────────────────────────────────────────────┐
  │              Nacos Data Key                            │
  │  com.alibaba.nacos.naming.iplist.ephemeral.           │
  │  {namespaceId}##{serviceName}                         │
  └──────────────────────┬───────────────────────────────┘
                         │
                         ▼
                   hash(Key) % N
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
     ┌─────────┐   ┌─────────┐   ┌─────────┐
     │ Node-1  │   │ Node-2  │   │ Node-3  │
     │(Owner)  │   │(Replica)│   │(Replica)│
     └─────────┘   └─────────┘   └─────────┘
```

每个 Key 都有一个 **Owner 节点**（负责数据的读写）和若干 **Replica 节点**（备份数据）。

## 2.3 Distro 写入流程

```java
// DistroConsistencyServiceImpl.java
@Override
public void put(String key, Record value) {
    // 1. 写入本节点
    onPut(key, value);
    
    // 2. 判断本节点是否是 Owner
    String ownerServer = distroHash.getServer(key);
    
    if (!NetUtils.localServer().equals(ownerServer)) {
        // 本节点不是 Owner → 转发请求到 Owner
        distroProtocol.sync(new DistroKey(key), DataOperation.ADD, 0L);
        return;
    }
    
    // 3. 本节点是 Owner → 异步同步到其他 Replica 节点
    for (Member member : memberManager.allMembersWithoutSelf()) {
        syncToTarget(key, DataOperation.ADD, member.getAddress());
    }
    
    // 4. 直接返回成功（不需要等待 Replica ACK）
}

// 同步数据到目标节点
private void syncToTarget(String key, DataOperation action, String targetServer) {
    byte[] data = serializer.serialize(dataStore.get(key));
    
    try {
        // HTTP 异步发送
        HttpClient.asyncHttpPost(
            "http://" + targetServer + "/nacos/v1/ns/distro/datum",
            data,
            new AsyncCompletionHandler<Response>() {
                @Override
                public Response onCompleted(Response response) {
                    if (response.getStatusCode() != 200) {
                        // 同步失败 → 加入重试队列
                        distroTaskEngineHolder.getRetryTaskEngine()
                            .addTask(targetServer, new DistroTask(key, action));
                    }
                    return response;
                }
            });
    } catch (Exception e) {
        // 网络异常 → 加入重试队列
        retryQueue.add(new RetryTask(key, action, targetServer));
    }
}
```

## 2.4 Distro 反熵（Anti-Entropy）机制

```java
// DistroVerifyTask.java
@Component
public class DistroVerifyTask implements Runnable {
    
    // 定时进行数据校验（默认每 5s）
    @Override
    public void run() {
        for (Member member : memberManager.allMembersWithoutSelf()) {
            // 1. 获取远端节点的所有 Key 摘要列表
            List<String> remoteChecksumKeys = getChecksumKeysFromRemote(member);
            
            // 2. 获取本地所有 Key 摘要列表
            List<String> localChecksumKeys = getChecksumKeysFromLocal();
            
            // 3. 对比差异
            for (String key : localChecksumKeys) {
                if (shouldBeResponsibleFor(key)) {
                    // 本节点负责的 Key
                    String localChecksum = getChecksum(key);
                    String remoteChecksum = remoteChecksumMap.get(key);
                    
                    if (!localChecksum.equals(remoteChecksum)) {
                        // 数据不一致 → 全量同步
                        syncFullDataToTarget(key, member.getAddress());
                    }
                }
            }
        }
    }
    
    // 获取本地数据的 checksum
    private String getChecksum(String key) {
        Datum datum = dataStore.get(key);
        if (datum == null) return null;
        
        // 用 MD5 作为摘要
        return MD5Utils.md5Hex(datum.value.toString(), "UTF-8");
    }
}
```

## 2.5 Distro 数据同步总览

```
Client Request (写入)
     │
     ▼
Node-1 (接收请求)
     │
     ├──→ 写入本节点 DataStore
     │
     ├──→ 判断 Owner
     │     ├── 是 Owner → 直接处理
     │     └── 非 Owner → 转发到 Owner
     │
     ├──→ 异步同步到其他节点
     │     ├── 同步成功 → 完成
     │     └── 同步失败 → 加入重试队列
     │            └── 重试 N 次后失败 → 记录日志
     │
     └──→ 反熵定时任务（5s）
           └── 全量对比 → 修复差异
```

# 3. Raft 协议（CP 模式）

## 3.1 JRaft 引擎

Nacos 使用蚂蚁金服开源的 [SOFAJRaft](https://github.com/sofastack/sofa-jraft) 作为 Raft 协议引擎：

```java
// RaftCore.java (基于 SOFAJRaft)
@Component
public class RaftCore {
    
    // Raft Group Map: groupId → RaftGroupService
    private final Map<String, RaftGroupService> raftGroupMap = new ConcurrentHashMap<>();
    
    // RPC Server
    private RpcServer rpcServer;
    
    // 状态机
    private NacosStateMachine stateMachine;
    
    public void init() {
        // 1. 初始化 Raft RPC Server
        rpcServer = RaftRpcServerFactory.createRaftRpcServer(
            NetUtils.localServer() + ":" + getRaftPort());
        
        // 2. 初始化状态机
        stateMachine = new NacosStateMachine();
        
        // 3. 为每个 Group 创建 Raft 节点
        // (配置管理使用一个全局 Group，持久服务每个 namespace 一个 Group)
    }
}
```

## 3.2 NacosStateMachine 状态机

```java
// NacosStateMachine.java
public class NacosStateMachine extends StateMachineAdapter {
    
    // 持久化存储
    @Autowired
    private PersistService persistService;
    
    /**
     * 提交日志（Raft 日志应用到状态机）
     */
    @Override
    public void onApply(Iterator iter) {
        while (iter.hasNext()) {
            ByteBuffer data = iter.getData();
            LogEntry logEntry = serializer.deserialize(data.array());
            
            if (logEntry.getOperation() == Operation.WRITE) {
                // 写入操作
                Datum datum = logEntry.getDatum();
                persistService.put(datum.getKey(), datum.getValue());
            } else if (logEntry.getOperation() == Operation.DELETE) {
                // 删除操作
                persistService.remove(logEntry.getKey());
            }
            
            iter.next();
        }
    }
    
    /**
     * 创建快照（用于日志压缩）
     */
    @Override
    public void onSnapshotSave(SnapshotWriter writer, Closure done) {
        // 将当前所有数据写入快照文件
        String snapshotPath = writer.getPath();
        persistServer.dumpAllToSnapshot(snapshotPath);
        done.run(Status.OK());
    }
    
    /**
     * 加载快照（节点重启或 Follower 追日志时）
     */
    @Override
    public boolean onSnapshotLoad(SnapshotReader reader) {
        // 从快照文件恢复数据
        String snapshotPath = reader.getPath();
        persistService.restoreFromSnapshot(snapshotPath);
        return true;
    }
}
```

## 3.3 Raft 写入流程

```
Client (配置写入)
    │
    ▼
Leader 节点
    │
    ├──→ 1. 写入本地日志
    │
    ├──→ 2. 复制日志到 Follower
    │     ├── Follower-1 收到日志 → 写入本地 → 返回 ACK
    │     └── Follower-2 收到日志 → 写入本地 → 返回 ACK
    │
    ├──→ 3. 收到多数派 ACK (≥ N/2+1)
    │     └── 日志已提交 (committed)
    │
    ├──→ 4. 应用到状态机
    │     └── NacosStateMachine.onApply()
    │           └── 写入 MySQL + 内存
    │
    └──→ 5. 返回客户端成功
```

## 3.4 Raft 读一致性

```java
// RaftConsistencyServiceImpl.java
@Override
public Datum get(String key) {
    // 使用 Raft ReadIndex 保证线性一致性读
    ReadIndexRequest request = new ReadIndexRequest();
    request.setGroupId(group);
    
    // 1. 确认当前 Leader 身份
    raftGroupService.readIndex(request, new ReadIndexClosure() {
        @Override
        public void run(Status status, long index, byte[] reqCtx) {
            if (status.isOk()) {
                // 2. 等待本地应用到该 index 之后的状态
                stateMachine.waitForAppliedIndex(index);
            }
        }
    });
    
    // 3. 从本地状态机读取（保证读到最新提交的数据）
    return stateMachine.getDatum(key);
}
```

# 4. 一致性协议对比

| 特性 | Distro (AP) | Raft (CP) |
|------|-------------|-----------|
| 设计目标 | 高可用 + 最终一致 | 强一致性 |
| Leader/选举 | 无中心 / 无选举 | 有 Leader / 选举机制 |
| 写入方式 | 异步复制 | 多数派同步复制 |
| 读一致性 | 可能读到旧数据 | 线性一致性读 |
| 数据分片 | 一致性 Hash | 无（全量复制） |
| 恢复机制 | 反熵定时对账 | 日志恢复 + 快照 |
| 网络分区处理 | 各分区独立服务 | 少数派分区不可用 |
| 性能 | 高吞吐、低延迟 | 中等吞吐、较高延迟 |
| 适用场景 | 服务注册（临时实例） | 配置管理、持久实例 |

# 5. 脑裂问题处理

## 5.1 Distro 的脑裂处理

Distro 协议采用**节点对等 + 最终一致**的方式，天然不会脑裂：

```
网络分区前:
  ┌────┐ ┌────┐ ┌────┐
  │ N1 │ │ N2 │ │ N3 │
  └────┘ └────┘ └────┘

网络分区后:
  ┌────┐       ┌────┐ ┌────┐
  │ N1 │       │ N2 │ │ N3 │
  └────┘       └────┘ └────┘
  (分区A)       (分区B)
  
  两个分区都可以独立处理写请求
  分区恢复后，通过反熵机制进行数据合并
```

## 5.2 Raft 的脑裂处理

Raft 通过以下机制防止脑裂：

1. **任期（Term）递增**：每个选举周期 Term 递增，旧 Term 的 Leader 自动失效
2. **多数派选举**：必须获得 N/2+1 节点投票才能成为 Leader
3. **预投票（Pre-Vote）**：网络恢复后，旧 Leader 发起预投票，发现已经有新 Leader 后自动降级

```
Raft 集群 3 节点，网络分区:

┌────┐           ┌────┐ ┌────┐
│ N1 │           │ N2 │ │ N3 │
└────┘           └────┘ └────┘
(旧 Leader)       (Follower 多数派)

N1 无法获得多数派 ACK (2/3) → 写入失败
N2, N3 选举新 Leader → 新 Leader 可写入
分区恢复后 → N1 发现 Term 过期 → 降级为 Follower
```

# 6. 一致性协议在 Nacos 中的实际应用

```java
// 服务注册场景（AP 模式）
// 临时实例注册
serviceManager.registerInstance(namespaceId, serviceName, instance);
// → DistroConsistencyServiceImpl.put(key, service)
//   → 写入本地 → 异步同步 → 立即返回

// 配置管理场景（CP 模式）
configService.publishConfig(dataId, group, content);
// → RaftConsistencyServiceImpl.put(key, config)
//   → Leader 写入日志 → 复制到多数派 → 提交 → 返回

// 持久实例注册场景（CP 模式）
instance.setEphemeral(false);
serviceManager.registerInstance(namespaceId, serviceName, instance);
// → RaftConsistencyServiceImpl.put(key, service)
//   → Raft 共识 → 强一致性写入
```

# 7. 数据恢复机制

## 7.1 节点宕机恢复

```java
// Distro 模式：从其他节点的反熵同步中恢复
// 新节点加入集群后，通过 Distro 数据校验任务
// 逐步从其他节点同步缺失的数据

// Raft 模式：从 Leader 的日志中恢复
// Follower 重启后：
// 1. 加载本地快照
// 2. 从 Leader 拉取快照之后的日志
// 3. 重放日志到状态机
```

## 7.2 全量数据恢复

```java
// 从 MySQL 恢复（适用于集群全部重启）
// Nacos 启动时从 MySQL 加载全部配置和服务数据
@PostConstruct
public void loadFromDatabase() {
    // 1. 加载配置数据
    Page<ConfigInfo> configPage = persistService.findAllConfigInfo();
    for (ConfigInfo config : configPage.getContent()) {
        CacheItem cache = new CacheItem(config.getDataId(), config.getGroup());
        cache.setContent(config.getContent());
        cache.setMd5(config.getMd5());
        configCacheService.addCache(cache);
    }
    
    // 2. 加载持久实例数据
    List<Instance> instances = persistService.findAllPersistentInstances();
    for (Instance instance : instances) {
        serviceManager.registerInstance(instance);
    }
}
```
