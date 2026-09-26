
# 1. 概述

Mycat2 支持在不重启服务的情况下动态修改配置，包括数据源、分片规则、用户权限等。

```
┌────────────────────────────────────────┐
│          配置变更方式                    │
│                                         │
│  1. 管理端口 SQL 命令 (9066)              │
│  2. HTTP API                            │
│  3. 配置中心推送 (ZK/Nacos)              │
│  4. 修改配置文件 + reload 命令            │
└────────────────────────────────────────┘
```

# 2. 管理端口操作

```sql
-- 连接管理端口
mysql -h 127.0.0.1 -P 9066 -u root -p123456

-- 重新加载配置
/*!mycat:reloadConfig*/

-- 查看当前数据源
/*!mycat:showDataSources*/

-- 查看当前 Schema
/*!mycat:showSchemas*/
```

# 3. 配置中心模式

```yaml
# mycat.yml
server:
  mode: cluster
  configCenter:
    type: nacos
    address: nacos1:8848,nacos2:8848,nacos3:8848
    namespace: mycat2
    group: MYCAT_GROUP
```

```
Nacos 配置变更流程：
  1. 运维修改 Nacos 中的配置文件
  2. Nacos 推送变更通知给所有 Mycat2 实例
  3. Mycat2 校验新配置
  4. 增量更新（仅更新变更的部分）
  5. 热加载生效
```

# 4. 增量配置更新

Mycat2 的配置更新采用**增量策略**，只更新变更的部分：

```
新增数据源：
  1. 校验新数据源连接可用
  2. 创建连接池
  3. 注册到路由表
  4. 生效（新请求使用新数据源）

修改分片规则：
  1. 校验新规则
  2. 更新路由表
  3. 旧分片连接池逐步关闭
  4. 告警：分片规则变更可能导致数据路由错误

删除表：
  1. 从路由表移除
  2. 关闭对应的数据源连接池
```

# 5. 在线分片扩容

```
步骤：
  1. 准备新 MySQL 实例并建表
  2. 在 Mycat2 配置中添加新分片（status=backup）
  3. 【reloadConfig】加载新配置
  4. 数据迁移（全量 + 增量）
  5. 验证数据一致
  6. 修改分片状态为 active
  7. 【reloadConfig】生效

示例配置变更：

// 原配置：2 个分片
{
  "dataNodes": "dn0.orders,dn1.orders",
  "sharding": { "count": 2 }
}

// 新配置：4 个分片
{
  "dataNodes": "dn0.orders,dn1.orders,dn2.orders,dn3.orders",
  "sharding": { "count": 4 }
}
```

# 6. 灰度发布配置

```json
// 灰度配置：为新分片设置权重，逐步引入流量
{
  "dataNodes": {
    "dn0": { "weight": 100 },
    "dn1": { "weight": 100 },
    "dn2": { "weight": 5 }    // 新分片，先放 5% 流量
  }
}
```

# 7. 配置回滚

```sql
-- 方式 1：管理端口回滚到上一次配置
/*!mycat:rollbackConfig*/

-- 方式 2：Nacos 历史版本回滚
-- 在 Nacos 控制台 → 配置管理 → 历史版本 → 回滚

-- 方式 3：Git 管理的配置文件
git revert <commit>
./mycat reload
```

# 8. 配置校验

```java
// Mycat2 在加载配置前进行校验
public class ConfigValidator {

    public void validate(SchemaConfig config) {
        // 1. JSON 格式校验
        // 2. 数据源连接可达性校验
        // 3. 分片键在表定义中存在
        // 4. 全局表数据节点一致性
        // 5. 分片算法参数合法性
        // 6. 循环依赖检查（ER 表）
    }
}
```

# 9. 配置变更审计

```yaml
# 配置审计日志
server:
  audit:
    enabled: true
    logFile: /var/log/mycat2/audit.log
```

```
# audit.log 格式
2024-01-15 10:30:00|admin@192.168.1.100|reloadConfig|SUCCESS
2024-01-15 10:35:00|admin@192.168.1.100|addDataNode|dn3|SUCCESS
2024-01-15 10:40:00|admin@192.168.1.100|updateShardRule|orders|FAILED: invalid count
```

# 10. 最佳实践

1. **变更前备份**：任何配置变更前备份当前配置
2. **变更窗口**：配置变更在业务低峰期进行
3. **灰度验证**：新配置先在测试/预发环境验证
4. **回滚预案**：提前准备回滚操作和应急预案
5. **监控观察**：变更后观察 30 分钟监控指标
6. **分批执行**：多实例环境逐台 reload，观察无异常再下一台
