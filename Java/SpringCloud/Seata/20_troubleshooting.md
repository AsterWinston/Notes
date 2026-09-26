
# 1. 常见问题速查

| 现象 | 可能原因 | 解决节点 |
|------|---------|---------|
| 全局事务不生效 | 未开启数据源代理 | 检查 `seata.data-source-proxy-mode` |
| TC 连接失败 | TC Server 未启动/网络不通 | 检查 TC 状态和网络 |
| undo_log 膨胀 | 未定期清理 | 设置清理策略 |
| 全局锁等待超时 | 并发冲突/长事务 | 缩短事务/增加重试次数 |
| xid 丢失 | 异步线程未传递 | 手动传递 xid |
| Branch 注册失败 | RM 连接 TC 失败 | 检查 RM 配置和网络 |

# 2. TC 启动失败

```bash
# 1. 检查端口占用
netstat -an | grep 8091
netstat -an | grep 7091

# 2. 检查数据库连接
mysql -h localhost -u root -p seata
SELECT COUNT(*) FROM global_table;  # 表是否存在

# 3. 检查 Nacos 连接
curl http://localhost:8848/nacos/v1/ns/instance/list?serviceName=seata-server

# 4. 查看 TC 日志
tail -f seata/logs/seata-server.log

# 常见错误：
# - "Access denied for user" → 数据库用户名/密码错误
# - "Unknown database" → seata 库不存在
# - "Connection refused" → Nacos 未启动
```

# 3. 全局事务不生效

```bash
# 1. 检查数据源是否被代理
# Bean 类型应该是 DataSourceProxy 而非 HikariDataSource
@Autowired
private DataSource dataSource;

@PostConstruct
public void check() {
    log.info("DataSource type: {}", dataSource.getClass().getName());
    // 预期: io.seata.rm.datasource.DataSourceProxy
}

# 2. 检查 xid 是否绑定
# 在执行 @GlobalTransactional 方法内打印
log.info("xid: {}", RootContext.getXID());
# 预期: 192.168.1.100:8091:2039718144

# 3. 检查分支是否注册
# 在 TC Server 日志中查看
grep "branch register" seata-server.log

# 4. 检查异常是否被吞掉
# 确保 @GlobalTransactional 方法中抛出的异常能传播到 Seata 拦截器
```

# 4. undo_log 膨胀

```sql
-- 1. 查看当前的 undo_log 大小
SELECT
  COUNT(*) AS total_rows,
  ROUND(SUM(LENGTH(rollback_info)) / 1024 / 1024, 2) AS data_mb,
  MAX(log_created) AS oldest,
  MIN(log_created) AS newest
FROM undo_log;

-- 2. 查看各状态分布
SELECT log_status, COUNT(*) FROM undo_log GROUP BY log_status;
-- log_status = 0: 未决（异常）
-- log_status = 1: 已提交（可清理）

-- 3. 手动清理
DELETE FROM undo_log
WHERE log_status = 1
  AND log_modified < DATE_SUB(NOW(), INTERVAL 7 DAY)
LIMIT 10000;

-- 4. 清理后优化表
OPTIMIZE TABLE undo_log;
```

# 5. 全局锁超时

```bash
# 1. 检查 lock_table 是否有异常持锁
SELECT * FROM lock_table
WHERE gmt_modified < DATE_SUB(NOW(), INTERVAL 10 MINUTE);

# 2. 查看锁竞争情况
# 在 TC Server 日志中搜索
grep "Global lock acquire" seata-server.log | tail -50

# 3. 增大锁重试配置
seata:
  client:
    lock:
      retry-interval: 20      # 从 10ms 增大
      retry-times: 50          # 从 30 增大（共 1000ms）
```

# 6. xid 丢失/不传播

```java
// 1. 检查 Dubbo Filter
// Provider 端启动日志应有：
// [SEATA] SeataTransactionPropagationProviderFilter activated

// 2. 检查 Feign 拦截器
// 确认请求头包含 TX_XID
@FeignClient(name = "account-service", configuration = FeignConfig.class)
public interface AccountClient { ... }

@Configuration
public class FeignConfig {
    @Bean
    public RequestInterceptor seataInterceptor() {
        return template -> {
            // 手动添加（调试用，通常自动传播）
            String xid = RootContext.getXID();
            if (xid != null) {
                template.header("TX_XID", xid);
            }
        };
    }
}

// 3. 异步线程手动传递
CompletableFuture.runAsync(() -> {
    String xid = RootContext.getXID();  // ← 新线程中为空！
    // 需要在父线程中获取 xid 并传递
    RootContext.bind(parentXid);
    try {
        accountClient.debit(userId, amount);
    } finally {
        RootContext.unbind();
    }
});
```

# 7. 常见错误码

| 错误 | 含义 | 处理 |
|------|------|------|
| `TransactionException: begin failed` | TM 无法向 TC 发起 begin | 检查 TC 连接 |
| `TransactionException: branch register timeout` | 分支注册超时 | 检查 TC 负载 |
| `Global lock acquire timeout` | 全局锁获取超时 | 增加重试或检查死锁 |
| `Could not found global transaction xid` | TC 找不到全局事务 | 事务可能超时已清理 |
| `Should never be here` | undo_log 数据校验失败 | schema 不匹配 |

# 8. 诊断命令速查

```bash
# TC Server
curl http://localhost:7091/health                          # 健康检查
curl http://localhost:7091/metadata/v1/cluster-info        # 集群信息

# TC 数据库
SELECT status, COUNT(*) FROM global_table GROUP BY status; # 全局事务状态分布
SELECT COUNT(*) FROM lock_table;                            # 全局锁数量
SELECT COUNT(*) FROM undo_log WHERE log_status = 0;         # 未清理的 undo_log

# Client
curl http://localhost:8719/actuator/health                  # 应用健康

# Nacos
curl http://localhost:8848/nacos/v1/ns/instance/list?serviceName=seata-server
```
