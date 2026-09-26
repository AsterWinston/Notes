
# 1. 启动失败

```bash
# 1. 检查日志
tail -f logs/wrapper.log

# 常见启动错误：
# - Address already in use → 端口被占用
netstat -an | grep 8066
lsof -i :8066

# - Config file parse error → JSON 格式错误
python3 -m json.tool conf/schemas.json
python3 -m json.tool conf/datasources.json

# - MySQL connection refused → 后端数据库不通
mysql -h 192.168.1.10 -u mycat -pmycat123 -e "SELECT 1"

# - OutOfMemoryError → 内存不足
java -Xms2g -Xmx4g -jar mycat2.jar
```

# 2. 连接 Mycat2 失败

```bash
# 1. 检查 Mycat2 是否启动
ps aux | grep mycat2

# 2. 检查端口是否监听
netstat -tlnp | grep 8066

# 3. 测试连接
mysql -h 127.0.0.1 -P 8066 -u root -p123456

# 4. 检查用户配置
cat conf/users.json

# 5. 检查 IP 白名单
# users.json 中的 ipWhiteList
```

# 3. 查询返回空结果

```bash
# 1. 检查路由是否正确
EXPLAIN SELECT * FROM orders WHERE id = 1;
# → 确认路由到正确的分片

# 2. 直接连接后端 MySQL 查询
mysql -h 192.168.1.10 -u mycat -p -e "SELECT * FROM order_db.orders WHERE id = 1"

# 3. 检查数据是否存在
# Mycat2 是透明代理，数据在后端 MySQL 中

# 4. 检查 Schema 配置
/*!mycat:showSchemas*/
```

# 4. 分片数据倾斜

```sql
-- 1. 检查各分片数据量
-- 连接每个后端 MySQL
SELECT COUNT(*) FROM orders;  -- 比较各分片行数

-- 2. 检查分片算法
/*!mycat:showSchemas*/
-- 确认分片算法的 clazz 和 count 配置

-- 3. 分析原因
-- 取模算法 → 一般均匀，不均是数据分布问题
-- 范围算法 → 某范围数据过多
-- 枚举算法 → 某枚举值数据过多
```

**解决**：

```json
// 调整分片算法
{
  "clazz": "io.mycat.router.function.PartitionByMurmurHash",
  "properties": {
    "columnName": "id",
    "count": 8,
    "seed": 0,
    "virtualBucketTimes": 160    // 增加虚拟桶提高均匀度
  }
}
```

# 5. 读写分离不一致

```
问题：写后立即读取不到最新数据

排查：
  1. 检查 Slave 复制延迟
     SHOW SLAVE STATUS; → Seconds_Behind_Master

  2. 检查代码是否强制走 Master
     /*!mycat:db_type=master*/ SELECT ... WHERE id = 1;

  3. 关键读操作使用 Hint 强制 Master
```

# 6. OOM（内存溢出）

```bash
# 1. 查看 GC 日志
jstat -gcutil $(pgrep -f mycat2) 1000

# 2. 生成 heap dump
jmap -dump:live,format=b,file=heap.hprof $(pgrep -f mycat2)

# 3. 常见 OOM 原因
# - 前端连接数过多 → 增大 maxCon 或限制
# - 大结果集未流式处理 → 检查 SQL 是否少了 LIMIT
# - 堆外内存不足 → 增大 MaxDirectMemorySize

# 4. 临时缓解
/*!mycat:reloadConfig*/
```

# 7. SQL 执行慢

```bash
# 1. 查看慢 SQL
/*!mycat:showSlowSQL*/

# 2. 查看执行计划
EXPLAIN SELECT ... FROM orders WHERE ...;
# 关注是否全分片扫描（node 列是否有多个分片）

# 3. 直接在后端 MySQL 执行
mysql -h backend -e "SELECT ..."
# 判断慢在 Mycat2 层还是 MySQL 层

# 4. 检查分片命中
# WHERE 条件缺少分片键 → 全分片扫描
```

# 8. 全局锁等待

```sql
-- 查看锁等待
SELECT * FROM information_schema.innodb_lock_waits;

-- 查看当前事务
SELECT * FROM information_schema.innodb_trx;

-- 杀死长时间持有锁的事务
KILL <trx_id>;
```

# 9. 常见错误码

| 错误 | 含义 | 解决 |
|------|------|------|
| `Can't connect to MySQL server on '...'` | 后端 MySQL 不可达 | 检查网络和 MySQL 状态 |
| `Access denied for user` | 后端 MySQL 认证失败 | 检查数据源用户密码 |
| `Table doesn't exist` | 后端 MySQL 中表不存在 | 确认表已创建 |
| `XA_RBDEADLOCK` | XA 事务死锁 | 优化事务，缩短持锁时间 |
| `Too many connections` | 后端连接池满 | 增大 maxCon 或减少并发 |
| `No dataNode found` | 路由找不到数据节点 | 检查 Schema 配置 |

# 10. 快速诊断命令

```bash
# Mycat2 进程
ps aux | grep mycat2
jstack $(pgrep -f mycat2) > mycat2.jstack

# 连接状态
netstat -an | grep 8066 | wc -l          # 前端连接数
netstat -an | grep 3306 | wc -l          # 后端连接数

# 日志
tail -100 logs/wrapper.log | grep ERROR
tail -100 logs/mycat2.log | grep -i exception

# 管理命令
mysql -h 127.0.0.1 -P 9066 -u root -p -e "/*!mycat:showReactors*/"
mysql -h 127.0.0.1 -P 9066 -u root -p -e "/*!mycat:showHeartbeatStatus*/"
```
