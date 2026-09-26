
# 1. 连接类问题

## 1.1 无法连接MongoDB

```bash
# 症状：mongosh或应用无法连接

# 排查步骤：
# 1. 检查MongoDB服务是否运行
systemctl status mongod
# 或
ps aux | grep mongod

# 2. 检查端口是否监听
netstat -an | grep 27017

# 3. 检查bindIp配置
# 如果bindIp是127.0.0.1，只能本地连接
# 需要改为0.0.0.0或特定IP才能外部访问

# 4. 检查防火墙
sudo ufw status
# 或
sudo iptables -L

# 5. 检查认证配置
mongosh --eval "db.runCommand({connectionStatus: 1})"
```

## 1.2 认证失败

```
Authentication failed.

常见原因：
1. 用户名/密码错误
2. 认证数据库不正确（authSource）
3. 认证机制不匹配（SCRAM-SHA-1 vs SCRAM-SHA-256）
4. 用户不存在或权限不足
5. keyFile不匹配（副本集节点间认证）

排查方法：
mongosh -u user -p pass --authenticationDatabase admin
mongosh "mongodb://user:pass@host:27017/admin"  # 连接字符串指定authSource
```

## 1.3 连接池耗尽

```java
// 症状：
// com.mongodb.MongoTimeoutException: 
// Timed out after 2000 ms while waiting to connect

// 原因：
// 1. 连接池maxSize太小
// 2. 存在连接泄漏（没有正确关闭资源）
// 3. 慢查询占用连接时间过长
// 4. 服务端连接数达到上限

// 排查：
db.serverStatus().connections
// 如果current ≈ available → 连接数不足
// 如果连接数异常高 → 可能存在连接泄漏

// 解决：
// 1. 增加maxSize
.maxSize(100)
// 2. 确保每次操作后关闭游标
try (MongoCursor<Document> cursor = collection.find().iterator()) {
    // ...
}
// 3. 检查慢查询
db.currentOp({"secs_running": {$gt: 5}})
```

# 2. 性能类问题

## 2.1 查询突然变慢

```
排查步骤：

1. 确认有无索引变化
db.collection.getIndexes()

2. 检查执行计划
db.collection.find({field: value}).explain("executionStats")

3. 检查计划缓存
db.collection.getPlanCache().listQueryShapes()
// 如果查询计划突然改变 → 清除计划缓存
db.collection.getPlanCache().clear()

4. 检查有无长时间运行的操作
db.currentOp({"secs_running": {$gt: 5}})

5. 检查有无大量并发写入
db.serverStatus().opcounters
db.currentOp({"op": "insert", "active": true})

6. 检查磁盘IO
iostat -x 1
// 如果 %util 接近100% → 磁盘瓶颈

7. 检查内存
db.serverStatus().wiredTiger.cache
// 如果 pages evicted by application threads 很高 → 缓存不足
```

## 2.2 写入变慢

```
排查步骤：

1. 检查索引数量
db.collection.getIndexes().length
// 每个索引都拖慢写入

2. 检查Write Concern
// w: "majority"比w: 1慢很多
// j: true会增加journal写入等待

3. 检查oplog大小
rs.printReplicationInfo()
// oplog太小，频繁全量同步

4. 检查磁盘空间
db.stats().storageSize
// disk full? → 扩容或清理

5. 分片集群检查Balancer
sh.isBalancerRunning()
// 如果是，停止Balancer或设置非高峰期窗口
```

## 2.3 内存使用过高

```javascript
// 症状：WiredTiger cache使用率接近100%，频繁换页

// 检查内存使用
db.serverStatus().wiredTiger.cache
// "bytes currently in the cache" / "maximum bytes configured" → 缓存使用率
// "pages evicted by application threads" → 换页次数

// 解决方案：
// 1. 增大WiredTiger缓存（修改mongod.conf）
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 8

// 2. 检查是否有未使用的索引（占缓存）
db.collection.aggregate([{$indexStats: {}}])

// 3. 优化查询减少全表扫描
// 4. 考虑分片分散数据
```

## 2.4 聚合管道内存溢出

```
Error: Exceeded memory limit for $group, but didn't allow external sort.

原因：
- $group、$sort、$bucket等阶段默认限制100MB内存
- 数据量超出限制

解决方案：
// 1. 允许使用磁盘（性能下降）
db.collection.aggregate([...], { allowDiskUse: true })

// 2. 在$group前添加$match过滤数据
// 3. 为$sort阶段创建索引
// 4. 分批聚合（使用多个管道）
```

# 3. 副本集问题

## 3.1 从节点同步延迟

```javascript
// 查看延迟
rs.printSecondaryReplicationInfo()

// 原因：
// 1. 网络延迟
// 2. 从节点硬件性能不足
// 3. 主节点写入量过大
// 4. 从节点正在处理慢查询

// 解决：
// 1. 增加从节点硬件配置
// 2. 调整read preference减少从节点读负载
// 3. 增大oplog大小
// 4. 检查从节点是否有慢查询占用资源
```

## 3.2 选举频繁

```javascript
// 症状：rs.status()显示主节点频繁切换

// 原因：
// 1. 网络不稳定（心跳超时）
// 2. 主节点资源不足（CPU/内存/IO过载）
// 3. electionTimeoutMillis设置太短

// 排查：
// 查看选举历史
rs.status().members.forEach(m => print(m.name, m.stateStr))

// 查看日志
// grep "step down" / grep "election" mongod.log
```

## 3.3 Rollback（数据回滚）

```
场景：旧主节点恢复后数据被回滚

发生条件：
1. 主节点A提交了写入（w: 1）
2. 从节点B、C未收到
3. 主节点A宕机，B被选为新的主节点
4. A恢复后发现自己的数据不在新主节点的oplog中
5. A回滚这些数据

预防：
- 始终使用 w: "majority" 确保数据被大多数节点确认
- 监控复制延迟确保从节点及时跟上

恢复：
- 回滚文件保存在 <dbPath>/rollback/
- 使用mongorestore将回滚数据恢复到其他集合
```

# 4. 分片集群问题

## 4.1 数据分布不均

```javascript
// 检查分布
db.collection.getShardDistribution()

// 原因：
// 1. 分片键基数低
// 2. 分片键单调递增（热点）
// 3. chunk分裂/迁移策略问题

// 解决：
// 1. 重新选择分片键（5.0+支持reshardCollection）
// 2. 预分片（提前分配空chunk）
// 3. 使用哈希分片
// 4. 手动迁移chunk均衡数据
sh.moveChunk("mydb.collection", {key: "value"}, "targetShard")
```

## 4.2 mongos路由错误

```
症状：查询返回不完整数据或错误

常见原因：
1. config server不可用 → mongos无法获取路由信息
2. mongos配置缓存过期 → 需要刷新
3. mongos与分片节点网络不通

排查：
db.adminCommand("flushRouterConfig")
sh.status()  // 检查集群状态
```

# 5. 常见错误码

| 错误码 | 含义 | 解决方案 |
|--------|------|---------|
| 11000 | 唯一索引冲突 | 检查_id或唯一字段是否重复 |
| 11001 | 唯一索引冲突（复制） | 同上 |
| 112 | WriteConflict写冲突 | 在事务中等待并重试 |
| 211 | 键过长（>1024字节） | 缩短索引字段的值 |
| 10334 | BSON文档过大（>16MB） | 拆分文档或使用GridFS |
| 253 | 集合不存在 | 检查集合名、数据库名是否正确 |
| 225 | 事务超时 | 缩短事务/缩短操作 |
| 10107 | 不是主节点 | 检查当前连接的节点角色 |
| 11600 | 节点不健康 | 检查节点网络和状态 |
| 8000 | 认证失败 | 检查凭据 |

# 6. 诊断工具速查

```bash
# 1. 查看当前操作
mongosh --eval "db.currentOp()"

# 2. 终止长时间操作
mongosh --eval "db.killOp(<opid>)"

# 3. 检查复制延迟
mongosh --eval "rs.printSecondaryReplicationInfo()"

# 4. 查看慢查询
mongosh --eval "db.system.profile.find({millis:{\$gt:100}}).sort({ts:-1}).limit(5).pretty()"

# 5. 检查锁等待
mongosh --eval "db.serverStatus().locks"

# 6. 检查连接数
mongosh --eval "db.serverStatus().connections"

# 7. 分析磁盘IO
iostat -x 1 5

# 8. 分析内存
free -h
mongosh --eval "db.serverStatus().wiredTiger.cache"

# 9. 日志分析工具
mlogfilter mongod.log --slow 200

# 10. 网络延迟检查（mongod日志中搜索）
grep "Slow query" mongod.log | tail -20
grep "WiredTiger" mongod.log | tail -20
```

# 7. 紧急操作手册

## 7.1 磁盘空间不足

```bash
# 1. 确认磁盘使用
df -h

# 2. 立即措施
# - 删除不需要的数据库/集合
# - 清理journal日志（mongod会自动管理）
# - 压缩集合
mongosh --eval "db.runCommand({compact: 'collectionName'})"

# 3. 长期解决方案
# - 扩展磁盘
# - 分片集群添加更多分片
# - 清理旧数据
```

## 7.2 节点宕机

```bash
# 1. 检查mongod状态
systemctl status mongod
ps aux | grep mongod

# 2. 检查日志
tail -100 /var/log/mongodb/mongod.log

# 3. 尝试重启
systemctl restart mongod

# 4. 如果启动失败
# - 检查磁盘空间 df -h
# - 检查文件权限 ls -la /data/db/
# - 检查mongod.lock文件是否存在
# - 检查配置文件语法
mongod --config /etc/mongod.conf --dry-run

# 5. 紧急恢复（从备份或从其他节点同步）
```

## 7.3 性能雪崩

```bash
# 当性能突然急剧下降时

# 1. 暂停非关键操作
mongosh --eval "
    // 终止长时间运行的查询
    db.currentOp({'secs_running': {\$gt: 10}}).inprog.forEach(op => {
        db.killOp(op.opid)
    })
"

# 2. 检查是否有索引构建
db.currentOp({'command.createIndexes': {\$exists: true}})

# 3. 分片集群暂停Balancer
mongosh --eval "sh.stopBalancer()"

# 4. 降低写入压力
# - 应用层限流
# - 暂时降低write concern
# - 暂停非关键的批量任务

# 5. 如果需要紧急重启
systemctl restart mongod
```
