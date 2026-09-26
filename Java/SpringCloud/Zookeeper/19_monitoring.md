
# 1. 监控概述

ZooKeeper 作为分布式系统的核心基础设施，其健康状态直接影响所有依赖它的服务。完善的监控体系是生产运维的必备条件。

```text
监控的四个黄金信号：
+----------------------------------------------------------------------+
|                                                                       |
|  1. 延迟（Latency）：请求处理耗时                                     |
|  2. 流量（Traffic）：请求量（QPS）                                    |
|  3. 错误（Errors）：失败率、异常类型                                  |
|  4. 饱和度（Saturation）：队列深度、连接数、Watch 数                  |
|                                                                       |
+----------------------------------------------------------------------+
```

# 2. 四字命令（Four Letter Words）

ZooKeeper 提供了一套轻量级的统计接口，通过 `nc` 或 `telnet` 发送四字母命令到客户端端口即可获取信息。

## 2.1 常用四字命令

| 命令 | 说明 | 常用度 |
|------|------|--------|
| **mntr** | 监控指标汇总（最常用） | ★★★★★ |
| **stat** | 服务端状态和统计 | ★★★★★ |
| **ruok** | 服务是否正常运行（imok） | ★★★★ |
| **srvr** | 服务端详细信息 | ★★★★ |
| **cons** | 所有客户端连接详情 | ★★★ |
| **wchc** | Watch 路径信息 | ★★★ |
| **dump** | 未完成的 Session 和临时节点 | ★★ |
| **envi** | 服务端环境配置 | ★★ |
| **srst** | 重置统计计数器 | ★★ |
| **conf** | 服务端配置详情 | ★★ |

## 2.2 命令使用

```bash
# 基础用法
echo mntr | nc localhost 2181

# mntr 输出示例（最重要的监控命令）
zk_version      3.9.2
zk_avg_latency  3
zk_max_latency  45
zk_min_latency  0
zk_packets_received     1502340
zk_packets_sent         1502345
zk_num_alive_connections        28
zk_outstanding_requests 2
zk_server_state follower
zk_znode_count  12500
zk_watch_count  320
zk_ephemerals_count     45
zk_approximate_data_size        5242880
zk_open_file_descriptor_count   128
zk_max_file_descriptor_count    65536
zk_fsync_threshold_exceed_count 0
zk_learners      2
zk_synced_followers     2
zk_synced_non_voting_followers  2
zk_quit_leading_due_to_disloyal_voter     0
```

```bash
# stat — 基本状态信息
echo stat | nc localhost 2181
# Zookeeper version: 3.9.2-..., built on ...
# Clients:
#  /192.168.1.100:52341[0](queued=0,recved=1,sent=0)
# Latency min/avg/max: 0/3/45
# Received: 1502340
# Sent: 1502345
# Connections: 28
# Outstanding: 2
# Zxid: 0x20000001a
# Mode: follower
# Node count: 12500

# ruok — 快速健康检查
echo ruok | nc localhost 2181
# 输出: imok（表示服务正常）

# srvr — 详细信息
echo srvr | nc localhost 2181

# cons — 所有客户端连接
echo cons | nc localhost 2181
```

## 2.3 安全注意事项

```properties
# zoo.cfg 中配置四字命令白名单（3.5+）
# 生产环境应限制可用命令

# 仅开放 mntr, ruok, stat（安全级别最小化）
4lw.commands.whitelist=mntr,ruok,stat,srvr

# 禁用所有四字命令（最安全，但影响监控）
# 4lw.commands.whitelist=
```

# 3. JMX 监控

ZooKeeper 暴露了丰富的 JMX MBean 指标：

```bash
# 启用 JMX（zkServer.sh 中默认启用）
export JMXPORT=9999
export JMXAUTH=false
export JMXSSL=false

# 使用 jconsole 或 jmxterm 连接
jconsole localhost:9999
```

## 3.1 关键 JMX MBean

```text
ZooKeeper JMX MBean 分类：

1. org.apache.ZooKeeperService
   └── ReplicatedServer_id{1-3}
       ├── Leader / Follower / Observer（角色相关指标）
       ├── InMemoryDataTree（ZNode 数量、Watch 数等）
       ├── SessionTracker（Session 统计）
       ├── ConnectionMetrics（连接统计）
       └── name0=StandaloneServer_port{2181}
           ├── ClientPort（客户端连接统计）
           └── RequestProcessor（请求处理统计）

2. java.lang（标准 JVM MBean）
   ├── Memory（堆内存使用）
   ├── GarbageCollector（GC 统计）
   └── Threading（线程统计）
```

## 3.2 Java 中读取 JMX 指标

```java
// 使用 JMX 编程式获取指标
public class ZkJmxMonitor {
    public Map<String, Object> getZkMetrics() throws Exception {
        JMXServiceURL url = new JMXServiceURL(
            "service:jmx:rmi:///jndi/rmi://localhost:9999/jmxrmi");
        JMXConnector connector = JMXConnectorFactory.connect(url);
        MBeanServerConnection mbsc = connector.getMBeanServerConnection();

        // 查询 ZK MBean
        String zkBean = "org.apache.ZooKeeperService:name0=*";
        Set<ObjectName> names = mbsc.queryNames(
            new ObjectName(zkBean), null);

        Map<String, Object> metrics = new HashMap<>();
        for (ObjectName name : names) {
            MBeanInfo info = mbsc.getMBeanInfo(name);
            for (MBeanAttributeInfo attr : info.getAttributes()) {
                metrics.put(
                    name + "." + attr.getName(),
                    mbsc.getAttribute(name, attr.getName())
                );
            }
        }
        return metrics;
    }
}
```

# 4. Prometheus 集成

## 4.1 ZooKeeper Exporter

```bash
# 使用 Prometheus JMX Exporter 暴露 ZK 指标
# 1. 下载 JMX Exporter
wget https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.20.0/jmx_prometheus_javaagent-0.20.0.jar

# 2. 创建 JMX Exporter 配置文件（zk-jmx-config.yaml）
cat > /usr/local/zookeeper/conf/jmx-config.yaml << 'EOF'
rules:
  - pattern: "org.apache.ZooKeeperService.*"
  - pattern: "java.lang.*"
EOF

# 3. 修改 zkServer.sh，添加 javaagent
export JVMFLAGS="
  -javaagent:/path/to/jmx_prometheus_javaagent.jar=9500:/path/to/jmx-config.yaml
  ${JVMFLAGS}
"
```

## 4.2 Prometheus 配置文件

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'zookeeper'
    static_configs:
      - targets:
        - 'zk1.example.com:9500'
        - 'zk2.example.com:9500'
        - 'zk3.example.com:9500'
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        regex: '(.*):\d+'
        replacement: '${1}'
```

## 4.3 关键 Prometheus 指标

```text
关键 ZK 指标（Prometheus 格式）：

# 客户端连接数
zookeeper_num_alive_connections 28

# 平均延迟（ms）
zookeeper_avg_latency 3

# ZNode 总数
zookeeper_znode_count 12500

# Watch 总数
zookeeper_watch_count 320

# 临时节点数
zookeeper_ephemerals_count 45

# 未完成请求数
zookeeper_outstanding_requests 2

# 已同步的 Follower 数
zookeeper_synced_followers 2

# 服务角色（1=Leader, 2=Follower, 3=Observer）
zookeeper_server_state 2

# 数据大小
zookeeper_approximate_data_size 5242880

# JVM 堆内存
jvm_memory_bytes_used{area="heap"} 2147483648
```

# 5. 告警规则

```yaml
# Prometheus 告警规则（alerts.yml）
groups:
  - name: zookeeper
    rules:
      # 1. ZK 节点宕机
      - alert: ZookeeperDown
        expr: up{job="zookeeper"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "ZooKeeper instance {{ $labels.instance }} is down"

      # 2. Leader 不存在或 Leader 切换
      - alert: ZookeeperNoLeader
        expr: sum(zookeeper_server_state == 1) == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "No Leader in ZooKeeper cluster"

      # 3. 平均延迟过高
      - alert: ZookeeperHighLatency
        expr: zookeeper_avg_latency > 50
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "ZooKeeper avg latency > 50ms"

      # 4. 未完成请求堆积
      - alert: ZookeeperOutstandingQueue
        expr: zookeeper_outstanding_requests > 500
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "ZK outstanding requests > 500"

      # 5. 文件描述符接近上限
      - alert: ZookeeperFileDescriptorNearLimit
        expr: (zookeeper_open_file_descriptor_count /
               zookeeper_max_file_descriptor_count) > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "ZK file descriptor usage > 80%"

      # 6. Session 过期频繁
      - alert: ZookeeperHighSessionExpiration
        expr: rate(zookeeper_session_expired_total[5m]) > 10
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High rate of session expiration"

      # 7. ZNode 数量异常增长
      - alert: ZookeeperZnodeCountGrowth
        expr: deriv(zookeeper_znode_count[1h]) > 1000
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "ZNode count growing rapidly (+1000/hour)"
```

# 6. 日志管理

## 6.1 日志配置

```properties
# logback.xml / log4j.properties

# ZooKeeper 服务器日志
zookeeper.root.logger=INFO, ROLLINGFILE
zookeeper.console.threshold=WARN

# 日志目录
zookeeper.log.dir=/var/log/zookeeper
zookeeper.log.file=zookeeper.log

# 日志轮转
zookeeper.log.maxfilesize=100MB
zookeeper.log.maxbackupindex=10
```

## 6.2 关键日志内容

```text
需要重点关注的日志内容：

1. Leader 选举日志
   [QuorumPeer] - LOOKING -> LEADING/FOLLOWING
   [FastLeaderElection] - Notification, LOOKING

2. Session 过期日志
   [SessionTracker] - Expiring session 0x...
   [SessionTrackerImpl] - Session 0x... is expired

3. 数据同步日志
   [LearnerHandler] - Sending SNAP/DIFF/TRUNC

4. 客户端连接日志
   [NIOServerCnxn] - Established session 0x...
   [NIOServerCnxn] - Closed session 0x...

5. 快照日志
   [FileTxnSnapLog] - Snapshotting: 0x... to snapshot.

6. 错误日志
   [NIOServerCnxn] - Unexpected exception
   [CommitProcessor] - ...
```

# 7. 管理工具

| 工具 | 说明 | 链接/命令 |
|------|------|----------|
| **zkCli.sh** | 官方 CLI | 自带 |
| **zkServer.sh** | 启动/停止/状态 | 自带 |
| **zkCleanup.sh** | 清理旧快照和日志 | 自带 |
| **zkSnapShotToolkit.sh** | 快照分析工具 | 自带 |
| **exhibitor** | Netflix 开源的 ZK 管理 UI | GitHub |
| **zk-web** | 基于 Web 的 ZK 管理工具 | GitHub |
| **prettyZoo** | 桌面端 ZK GUI 工具 | GitHub |

# 8. 运维 Checklist

```text
日常巡检 Checklist：

□ 所有节点进程存活
□ Leader 存在且稳定（不超过 1 次切换/天）
□ avg_latency < 10ms
□ outstanding_requests < 100
□ num_alive_connections 在预期范围内
□ znode_count 无异常暴增
□ watch_count 无异常增长
□ 磁盘空间充足（> 30% 可用）
□ 文件描述符使用率 < 60%
□ GC 停顿时间 < syncLimit * tickTime
□ 无 ERROR 级别日志增量
□ 快照自动清理正常工作

周/月检查：

□ 回顾 Prometheus 大盘趋势
□ 检查快照文件大小和数量
□ 检查事务日志文件大小
□ 审查 ACL 配置
□ 演练备份恢复流程
□ 检查是否有可用版本升级
```
