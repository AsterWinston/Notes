
# 1. 概述

Canal 的高可用基于 ZooKeeper 实现，通过主备模式（Active-Standby）确保在 Canal Server 节点故障时自动切换，保障数据订阅的连续性。

```
┌───────────────────────────────────────────────────────────────┐
│                      ZooKeeper Cluster                        │
│                                                               │
│  /otter/canal/destinations/example/                           │
│       ├── running  → 192.168.1.10:11111  (当前 Master)       │
│       ├── cluster/                                           │
│       │   ├── 192.168.1.10:11111   (EPHEMERAL)               │
│       │   ├── 192.168.1.11:11111   (EPHEMERAL)               │
│       │   └── 192.168.1.12:11111   (EPHEMERAL)               │
│       └── cursor  → {journalName: "mysql-bin.003", ...}      │
└───────────────────────────────────────────────────────────────┘
         │                    │                    │
    ┌────▼────┐          ┌────▼────┐          ┌────▼────┐
    │ Canal   │          │ Canal   │          │ Canal   │
    │ Server  │          │ Server  │          │ Server  │
    │ Node-1  │          │ Node-2  │          │ Node-3  │
    │(Running)│          │(Standby)│          │(Standby)│
    └─────────┘          └─────────┘          └─────────┘
```

# 2. ZooKeeper HA 原理

## 2.1 ZK 节点结构

```
/otter/canal/
├── cluster/
│   └── ...
├── destinations/
│   └── example/
│       ├── running               ← 当前运行 Server 的 IP:Port
│       ├── cluster/
│       │   ├── 192.168.1.10:11111  (临济节点)
│       │   ├── 192.168.1.11:11111
│       │   └── 192.168.1.12:11111
│       └── cursor                  ← 消费位点数据
```

## 2.2 主节点选举

```java
public class CanalHAController {
    
    /**
     * 启动 HA 控制器，参与选主
     */
    public void start() {
        
        // 1. 在 ZK 上注册自己的临时节点
        String path = ZK_PATH_DESTINATIONS + "/" + destination + "/cluster/" + serverInfo;
        zkClient.create()
            .creatingParentsIfNeeded()
            .withMode(CreateMode.EPHEMERAL)
            .forPath(path);
        
        // 2. 尝试抢占 running 节点（选主）
        tryToBecomeMaster();
        
        // 3. 监听 running 节点变化
        watchRunningNode();
    }
    
    /**
     * 尝试成为 Master
     */
    private void tryToBecomeMaster() {
        
        String runningPath = ZK_PATH_DESTINATIONS + "/" + destination + "/running";
        
        try {
            // 原子创建 running 节点
            zkClient.create()
                .creatingParentsIfNeeded()
                .withMode(CreateMode.EPHEMERAL)
                .forPath(runningPath, serverInfo.getBytes());
            
            // 创建成功 → 成为 Master
            becomeMaster();
            
        } catch (NodeExistsException e) {
            // 节点已存在 → 当前已有 Master
            // 读取 running 节点内容
            String currentMaster = new String(zkClient.getData().forPath(runningPath));
            
            if (currentMaster.equals(serverInfo)) {
                // 竟然是自己的节点（可能是网络断连后的残留）
                becomeMaster();
            } else {
                // 其他节点是 Master，自己作为 Standby
                becomeStandby(currentMaster);
            }
        }
    }
    
    /**
     * 成为 Master 后的操作
     */
    private void becomeMaster() {
        logger.info("become master for destination [{}]", destination);
        
        // 1. 启动 EventParser（开始拉取 binlog）
        eventParser.setMasterAddress(masterAddress);
        eventParser.start();
        
        // 2. 启动 EventSink 和 Store
        eventSink.start();
        eventStore.start();
        
        // 3. 标记状态
        runningStatus = RunningStatus.MASTER;
    }
    
    /**
     * 成为 Standby 后的操作
     */
    private void becomeStandby(String masterServer) {
        logger.info("become standby for destination [{}], master is [{}]", 
            destination, masterServer);
        
        // Standby 不启动 Parser，保持监听 running 节点
        runningStatus = RunningStatus.STANDBY;
    }
    
    /**
     * 监听 running 节点变化
     */
    private void watchRunningNode() {
        
        String runningPath = ZK_PATH_DESTINATIONS + "/" + destination + "/running";
        
        // 注册 Watcher
        zkClient.checkExists()
            .usingWatcher((CuratorWatcher) event -> {
                
                if (event.getType() == Watcher.Event.EventType.NodeDeleted) {
                    // Master 节点宕机（running 节点被删除）
                    logger.info("master node deleted, try to become master");
                    
                    // 参与新一轮选主
                    tryToBecomeMaster();
                }
                
            })
            .forPath(runningPath);
    }
}
```

# 3. HA 故障切换流程

## 3.1 正常情况

```
Node-1 (Master) → running 节点存在
Node-2 (Standby) → 等待
Node-3 (Standby) → 等待

Client → ZK → 获取 running = Node-1 → 连接 Node-1
```

## 3.2 Node-1 宕机

```
1. Node-1 宕机 → ZK 检测到会话超时

2. ZK 自动删除 Node-1 的所有临时节点：
   /running → 被删除
   /cluster/192.168.1.10:11111 → 被删除

3. Node-2 的 Watcher 触发：
   → NodeDeleted 事件
   → Node-2 尝试抢占 running 节点

4. Node-2 抢占成功 → 成为新 Master：
   → 从 ZK 读取 cursor（位点信息）
   → 启动 EventParser，从保存的位点继续
   → 更新 running 节点内容为 Node-2 的地址

5. Client 检测到 running 节点变化：
   → 自动断开 Node-1 连接
   → 连接到 Node-2
   → 通过 rollback 机制恢复未确认的消息
```

## 3.3 故障恢复时间

```
Node 宕机检测:     ZK Session Timeout (默认 3-10s)
+ Running 节点删除: 即时（ZK 自动）
+ Watcher 通知:     即时
+ 抢占 running:      < 100ms
+ Parser 重启:       1-3s（取决于 MySQL 响应速度）
+ Client 重连:       < 500ms
─────────────────────────────────
总切换时间:          约 5-15 秒
```

# 4. 配置

## 4.1 canal.properties

```properties
#################################################
#########    ZooKeeper 配置    ################
#################################################

# ZooKeeper 地址（多个用逗号分隔）
canal.zkServers = 127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183

# ZK 会话超时（毫秒）
canal.zookeeper.session.timeout.ms = 15000

# ZK 连接超时（毫秒）
canal.zookeeper.connection.timeout.ms = 15000

# ZK 重试策略
canal.zookeeper.retry.interval.ms = 5000
canal.zookeeper.retry.max = 3

# 本地注册 IP（Server 对外暴露的地址）
canal.register.ip = 192.168.1.10

# 实例管理模式
canal.instance.global.mode = manager

# 全局 Spring 配置文件
canal.instance.global.spring.xml = classpath:spring/default-instance.xml
```

## 4.2 实例管理 Spring 配置

```xml
<!-- spring/default-instance.xml — HA 模式下的实例管理 -->
<beans>
    <!-- ZK 连接 -->
    <bean id="zkClientx" class="com.alibaba.otter.canal.common.zookeeper.ZkClientx">
        <constructor-arg>
            <bean class="org.I0Itec.zkclient.ZkClient">
                <constructor-arg value="${canal.zkServers}"/>
            </bean>
        </constructor-arg>
    </bean>
    
    <!-- 实例配置管理器 -->
    <bean id="instanceConfigMonitor" 
          class="com.alibaba.otter.canal.instance.manager.spring.SpringInstanceConfigMonitor">
        <property name="zkClientx" ref="zkClientx"/>
    </bean>
</beans>
```

# 5. Client 端 HA

## 5.1 集群连接器

```java
// Client 使用 ClusterCanalConnector 自动发现 Master
CanalConnector connector = CanalConnectors.newClusterConnector(
    "192.168.1.10:2181,192.168.1.11:2181",  // ZK 地址
    "example",                                // destination
    "",                                       // username
    ""                                        // password
);

connector.connect();
connector.subscribe(".*\\..*");

// 当 Master 发生切换时，Client 自动重连到新 Master
// 不需要任何特殊的重连逻辑
```

## 5.2 Client 端自动重连流程

```java
public class ClusterCanalConnector implements CanalConnector {
    
    private volatile CanalConnector currentConnector;
    private volatile String currentMasterAddress;
    
    @Override
    public Message getWithoutAck(int batchSize, Long timeout, TimeUnit unit) {
        
        while (running) {
            try {
                if (currentConnector == null) {
                    initConnector();
                }
                
                return currentConnector.getWithoutAck(batchSize, timeout, unit);
                
            } catch (SocketException | ConnectionException e) {
                // 连接断开 → 可能是 Master 切换
                logger.warn("connection lost, try to reconnect to new master");
                
                // 关闭旧连接
                closeCurrentConnector();
                
                // 等待并重试
                sleep(2000);
            }
        }
        
        return null;
    }
    
    private void initConnector() {
        // 从 ZK 获取当前 running 的 Master
        String runningPath = "/otter/canal/destinations/" + destination + "/running";
        byte[] data = zkClient.getData().forPath(runningPath);
        String masterAddress = new String(data);
        
        if (!masterAddress.equals(currentMasterAddress)) {
            // Master 变化 → 创建新连接
            String[] parts = masterAddress.split(":");
            InetSocketAddress address = new InetSocketAddress(parts[0], 
                Integer.parseInt(parts[1]));
            
            CanalConnector newConnector = CanalConnectors.newSingleConnector(
                address, destination, username, password);
            newConnector.connect();
            newConnector.subscribe(filter);
            newConnector.rollback();  // 回滚到上次 ACK 的位置
            
            // 原子切换
            CanalConnector old = this.currentConnector;
            this.currentConnector = newConnector;
            this.currentMasterAddress = masterAddress;
            
            if (old != null) {
                old.disconnect();
            }
        }
    }
}
```

# 6. Server 端防脑裂

## 6.1 脑裂问题

```
场景：Node-1 与 ZK 之间网络断开，但 Node-1 与 MySQL 连接正常

┌──────────┐     ✗ 断开 ✗    ┌──────────┐
│  Node-1  │                │ ZooKeeper│
│ (旧Master)│                └────┬─────┘
└────┬─────┘                     │
     │ ✓ 正常 ✓                  │
┌────▼─────┐               ┌────▼─────┐
│  MySQL   │               │  Node-2   │
│          │               │(新 Master)│
└──────────┘               └────┬─────┘
                                │ ✓ 正常 ✓
                           ┌────▼─────┐
                           │  MySQL   │
                           └──────────┘

后果：两个 Master 同时向 MySQL 拉取 binlog → 可能数据不一致
```

## 6.2 防脑裂机制

```java
public class SplitBrainGuard {
    
    /**
     * 防脑裂：成为 Master 后，先检查是否已有其他 Master
     */
    @Override
    public void startEventParser() {
        
        // 1. 尝试在 ZK 上创建 running 节点
        String runningPath = buildRunningPath();
        
        try {
            zkClient.create()
                .creatingParentsIfNeeded()
                .withMode(CreateMode.EPHEMERAL)
                .forPath(runningPath, currentServer.getBytes());
            
        } catch (NodeExistsException e) {
            // running 已存在 → 已有其他 Master
            // 读取 running 中的 Server 地址
            byte[] data = zkClient.getData().forPath(runningPath);
            String existingMaster = new String(data);
            
            if (!existingMaster.equals(currentServer)) {
                // 真的是其他节点在运行 → 放弃成为 Master
                logger.warn("running node already exists: {}, give up", existingMaster);
                becomeStandby();
                return;
            }
            // 否则是自己之前的残留节点，继续
        }
        
        // 2. 周期性检查 running 节点
        startPeriodicCheck();
        
        // 3. 启动 Parser
        eventParser.start();
    }
    
    /**
     * 周期性检查自己是否还是 Master
     */
    private void startPeriodicCheck() {
        scheduler.scheduleWithFixedDelay(() -> {
            try {
                String runningPath = buildRunningPath();
                
                if (zkClient.checkExists().forPath(runningPath) == null) {
                    // running 节点消失 → 被 ZK 认为已宕机
                    logger.error("running node disappeared! stopping parser immediately");
                    
                    // 立即停止 Parser（防止脑裂）
                    eventParser.stop();
                    running = false;
                    
                    // 发送告警
                    alarmHandler.sendAlarm("Split brain detected! Parser stopped.");
                    
                } else {
                    // running 存在，检查内容是否是自己
                    byte[] data = zkClient.getData().forPath(runningPath);
                    String master = new String(data);
                    
                    if (!master.equals(currentServer)) {
                        logger.error("running node changed to {}! stopping parser", master);
                        eventParser.stop();
                        running = false;
                    }
                }
                
            } catch (Exception e) {
                logger.error("periodic check failed", e);
            }
        }, 5, 5, TimeUnit.SECONDS);
    }
}
```

# 7. GTID 模式的 HA

## 7.1 GTID 对 HA 的优势

```properties
# 传统位点模式的问题
# Master-A: mysql-bin.000123, pos=456789
# 切换到 Master-B 时，binlog 文件和位置完全不同
# 无法直接从保存的位点恢复

# GTID 模式的优势
# Master-A: uuid_a:1-1000
# 切换到 Master-B：发送 GTID = uuid_a:1-1000
# MySQL 自动定位到包含这些 GTID 的 binlog 位置
```

## 7.2 GTID HA 配置

```properties
# instance.properties
canal.instance.gtidon = true

# Canal 会使用 GTID 模式进行位点管理
# 切换 Master 后，Canal 发送 GTID SET 给新 Master
# MySQL 返回包含所需 GTID 的事务
```

# 8. 多实例 HA 负载

```
┌──────────────────────────────────────────────────────────┐
│                    ZooKeeper                              │
│                                                          │
│  /destinations/instance-a/running → Node-1  (Master)     │
│  /destinations/instance-b/running → Node-2  (Master)     │
│  /destinations/instance-c/running → Node-1  (Master)     │
│  /destinations/instance-d/running → Node-3  (Master)     │
└──────────────────────────────────────────────────────────┘
         │                │                │
    ┌────▼────┐      ┌────▼────┐      ┌────▼────┐
    │  Node-1 │      │  Node-2 │      │  Node-3 │
    │ A(M)    │      │ B(M)    │      │ D(M)    │
    │ C(M)    │      │ C(S)    │      │ A(S)    │
    │ B(S)    │      │ D(S)    │      │ B(S)    │
    │ D(S)    │      │ A(S)    │      │         │
    └─────────┘      └─────────┘      └─────────┘
    
    M = Master (运行中)
    S = Standby (待命)
    
    通过合理的 Instance 分配，实现粗粒度的负载均衡
```

# 9. 监控与告警

## 9.1 关键监控指标

| 指标 | 说明 | 告警阈值 |
|------|------|---------|
| Master 存在性 | running 节点是否始终存在 | 节点消失 > 10s |
| Master 切换次数 | 单位时间内切换次数 | > 3 次/小时 |
| Standby 节点数 | 可用的备节点数量 | = 0 |
| 消费延迟 | 当前位点 vs 最新位点 | > 60s |
| ZK 连接状态 | Server 与 ZK 的连接 | CONNECTED / LOST |

## 9.2 告警配置

```properties
canal.alarm.mode = email

# 邮件告警
canal.alarm.email.host = smtp.example.com
canal.alarm.email.port = 587
canal.alarm.email.username = canal@example.com
canal.alarm.email.password = password
canal.alarm.email.from = canal@example.com
canal.alarm.email.to = ops-team@example.com
canal.alarm.email.cc = 
canal.alarm.email.timeout = 10000
```
