
# 1. 路由不正确

```sql
-- 1. 检查 SQL 是否正确路由
PREVIEW SELECT * FROM t_order WHERE order_id = 1;

-- 预期输出：
-- +------------------+------------------------------------------------+
-- | dataSourceName   | actualSQL                                      |
-- +------------------+------------------------------------------------+
-- | ds0              | SELECT * FROM t_order_1 WHERE order_id = 1    |
-- +------------------+------------------------------------------------+

-- 2. 检查分片规则配置
SHOW SHARDING TABLE RULE t_order;

-- 3. 常见原因：
-- - 分片算法表达式错误（如 user_id % 2 写成 order_id % 2）
-- - 分片键列名与 SQL 中的列名不一致
-- - 分片策略类型选择不当
```

# 2. Proxy 连接失败

```bash
# 1. 检查 Proxy 进程
ps aux | grep shardingsphere-proxy

# 2. 检查端口监听
netstat -tlnp | grep 3307

# 3. 测试连通性
mysql -h 127.0.0.1 -P 3307 -u root -p

# 4. 常见错误：
# ERROR 2003 (HY000): Can't connect → Proxy 未启动
# ERROR 1045 (28000): Access denied → 用户名/密码错误
# ERROR 1040 (HY000): Too many connections → 连接数满

# 5. 查看 Proxy 日志
tail -100 /opt/shardingsphere-proxy/logs/stdout.log | grep ERROR
```

# 3. 分布式事务超时

```bash
# 1. 检查事务超时配置
SHOW VARIABLES LIKE '%transaction%';

# 2. 查看 XA 事务状态
# MySQL:
XA RECOVER;

# 3. 手动清理悬挂的 XA 事务
XA COMMIT 'xid-xxx';     # 或
XA ROLLBACK 'xid-xxx';

# 4. 常见原因：
# - 数据库连接断开导致 prepared 事务悬挂
# - 事务执行时间超过 timeout
# - 网络波动导致 RPC 失败

# 5. 缩短事务时间
# - 业务拆分为多个小事务
# - 减少事务中的 RPC 调用
# - 增加事务超时阈值
```

# 4. SQL 解析错误

```bash
# 1. 使用 PARSE 命令测试
PARSE SELECT * FROM t_order WHERE ...;

# 2. 常见不支持的语法：
# - 存储过程 CALL
# - 复杂子查询（嵌套超过 2 层）
# - 特定数据库的专有函数（如 MySQL 的 GROUP_CONCAT）

# 3. 解决：
# - 简化 SQL
# - 拆分为多条 SQL
# - 使用 Hint 路由绕过解析
```

# 5. 内存溢出

```bash
# 1. 查看 JVM 内存
jstat -gcutil $(pgrep -f shardingsphere) 1000

# 2. 生成 heap dump
jmap -dump:live,format=b,file=heap.hprof $(pgrep -f shardingsphere)

# 3. 常见原因：
# - 大结果集未流式归并 → 增加 executor-size 或调小结果集
# - 连接数过多 → 限制 maxPoolSize
# - 元数据缓存 → 定期 REFRESH TABLE METADATA

# 4. 增加内存
# -Xms2g -Xmx4g
```

# 6. 数据不一致

```bash
# 1. 检查各分片数据量
# 直接连接每个物理库查询
SELECT COUNT(*) FROM t_order_0;
SELECT COUNT(*) FROM t_order_1;

# 2. 验证分片算法
# 计算某个 ID 应该路由到哪个分片
# user_id = 5 → user_id % 2 = 1 → ds1

# 3. 常见原因：
# - 分片规则修改后历史数据未迁移
# - 数据写入时直连了物理库而非经过 ShardingSphere
# - 分布式主键生成器 worker-id 冲突

# 4. 修复：
# - 全量迁移到新分片规则
# - 启用 sql-show 确认 SQL 正确路由
```

# 7. 读写分离不一致

```bash
# 1. 检查主从延迟
# 连接从库
SHOW SLAVE STATUS\G
# Seconds_Behind_Master

# 2. 强制走主库（代码中）
HintManager hintManager = HintManager.getInstance();
hintManager.setWriteRouteOnly();

# 3. 检查读写分离规则
SHOW READWRITE_SPLITTING RULES;
```

# 8. 快速诊断命令

```bash
# ShardingSphere Proxy
SHOW STORAGE UNITS;                    # 查看所有存储单元
SHOW SHARDING TABLE RULES;             # 查看所有分片规则
PREVIEW SQL;                           # 预览 SQL 路由
SHOW INSTANCE LIST;                    # 查看集群节点
SHOW READWRITE_SPLITTING RULES;        # 查看读写分离

# 系统层面
netstat -an | grep 3307 | wc -l        # Proxy 连接数
jstack $(pgrep -f proxy) > jstack.log  # 线程栈
```
