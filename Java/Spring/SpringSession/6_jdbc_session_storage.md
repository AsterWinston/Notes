# 1. JDBC 作为 Session 存储

## 1.1 适用场景

| 场景 | 说明 |
|------|------|
| **不想引入 Redis** | 已有关系型数据库，不希望增加技术栈复杂度 |
| **Session 数量较少** | 中小规模应用，Session 并发量在数据库可承受范围内 |
| **需要 SQL 查询** | 需要进行 Session 统计、审计等操作 |
| **事务一致性** | Session 和业务数据在同一个数据库中，方便事务管理 |

## 1.2 依赖引入

```xml
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-jdbc</artifactId>
</dependency>
<!-- 加上数据库驱动 -->
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
</dependency>
<!-- 或 PostgreSQL -->
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
</dependency>
```

---

# 2. 数据库表结构

## 2.1 自动建表

Spring Session JDBC 在启动时会自动创建两张表：

```yaml
spring:
  session:
    jdbc:
      initialize-schema: always   # always | embedded | never
```

| 值 | 说明 |
|----|------|
| `always` | 每次启动都尝试建表（使用 CREATE TABLE IF NOT EXISTS） |
| `embedded` | 仅在嵌入式数据库时自动建表（H2、HSQLDB） |
| `never` | 不自动建表，需要手动执行 SQL |

## 2.2 SPRING_SESSION 表

```sql
CREATE TABLE SPRING_SESSION (
    PRIMARY_ID            CHAR(36) NOT NULL,              -- Session 主键（UUID）
    SESSION_ID            CHAR(36) NOT NULL,              -- Session ID
    CREATION_TIME         BIGINT   NOT NULL,              -- 创建时间（epoch 毫秒）
    LAST_ACCESS_TIME      BIGINT   NOT NULL,              -- 最后访问时间
    MAX_INACTIVE_INTERVAL INT      NOT NULL,              -- 最大非活动间隔（秒）
    EXPIRY_TIME           BIGINT   NOT NULL,              -- 过期时间（epoch 毫秒）
    PRINCIPAL_NAME        VARCHAR(100),                   -- 用户名（索引字段）

    CONSTRAINT SPRING_SESSION_PK PRIMARY KEY (PRIMARY_ID),
    CONSTRAINT SPRING_SESSION_IX1 UNIQUE (SESSION_ID),
    INDEX SPRING_SESSION_IX2 (EXPIRY_TIME),
    INDEX SPRING_SESSION_IX3 (PRINCIPAL_NAME)
) ENGINE=InnoDB ROW_FORMAT=DYNAMIC;
```

**字段说明**：

| 字段 | 说明 |
|------|------|
| `PRIMARY_ID` | 内部主键，每次 Session 创建时生成的 UUID |
| `SESSION_ID` | 对外暴露的 Session ID，即 Cookie 中传递的值 |
| `EXPIRY_TIME` | 过期时间 = `LAST_ACCESS_TIME + MAX_INACTIVE_INTERVAL * 1000` |
| `PRINCIPAL_NAME` | 用户名，用于按用户查找 Session（索引） |

## 2.3 SPRING_SESSION_ATTRIBUTES 表

```sql
CREATE TABLE SPRING_SESSION_ATTRIBUTES (
    SESSION_PRIMARY_ID CHAR(36)      NOT NULL,   -- 关联 SPRING_SESSION.PRIMARY_ID
    ATTRIBUTE_NAME     VARCHAR(200)  NOT NULL,   -- 属性名称
    ATTRIBUTE_BYTES    BLOB          NOT NULL,   -- 属性值（序列化后的字节）

    CONSTRAINT SPRING_SESSION_ATTRIBUTES_PK
        PRIMARY KEY (SESSION_PRIMARY_ID, ATTRIBUTE_NAME),
    CONSTRAINT SPRING_SESSION_ATTRIBUTES_FK
        FOREIGN KEY (SESSION_PRIMARY_ID) REFERENCES SPRING_SESSION(PRIMARY_ID)
            ON DELETE CASCADE
) ENGINE=InnoDB ROW_FORMAT=DYNAMIC;
```

**实体关系**：

```
SPRING_SESSION (1) ────── (N) SPRING_SESSION_ATTRIBUTES
     PRIMARY_ID ─────────────── SESSION_PRIMARY_ID (FK)
     (CASCADE DELETE)
```

---

# 3. JdbcSessionRepository 源码分析

## 3.1 保存 Session

```java
// 简化版源码
@Override
public void save(JdbcSession session) {
    if (session.isNew()) {
        // INSERT 新记录
        jdbcOperations.update(
            "INSERT INTO SPRING_SESSION " +
            "(PRIMARY_ID, SESSION_ID, CREATION_TIME, LAST_ACCESS_TIME, " +
            "MAX_INACTIVE_INTERVAL, EXPIRY_TIME, PRINCIPAL_NAME) " +
            "VALUES (?, ?, ?, ?, ?, ?, ?)",
            session.getPrimaryId(),
            session.getId(),
            session.getCreationTime().toEpochMilli(),
            session.getLastAccessedTime().toEpochMilli(),
            session.getMaxInactiveInterval().getSeconds(),
            calculateExpiryTime(session),
            extractPrincipalName(session)
        );

        // INSERT 属性
        saveAttributes(session);
    } else if (session.isChanged()) {
        // UPDATE 现有记录
        jdbcOperations.update(
            "UPDATE SPRING_SESSION SET " +
            "SESSION_ID = ?, LAST_ACCESS_TIME = ?, " +
            "MAX_INACTIVE_INTERVAL = ?, EXPIRY_TIME = ?, " +
            "PRINCIPAL_NAME = ? " +
            "WHERE PRIMARY_ID = ?",
            session.getId(),
            session.getLastAccessedTime().toEpochMilli(),
            session.getMaxInactiveInterval().getSeconds(),
            calculateExpiryTime(session),
            extractPrincipalName(session),
            session.getPrimaryId()
        );

        // 增量更新属性（仅变更的属性）
        saveDeltaAttributes(session);
    }
}
```

## 3.2 查找 Session

```java
// 简化版源码
@Override
public JdbcSession findById(String id) {
    List<JdbcSession> sessions = jdbcOperations.query(
        "SELECT s.PRIMARY_ID, s.SESSION_ID, s.CREATION_TIME, " +
        "s.LAST_ACCESS_TIME, s.MAX_INACTIVE_INTERVAL, " +
        "s.EXPIRY_TIME, s.PRINCIPAL_NAME, " +
        "a.ATTRIBUTE_NAME, a.ATTRIBUTE_BYTES " +
        "FROM SPRING_SESSION s " +
        "LEFT JOIN SPRING_SESSION_ATTRIBUTES a " +
        "ON s.PRIMARY_ID = a.SESSION_PRIMARY_ID " +
        "WHERE s.SESSION_ID = ?",
        new JdbcSessionResultSetExtractor(),
        id
    );

    if (sessions.isEmpty()) {
        return null;
    }

    JdbcSession session = sessions.get(0);
    if (session.isExpired()) {
        deleteById(id);
        return null;
    }

    return session;
}
```

## 3.3 定时清理过期 Session

```java
// 默认实现：通过 @Scheduled 定时清理
@Scheduled(cron = "${spring.session.jdbc.cleanup-cron:0 * * * * *}")
public void cleanUpExpiredSessions() {
    int deletedCount = jdbcOperations.update(
        "DELETE FROM SPRING_SESSION WHERE EXPIRY_TIME < ?",
        System.currentTimeMillis()
    );
    if (deletedCount > 0) {
        logger.debug("Cleaned up " + deletedCount + " expired sessions");
    }
}
```

---

# 4. 配置详解

## 4.1 application.yml

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/session_db?useSSL=false&serverTimezone=UTC
    username: root
    password: root
    driver-class-name: com.mysql.cj.jdbc.Driver

  session:
    timeout: 30m
    jdbc:
      initialize-schema: always
      table-name: SPRING_SESSION           # 可自定义表名
      cleanup-cron: 0 * * * * *            # 每分钟清理一次
      flush-mode: on_save
      save-mode: on_set_attribute

# MySQL 连接池配置
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
```

## 4.2 编程式配置

```java
package com.example.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.session.jdbc.config.annotation.web.http.EnableJdbcHttpSession;

@Configuration
@EnableJdbcHttpSession(
    maxInactiveIntervalInSeconds = 1800,
    tableName = "SPRING_SESSION",
    cleanupCron = "0 * * * * *"
)
public class JdbcSessionConfig {
}
```

---

# 5. 不同数据库适配

## 5.1 MySQL

```sql
-- MySQL 建表语句 (InnoDB)
CREATE TABLE SPRING_SESSION (
    PRIMARY_ID CHAR(36) NOT NULL,
    SESSION_ID CHAR(36) NOT NULL,
    CREATION_TIME BIGINT NOT NULL,
    LAST_ACCESS_TIME BIGINT NOT NULL,
    MAX_INACTIVE_INTERVAL INT NOT NULL,
    EXPIRY_TIME BIGINT NOT NULL,
    PRINCIPAL_NAME VARCHAR(100),
    CONSTRAINT SPRING_SESSION_PK PRIMARY KEY (PRIMARY_ID)
) ENGINE=InnoDB ROW_FORMAT=DYNAMIC;
```

**MySQL 特殊注意**：
- 使用 InnoDB 引擎（支持事务和外键）
- `BLOB` 字段可能影响性能，尽量控制 Session 属性大小
- 定期 `OPTIMIZE TABLE` 回收空间
- 使用 `innodb_buffer_pool_size` 缓存热 Session 数据

## 5.2 PostgreSQL

```sql
-- PostgreSQL 建表语句
CREATE TABLE SPRING_SESSION (
    PRIMARY_ID CHAR(36) NOT NULL,
    SESSION_ID CHAR(36) NOT NULL,
    CREATION_TIME BIGINT NOT NULL,
    LAST_ACCESS_TIME BIGINT NOT NULL,
    MAX_INACTIVE_INTERVAL INT NOT NULL,
    EXPIRY_TIME BIGINT NOT NULL,
    PRINCIPAL_NAME VARCHAR(100),
    CONSTRAINT SPRING_SESSION_PK PRIMARY KEY (PRIMARY_ID)
);

CREATE TABLE SPRING_SESSION_ATTRIBUTES (
    SESSION_PRIMARY_ID CHAR(36) NOT NULL,
    ATTRIBUTE_NAME VARCHAR(200) NOT NULL,
    ATTRIBUTE_BYTES BYTEA NOT NULL,
    CONSTRAINT SPRING_SESSION_ATTRIBUTES_PK
        PRIMARY KEY (SESSION_PRIMARY_ID, ATTRIBUTE_NAME),
    CONSTRAINT SPRING_SESSION_ATTRIBUTES_FK
        FOREIGN KEY (SESSION_PRIMARY_ID)
        REFERENCES SPRING_SESSION(PRIMARY_ID) ON DELETE CASCADE
);
```

## 5.3 自定义表名

```yaml
spring:
  session:
    jdbc:
      table-name: MY_SESSION_TABLE
```

注意：自定义表名时需设置 `initialize-schema: never`，并手动创建对应名称的表。

---

# 6. 性能优化

## 6.1 索引策略

```sql
-- 核心索引（自动创建）
CREATE UNIQUE INDEX SPRING_SESSION_IX1 ON SPRING_SESSION (SESSION_ID);
CREATE INDEX SPRING_SESSION_IX2 ON SPRING_SESSION (EXPIRY_TIME);
CREATE INDEX SPRING_SESSION_IX3 ON SPRING_SESSION (PRINCIPAL_NAME);

-- 建议添加的复合索引（如果按用户分页）
CREATE INDEX SPRING_SESSION_IX4
    ON SPRING_SESSION (PRINCIPAL_NAME, EXPIRY_TIME DESC);
```

## 6.2 SQL 优化

```java
// 批量删除过期 Session（比逐条删除更高效）
@Scheduled(fixedRate = 60000)
public void batchCleanExpiredSessions() {
    jdbcTemplate.update(
        "DELETE FROM SPRING_SESSION WHERE EXPIRY_TIME < ? LIMIT 1000",
        System.currentTimeMillis()
    );
}
```

## 6.3 连接池配置

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 30           # Session 请求可能瞬间激增
      minimum-idle: 10
      idle-timeout: 300000
      connection-timeout: 5000        # 等待连接超时
```

## 6.4 属性大小控制

```java
// 避免在 Session 中存储大数据
// 仅存储关键标识，详细数据从数据库查

// 不推荐：
session.setAttribute("userFullProfile", largeUserObject);

// 推荐：
session.setAttribute("userId", userId);
session.setAttribute("roles", roles);
// 需要 profile 时通过 userId 查缓存或数据库
```

---

# 7. JDBC vs Redis 对比

| 维度 | JDBC | Redis |
|------|------|-------|
| **读性能** | ms 级（有索引） | μs 级（内存） |
| **写性能** | ms 级（磁盘写入） | μs 级（内存） |
| **扩展性** | 主从读写分离 | 集群分片 |
| **运维成本** | 低（共用业务数据库） | 中（独立 Redis 集群） |
| **数据一致性** | 强（ACID 事务） | 弱（最终一致性） |
| **过期处理** | 定时任务（秒级延迟） | TTL 自动过期 |
| **并发能力** | 数千 QPS | 数万 QPS |
| **序列化** | BLOB（JDK 序列化） | 可自定义 |
| **SQL 查询** | 支持 | 不支持 |
| **适用场景** | 中小规模应用 | 高并发应用 |
