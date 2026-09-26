
# 1. 集群规划

## 1.1 节点数量选择

```text
ZooKeeper 集群节点数选择原则：
+----------------------------------------------------------------------+
|                                                                       |
|  节点数 | 容错数 | 过半投票数 | 适用场景                              |
|  ------|-------|----------|---------------------------------------   |
|  1     | 0     | 1        | 仅开发/测试                              |
|  3     | 1     | 2        | 生产最常用（最小规模）                     |
|  5     | 2     | 3        | 中大型生产，需要更高容错                  |
|  7     | 3     | 4        | 大规模集群，写性能明显下降                  |
|  >7    | >3    | >4       | 不推荐（写性能严重下降）                    |
|                                                                       |
|  为什么使用奇数节点？                                                  |
|  4 节点容错 1（需 3 票） = 3 节点容错 1（需 2 票）                   |
|  4 节点比 3 节点多消耗一台机器，但容错没有提升                         |
|  5 节点容错 2（需 3 票）> 3 节点容错 1（需 2 票）                    |
|                                                                       |
+----------------------------------------------------------------------+
```

## 1.2 硬件规划

| 资源 | 最低要求 | 推荐配置 | 说明 |
|------|---------|---------|------|
| CPU | 2 核 | 4-8 核 | ZK 是 I/O 密集型，CPU 需求中等 |
| 内存 | 2 GB 堆 | 4-8 GB 堆 | 所有数据在内存中，按 ZNode 数据总量决定 |
| 磁盘（dataDir） | 20 GB SSD | 50 GB+ SSD | 快照存储，建议 SSD |
| 磁盘（dataLogDir） | 20 GB SSD | 50 GB+ NVMe | 事务日志，强烈建议独立 SSD |
| 网络 | 1 Gbps | 10 Gbps | 集群间通信延迟直接影响写性能 |

# 2. 生产环境部署

## 2.1 基础配置

```properties
# zoo.cfg 生产环境推荐配置
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/data/zookeeper/data
dataLogDir=/data/zookeeper/log         # 独立磁盘
clientPort=2181
maxClientCnxns=100                      # 根据客户端数量调整

# 自动清理
autopurge.snapRetainCount=5
autopurge.purgeInterval=12

# Leader 最大 pending 请求数
globalOutstandingLimit=1000

# 快照触发阈值
snapCount=100000

# 事务日志预分配大小
preAllocSize=65536                      # 64MB

# 强制 fsync
forceSync=yes

# 管理员服务
admin.serverPort=8080
admin.serverAddress=内网IP

# 集群成员配置
server.1=zk1.internal.example.com:2888:3888
server.2=zk2.internal.example.com:2888:3888
server.3=zk3.internal.example.com:2888:3888

# Observer 节点（读扩展）
server.4=zk4.internal.example.com:2888:3888:observer
server.5=zk5.internal.example.com:2888:3888:observer
```

## 2.2 JVM 参数优化

```bash
# zkServer.sh 中设置 JVM 参数
export JVMFLAGS="
  -Xms4g -Xmx4g                              # 堆内存固定大小
  -XX:+UseG1GC                                # 使用 G1 垃圾回收器
  -XX:MaxGCPauseMillis=100                    # 最大 GC 停顿 100ms
  -XX:InitiatingHeapOccupancyPercent=35       # G1 触发 Mixed GC 的堆占用比
  -XX:+DisableExplicitGC                      # 禁用 System.gc()
  -XX:+PrintGCDetails                         # GC 日志（生产可关闭）
  -XX:+PrintGCDateStamps
  -Xloggc:/var/log/zookeeper/gc.log
  -XX:+UseGCLogFileRotation
  -XX:NumberOfGCLogFiles=10
  -XX:GCLogFileSize=50M
  -Djava.net.preferIPv4Stack=true             # 优先使用 IPv4

  # 避免 ZK 因为长时间 GC 被误判为故障
  # tickTime * syncLimit 应大于最大 GC 停顿时间
"
```

## 2.3 操作系统优化

```bash
# 1. 禁用 Swap（ZooKeeper 对延迟敏感）
echo "vm.swappiness=0" >> /etc/sysctl.conf
sysctl -p

# 2. 文件句柄数
echo "* soft nofile 65536" >> /etc/security/limits.conf
echo "* hard nofile 65536" >> /etc/security/limits.conf

# 3. 最大进程数
echo "* soft nproc 65536" >> /etc/security/limits.conf

# 4. 内核参数优化
cat >> /etc/sysctl.conf << EOF
# 减少 TIME_WAIT
net.ipv4.tcp_tw_reuse=1
net.ipv4.tcp_fin_timeout=30

# TCP 缓冲区
net.core.rmem_max=16777216
net.core.wmem_max=16777216

# 连接队列
net.core.somaxconn=4096
EOF
sysctl -p

# 5. 挂载选项（dataDir 和 dataLogDir）
# /etc/fstab 中添加 noatime 选项减少磁盘写
# /dev/sdb  /data  ext4  defaults,noatime,nodiratime  0  2
```

# 3. Observer 部署策略

```text
Observer 部署场景：
+----------------------------------------------------------------------+
|                                                                       |
|  场景 1：读写分离                                                     |
|  ┌──────────┐                                                        |
|  │ Leader   │ ← 唯一写节点                                           |
|  │ Follower1│ ← 参与投票                                             |
|  │ Follower2│ ← 参与投票                                             |
|  │ Observer1│ ← 只读，不参与投票（部署在应用所在区域）                |
|  │ Observer2│ ← 只读，不参与投票                                     |
|  └──────────┘                                                        |
|                                                                       |
|  场景 2：跨区域部署                                                   |
|  ┌─────────────────────────────────────────────────────────┐         |
|  │  区域 A（北京）          │  区域 B（上海）                │         |
|  │  Leader + 2 Follower    │  1-2 Observer                │         |
|  │  (完整集群，强一致)       │  (就近读服务，低延迟)         │         |
|  └─────────────────────────────────────────────────────────┘         |
|                                                                       |
|  优点：                                                               |
|    · Observer 不参与投票，不影响写性能                                 |
|    · 部署在客户端附近，降低读延迟                                      |
|    · 可以无限垂直扩展读能力                                            |
|                                                                       |
+----------------------------------------------------------------------+
```

# 4. 高可用架构

## 4.1 同城双活

```text
同城双数据中心高可用：

        数据中心 A                           数据中心 B
  ┌──────────────────────┐          ┌──────────────────────┐
  │  ZK1 ─ ZK2 ─ ZK3    │◄────────►│  ZK4 ─ ZK5 ─ ZK6    │
  │   (Leader)           │ 光纤直连   │   (Follower)         │
  └──────────────────────┘          └──────────────────────┘

方案：两个数据中心使用同一个 ZK 集群
  · 节点分布在两个机房
  · 通过调整 myid 顺序引导选举
    - 优先级高的机房设置较小的 myid（但 zxid 才是决定因素）
  · 网络延迟 < 5ms（条件：同城光纤）

局限性：
  · ZK 不支持异地多活（跨地域延迟高会导致写性能急剧下降）
  · 如果 B 机房全挂，A 机房需有 >= 过半节点才能正常工作
```

## 4.2 脑裂防护补充

```text
脑裂防护最好实践：

1. 过半机制（Quorum）：N/2+1 是天然防护
   任何少数派分区无法选出 Leader

2. 配合物理方式：
   · 奇数节点
   · 关键节点部署在同一个故障域（一个机架/一个电源）
   · 确保网络分区不可能形成均势

3. 使用 Observer 避免意外：
   · Observer 参与 Leader 选举的投票
   · 但不参与事务投票
   · 部署 Observer 时要考虑到它对选举的影响

4. ZooKeeper 3.5+ 动态重配置：
   · 可通过 reconfig 命令动态调整集群成员
   · 在灾难恢复时无需重启
```

# 5. 灾难恢复

## 5.1 备份策略

```bash
# 定期备份 dataDir 和 dataLogDir
# 重要：备份时必须先做快照，确保一致性

# 1. 通过 JMX 触发快照
echo "snap" | nc localhost 2181

# 2. 备份快照目录（备份最新快照）
rsync -avz /data/zookeeper/data/ backup-server:/backup/zk/

# 3. 备份事务日志
rsync -avz /data/zookeeper/log/ backup-server:/backup/zk/

# 4. 备份配置文件
rsync -avz /usr/local/zookeeper/conf/ backup-server:/backup/zk/
```

## 5.2 恢复流程

```bash
# 场景：整个集群数据丢失，从备份恢复

# 1. 在所有节点停止 ZooKeeper
bin/zkServer.sh stop

# 2. 在所有节点清理数据目录
rm -rf /data/zookeeper/data/version-2/*
rm -rf /data/zookeeper/log/version-2/*

# 3. 在某个节点恢复快照和数据
cp /backup/zk/data/version-2/snapshot.* /data/zookeeper/data/version-2/
cp /backup/zk/log/version-2/log.* /data/zookeeper/log/version-2/

# 4. 确保 myid 文件正确
cat /data/zookeeper/data/myid

# 5. 启动第一个节点
bin/zkServer.sh start

# 6. 验证数据
bin/zkCli.sh -server localhost:2181 ls /

# 7. 依次启动其他 Follower 节点（数据由 Leader 同步）
ssh zk2 "bin/zkServer.sh start"
ssh zk3 "bin/zkServer.sh start"
```

## 5.3 快照恢复注意事项

```text
快照恢复要点：
+----------------------------------------------------------------------+
|                                                                       |
|  1. 快照是某一时刻的完整数据镜像                                     |
|  2. 快照必须和对应的最新事务日志一起恢复                              |
|     snapshot.N 记录了 zxid=N 时的数据                                 |
|     log.N 之后的事务日志记录了 N+1 之后的变更                         |
|  3. 所有节点的数据目录格式必须一致（version-2）                        |
|  4. 如果只恢复了快照没有事务日志，会丢失快照之后的全部变更            |
|                                                                       |
+----------------------------------------------------------------------+
```

# 6. 升级策略

```text
ZooKeeper 滚动升级流程：
+----------------------------------------------------------------------+
|                                                                       |
|  假设 3 节点集群 (ZK1 Leader, ZK2 Follower, ZK3 Follower)              |
|                                                                       |
|  1. 停止 ZK3                                                         |
|  2. 升级 ZK3 的 ZooKeeper 版本                                        |
|  3. 启动 ZK3                                                         |
|  4. 等待 ZK3 数据同步完成（观察日志和 mntr 输出）                     |
|                                                                       |
|  5. 停止 ZK2                                                         |
|  6. 升级 ZK2                                                         |
|  7. 启动 ZK2                                                         |
|                                                                       |
|  8. 停止 ZK1（此时 Leader 选举到 ZK2 或 ZK3）                        |
|  9. 升级 ZK1                                                         |
|  10. 启动 ZK1                                                        |
|                                                                       |
|  注意：                                                               |
|    · 始终从 Follower 开始升级，最后升级 Leader                        |
|    · 每次只升级一个节点，等待数据同步完成                              |
|    · 升级前备份 dataDir 和 dataLogDir                                 |
|    · 跨大版本升级（如 3.4.x -> 3.6.x）特别要验证兼容性               |
|    · 3.5+ 支持动态重配置，可以在线添加新版本节点滚动升级              |
|                                                                       |
+----------------------------------------------------------------------+
```

# 7. 多应用共享集群

```text
多应用共享 ZooKeeper 集群的最佳实践：

1. 使用 namespace 隔离
   +----------------------------------------------------------------+
   |  CuratorFramework:                                              |
   |    .namespace("app-a")    --> 所有路径自动加 /app-a 前缀        |
   |    .namespace("app-b")    --> 所有路径自动加 /app-b 前缀        |
   |                                                                 |
   |  ZooKeeper 树：                                                 |
   |    /app-a                                                       |
   |      /services                                                  |
   |      /config                                                    |
   |      /locks                                                     |
   |    /app-b                                                       |
   |      /services                                                  |
   |      /config                                                    |
   |      /locks                                                     |
   +----------------------------------------------------------------+

2. 配额管理（ZooKeeper 内置）
   +----------------------------------------------------------------+
   |  # 设置 /app-a 下最多 1000 个节点                                |
   |  setquota -n 1000 /app-a                                        |
   |  # 设置 /app-a 下总数据最多 10MB                                 |
   |  setquota -b 10485760 /app-a                                    |
   |  # 查看配额                                                     |
   |  listquota /app-a                                               |
   +----------------------------------------------------------------+

3. ACL 隔离
   +----------------------------------------------------------------+
   |  /app-a  ACL: digest:app-a-user:...:cdrwa                       |
   |  /app-b  ACL: digest:app-b-user:...:cdrwa                       |
   |  确保应用 A 不能访问应用 B 的数据                                |
   +----------------------------------------------------------------+

4. 连接数控制
   +----------------------------------------------------------------+
   |  maxClientCnxns=200        # 根据总客户端数调整                  |
   |  minSessionTimeout=4000    # 最小超时                            |
   |  maxSessionTimeout=40000   # 最大超时                            |
   +----------------------------------------------------------------+
```

# 8. 弹性伸缩

```text
ZooKeeper 扩缩容策略：

1. 扩展读能力：增加 Observer
   +----------------------------------------------------------------+
   |  · 在 zoo.cfg 中添加 server.X=host:2888:3888:observer            |
   |  · 启动新节点（自动从 Leader 同步数据）                          |
   |  · 无需重启现有节点（3.5+ 动态重配置）                           |
   |  · 写性能不受影响                                                |
   +----------------------------------------------------------------+

2. 扩展读写能力：增加 Follower（不推荐）
   +----------------------------------------------------------------+
   |  · 需要在所有节点修改 zoo.cfg（增加 server.X）                  |
   |  · 使用 reconfig 命令动态添加（3.5+）                            |
   |  · 写性能下降（更多节点参与投票）                                |
   |  · 使用奇数节点                                                  |
   +----------------------------------------------------------------+

3. 动态重配置（3.5+）
   +----------------------------------------------------------------+
   |  # 在线添加新节点                                                |
   |  reconfig -add server.4=host4:2888:3888:participant              |
   |                                                                  |
   |  # 在线移除节点                                                  |
   |  reconfig -remove 4                                              |
   |                                                                  |
   |  # 查看当前配置                                                  |
   |  config -c                                                       |
   |                                                                  |
   |  · 需要预先开启 reconfigEnabled=true                              |
   |  · 配置会自动同步到所有节点                                      |
   +----------------------------------------------------------------+
```

# 9. 容器化部署

```yaml
# Kubernetes StatefulSet 部署 ZooKeeper 集群
apiVersion: v1
kind: Service
metadata:
  name: zookeeper
spec:
  clusterIP: None
  selector:
    app: zookeeper
  ports:
    - name: client
      port: 2181
    - name: follower
      port: 2888
    - name: election
      port: 3888
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: zookeeper
spec:
  serviceName: zookeeper
  replicas: 3
  selector:
    matchLabels:
      app: zookeeper
  template:
    metadata:
      labels:
        app: zookeeper
    spec:
      containers:
        - name: zookeeper
          image: zookeeper:3.9
          ports:
            - containerPort: 2181
            - containerPort: 2888
            - containerPort: 3888
          env:
            - name: ZOO_MY_ID
              valueFrom:
                fieldRef:
                  fieldPath: metadata.annotations['zookeeper.myid']
            - name: ZOO_SERVERS
              value: |
                server.1=zookeeper-0.zookeeper:2888:3888;2181
                server.2=zookeeper-1.zookeeper:2888:3888;2181
                server.3=zookeeper-2.zookeeper:2888:3888;2181
          volumeMounts:
            - name: data
              mountPath: /data
            - name: datalog
              mountPath: /datalog
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 20Gi
        storageClassName: ssd
    - metadata:
        name: datalog
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 20Gi
        storageClassName: ssd-fast
```
