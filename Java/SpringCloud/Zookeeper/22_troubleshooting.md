
# 1. 连接相关问题

## 1.1 客户端无法连接

```text
症状：客户端连接 ZK 失败，报 ConnectionLoss、连接超时

排查步骤：
+----------------------------------------------------------------------+
|                                                                       |
|  1. 检查网络连通性                                                     |
|     telnet zk-host 2181                                               |
|                                                                       |
|  2. 检查 ZooKeeper 服务是否正常                                       |
|     echo ruok | nc localhost 2181                                     |
|     正常输出：imok                                                    |
|                                                                       |
|  3. 检查连接数是否达到上限                                            |
|     echo mntr | nc localhost 2181                                     |
|     关注 zk_num_alive_connections 是否接近 maxClientCnxns             |
|                                                                       |
|  4. 检查客户端 connectString 是否正确                                 |
|     确保 IP/端口正确，多个节点用逗号分隔                               |
|                                                                       |
|  5. 检查 DNS 解析                                                     |
|     nslookup zk-host                                                  |
|                                                                       |
|  6. 检查防火墙                                                        |
|     iptables -L -n | grep 2181                                        |
|                                                                       |
+----------------------------------------------------------------------+
```

## 1.2 Session 频繁过期

```text
症状：客户端日志频繁出现 SessionExpired

可能原因及解决：
+----------------------------------------------------------------------+
|                                                                       |
|  1. 网络不稳定                                                        |
|     · 检查 ping 延迟和丢包率                                          |
|     · 增大 sessionTimeout                                             |
|                                                                       |
|  2. 客户端 GC 停顿过长                                                |
|     · 检查 JVM GC 日志                                                 |
|     · 优化 GC 参数，确保 GC 停顿 < sessionTimeout                     |
|     · 增大 sessionTimeout                                             |
|                                                                       |
|  3. ZooKeeper 服务端负载过高                                           |
|     · 检查 ZK 节点的 CPU/磁盘 I/O                                     |
|     · 增加 Observer 分摊读负载                                         |
|                                                                       |
|  4. 服务端 GC 导致心跳超时                                             |
|     · 优化 ZK 服务端 GC 参数                                           |
|     · 确保服务端 GC 停顿 < syncLimit * tickTime                       |
|                                                                       |
|  5. 连接数超限                                                        |
|     · 检查 maxClientCnxns 配置                                         |
|     · 适当增大限制                                                     |
|                                                                       |
+----------------------------------------------------------------------+
```

# 2. 集群问题

## 2.1 Leader 频繁切换

```text
症状：集群 Leader 反复切换，mntr 显示 zk_server_state 来回变化

可能原因：
+----------------------------------------------------------------------+
|                                                                       |
|  1. 网络问题                                                          |
|     · 节点间网络延迟不稳定或丢包                                       |
|     · Leader 与 Follower 的心跳超时                                   |
|     · 解决方案：检查网络质量，增大 syncLimit                          |
|                                                                       |
|  2. GC 导致心跳超时                                                    |
|     · Leader 长时间 GC 导致 Follower 认为 Leader 已挂                 |
|     · 解决方案：优化 GC，减少停顿                                     |
|                                                                       |
|  3. 磁盘 I/O 过慢                                                     |
|     · 事务日志写入过慢导致请求堆积                                     |
|     · 解决方案：使用 SSD，分离事务日志磁盘                            |
|                                                                       |
|  4. ZNode 数据量过大                                                   |
|     · 快照和同步时间过长                                               |
|     · 解决方案：减少 ZNode 数量，优化数据结构                         |
|                                                                       |
+----------------------------------------------------------------------+

排查方法：
  · grep "LEADING\|FOLLOWING\|LOOKING" zookeeper.log
  · 检查选举时间戳之间的间隔
  · echo mntr | nc localhost 2181 查看 avg_latency 和 outstanding_requests
```

## 2.2 节点数据不一致

```text
症状：不同节点读到的数据不同

排查步骤：
+----------------------------------------------------------------------+
|                                                                       |
|  1. 确认是否是正常的最终一致性延迟                                    |
|     · Follower 可能有短暂不一致（< 100ms）                             |
|     · 使用 sync() 确保读到最新数据                                    |
|                                                                       |
|  2. 检查是否有 Follower 数据同步失败                                  |
|     · 查看 Learner 日志                                               |
|     · grep "DIFF\|SNAP\|TRUNC" zookeeper.log                          |
|                                                                       |
|  3. 检查是否出现脑裂                                                  |
|     · 查看是否有两个节点同时宣称自己是 Leader                         |
|     · grep "LEADING" zookeeper.log 统计同一时段的 Leader 数量        |
|                                                                       |
+----------------------------------------------------------------------+
```

## 2.3 集群无法形成（一直处于 LOOKING）

```text
症状：所有节点状态都是 LOOKING，无法选出 Leader

排查步骤：
+----------------------------------------------------------------------+
|                                                                       |
|  1. 检查 myid 文件                                                    |
|     · 确保每个节点的 myid 与 zoo.cfg 中的 server.X 对应               |
|     · myid 文件只包含一个数字，无多余空格                              |
|                                                                       |
|  2. 检查选举端口（3888）连通性                                        |
|     · telnet zk-host 3888（从其他节点测试）                            |
|                                                                       |
|  3. 检查配置一致性                                                    |
|     · 所有节点的 zoo.cfg 中集群配置必须一致                            |
|     · server.X 配置不能有差异                                          |
|                                                                       |
|  4. 检查节点数量是否充足                                               |
|     · 需要至少 2 个节点才能选举（3 节点集群至少 2 个存活）             |
|                                                                       |
|  5. 检查集群间通信端口（2888）                                         |
|     · 端口是否被防火墙阻止                                             |
|                                                                       |
+----------------------------------------------------------------------+
```

# 3. 性能问题

## 3.1 读写延迟高

```text
读延迟高排查：
+----------------------------------------------------------------------+
|                                                                       |
|  1. 检查 ZK 服务端 GC 情况                                            |
|     · jstat -gcutil PID 1000                                          |
|     · GC 频繁或 GC 停顿时间长 → 调优 JVM                               |
|                                                                       |
|  2. 检查 CPU 使用率                                                    |
|     · top -p PID                                                       |
|                                                                       |
|  3. 检查 ZNode 数据量                                                  |
|     · echo mntr | nc localhost 2181 查看 zk_znode_count               |
|     · 节点数过多会降低查询效率                                         |
|                                                                       |
|  4. 检查连接数                                                        |
|     · 连接数过大可能超出处理能力                                       |
|                                                                       |
+----------------------------------------------------------------------+

写延迟高排查：
+----------------------------------------------------------------------+
|                                                                       |
|  1. 检查磁盘 I/O 性能（最关键）                                       |
|     · iostat -x 1                                                      |
|     · await（I/O 等待时间）应 < 10ms                                   |
|     · 使用 SSD 替换 HDD                                                 |
|                                                                       |
|  2. 检查事务日志磁盘是否独立                                           |
|     · dataLogDir 应使用独立磁盘                                        |
|                                                                       |
|  3. 检查 Follower 数量                                                |
|     · Follower 越多，写延迟越高（需要更多 ACK）                         |
|     · 使用 Observer 替代 Follower 做读扩展                              |
|                                                                       |
|  4. 检查 globalOutstandingLimit                                       |
|     · 未完成请求超过限制会阻塞新请求                                    |
|                                                                       |
+----------------------------------------------------------------------+
```

## 3.2 磁盘空间不足

```text
症状：ZK 节点磁盘满，服务异常

清理方法：
+----------------------------------------------------------------------+
|                                                                       |
|  1. 启用自动清理                                                      |
|     autopurge.snapRetainCount=3                                       |
|     autopurge.purgeInterval=12                                        |
|                                                                       |
|  2. 手动清理                                                          |
|     cd /data/zookeeper/data/version-2                                 |
|     # 保留最新的几个快照和对应的事务日志                                |
|     ls -la snapshot.* | sort                                          |
|     rm snapshot.0 snapshot.100 ...                                    |
|     rm log.1 log.2 ...                                                |
|                                                                       |
|  3. 使用官方清理脚本                                                  |
|     java -cp zookeeper.jar:lib/* \                                    |
|       org.apache.zookeeper.server.PurgeTxnLog \                       |
|       /data/zookeeper/data/version-2 \                                |
|       /data/zookeeper/data/version-2 -n 3                              |
|                                                                       |
+----------------------------------------------------------------------+
```

# 4. 数据问题

## 4.1 ZNode 删除失败

```text
症状：delete 操作报 NotEmpty（节点有子节点）

解决方法：
  · 确认节点确实有子节点：ls /path
  · 递归删除子节点
  · 使用 Curator：client.delete().deletingChildrenIfNeeded().forPath("/path")
  · 3.5+ CLI：deleteall /path
```

## 4.2 事务日志损坏

```text
症状：ZK 启动失败，日志报 "EOFException" 或 "Checksum failed"

恢复方法：
+----------------------------------------------------------------------+
|                                                                       |
|  1. 尝试跳过损坏的事务日志                                            |
|     修改 zoo.cfg 中 dataLogDir 指向空目录                              |
|     · ZK 会从最新快照恢复数据                                          |
|     · 会丢失快照之后的事务数据                                         |
|                                                                       |
|  2. 使用备份恢复                                                      |
|     · 从最近备份恢复快照和事务日志                                     |
|                                                                       |
|  3. 使用 LogFormatter 诊断                                            |
|     java -cp zookeeper.jar:lib/* \                                    |
|       org.apache.zookeeper.server.LogFormatter \                      |
|       /data/log/version-2/log.xxx                                     |
|     · 查看事务日志内容，定位损坏位置                                   |
|                                                                       |
+----------------------------------------------------------------------+
```

# 5. 内存与 OOM

```text
ZooKeeper OOM 排查：
+----------------------------------------------------------------------+
|                                                                       |
|  常见原因：                                                           |
|  1. Watch 数量过多导致的 OOM                                          |
|     · 检查 zk_watch_count                                             |
|     · 是否有客户端注册了大量 Watch 但未清理                           |
|                                                                       |
|  2. ZNode 数量过多（百万级别）                                        |
|     · 每个 ZNode 占用 ~200 字节内存                                    |
|     · 100万 ZNode ≈ 200MB 内存                                         |
|                                                                       |
|  3. outstanding 请求过多                                                |
|     · Leader 堆积了大量未处理的 Proposal                               |
|     · 检查 zk_outstanding_requests                                    |
|                                                                       |
|  4. Session 数量过多                                                   |
|     · 检查 zk_num_alive_connections                                   |
|                                                                       |
|  解决方案：                                                           |
|    · 增大堆内存（-Xmx）                                                |
|    · 控制 ZNode 和 Watch 数量                                         |
|    · 使用 namespace + 多集群拆分                                       |
|                                                                       |
+----------------------------------------------------------------------+
```

# 6. 常见错误码速查

| 错误码 | 异常类 | 含义 | 处理方式 |
|--------|--------|------|---------|
| CONNECTIONLOSS | ConnectionLossException | 连接丢失 | 重试（等连接恢复） |
| SESSIONEXPIRED | SessionExpiredException | Session 过期 | 重建 Session 和所有状态 |
| SESSIONMOVED | SessionMovedException | Session 迁移 | 重试（发生在重连到新节点时） |
| OPERATIONTIMEOUT | OperationTimeoutException | 操作超时 | 重试 |
| NONODE | NoNodeException | 节点不存在 | 创建节点或检查路径 |
| NODEEXISTS | NodeExistsException | 节点已存在 | 检查是否重复创建 |
| NOTEMPTY | NotEmptyException | 节点有子节点 | 先删除子节点或递归删除 |
| BADVERSION | BadVersionException | 版本冲突 | 重新读取再更新（CAS 重试） |
| NOAUTH | NoAuthException | 权限不足 | 调用 addAuthInfo() |
| NOCHILDRENFOREPHEMERALS | — | 临时节点不能有子节点 | 使用持久节点作为父节点 |
| RUNTIMEINCONSISTENCY | — | 运行时不一致 | 检查是否有逻辑错误 |

# 7. 诊断工具速查

| 场景 | 工具/命令 | 用法 |
|------|----------|------|
| 监控指标 | echo mntr \| nc localhost 2181 | 获取关键指标 |
| 健康检查 | echo ruok \| nc localhost 2181 | imok 表示正常 |
| 连接信息 | echo cons \| nc localhost 2181 | 所有客户端连接 |
| ZNode 查看 | zkCli.sh ls /path | 查看节点树 |
| GC 监控 | jstat -gcutil PID 1000 | 实时 GC 统计 |
| 线程监控 | jstack PID | 线程 dump |
| 堆内存 | jmap -heap PID | 堆使用情况 |
| 磁盘 I/O | iostat -x 1 | 磁盘性能 |
| 事务日志 | LogFormatter | 解析事务日志 |
| 快照分析 | SnapshotFormatter | 解析快照文件 |
| 网络延迟 | ping zk-host | 网络连通性 |
| 端口连通 | telnet zk-host 2181 | 服务可用性 |
