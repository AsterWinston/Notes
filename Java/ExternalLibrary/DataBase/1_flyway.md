# 1. 概述与简介

## 1.1 什么是 Flyway

Flyway 是一款开源的数据库迁移工具，由 Boxfuse 公司创建，现由 Redgate 维护。它使用版本控制的方式来管理数据库模式的变更，让数据库的演进像代码一样可追溯、可复现、可协作。

## 1.2 为什么需要数据库迁移

在团队开发中，数据库结构会随着需求不断变化。如果没有统一的迁移工具，会出现以下问题：

- 团队成员本地的数据库结构不一致
- 测试环境、预发布环境、生产环境的数据库不同步
- 缺少数据库变更的历史记录
- 回滚困难，无法快速恢复到某个历史版本
- 部署时容易遗漏 SQL 脚本

Flyway 通过版本化的迁移脚本解决了上述所有问题。

## 1.3 Flyway 的设计哲学

- **简单优先（Convention over Configuration）**：遵循命名约定即可工作，零配置启动
- **幂等性（Idempotent）**：相同版本只执行一次，保证可重复部署
- **可靠性（Reliability）**：通过校验和（checksum）检测脚本是否被篡改
- **可追溯（Accountability）**：`flyway_schema_history` 表记录每一次迁移的详细信息

## 1.4 Flyway 的工作原理

Flyway 的工作流程如下：

1. 检查 `flyway_schema_history` 表是否存在，不存在则自动创建
2. 扫描指定路径下的所有迁移脚本
3. 计算每个脚本的版本号和 checksum，与已执行的记录比对
4. 按版本号升序执行未执行过的迁移脚本
5. 每次执行成功后，在 `flyway_schema_history` 表中插入一条记录

```text
Application Start
      ↓
Check schema_history table → Not exist → Create it
      ↓
      Exist
      ↓
Scan migrations (classpath:db/migration)
      ↓
Compare with applied migrations
      ↓
Apply pending migrations (ordered by version)
      ↓
Update schema_history table
      ↓
Application Ready
```

# 2. 核心概念与架构

## 2.1 迁移类型

### 2.1.1 版本化迁移（Versioned Migration）

版本化迁移有明确的版本号，每个版本只会执行一次。命名格式为：`V<版本号>__<描述>.sql`

```text
V1__create_user_table.sql
V2__add_email_column.sql
V2_1__add_email_unique_index.sql
V3__create_order_table.sql
```

版本号的比较规则：
- 数字按数值比较：`V2` > `V1`，`V2.1` > `V2`
- 支持点分格式：`V1.2.3`、`V2024.01.15`
- 支持时间戳格式：`V20240115120000`

### 2.1.2 可重复迁移（Repeatable Migration）

可重复迁移没有版本号，每次 checksum 发生变化时都会重新执行。命名格式为：`R__<描述>.sql`

```text
R__create_views.sql
R__create_procedures.sql
R__grant_permissions.sql
```

典型使用场景：
- 视图（View）
- 存储过程（Stored Procedure）
- 函数（Function）
- 权限授予
- 物化视图刷新

可重复迁移**总是最后执行**，且在版本化迁移之后按文件名排序执行。

### 2.1.3 撤销迁移（Undo Migration — Flyway Teams 版）

撤销迁移用于回滚版本化迁移的变更。命名格式为：`U<版本号>__<描述>.sql`

```text
U1__create_user_table.sql   -- 撤销 V1
U2__add_email_column.sql    -- 撤销 V2
```

注意：撤销迁移是 Flyway Teams（商业版）的功能，开源版不支持。此外，撤销迁移存在破坏性，生产环境中建议使用"向前修复"策略而非回滚。

## 2.2 迁移命名约定

完整的文件名格式：

```
[Prefix][Version]__[Description].[Suffix]
  │          │            │           │
  V         1             create_user sql
```

| 部分 | 说明 | 示例 |
|------|------|------|
| Prefix | 迁移类型前缀 | `V`(版本化), `U`(撤销), `R`(可重复) |
| Version | 版本号 | `1`, `2.1`, `2024.01.15` |
| Separator | 分隔符 | `__`（两个下划线） |
| Description | 描述文字 | `create_user_table`, `add_email_column` |
| Suffix | 文件后缀 | `.sql`, `.java` |

命名规则要点：
- 版本号中的下划线会被视为点分隔符：`V2_1` 等同于 `V2.1`
- 描述中可以使用下划线或连字符，建议使用下划线
- 区分大小写，建议统一小写描述

## 2.3 校验和（Checksum）

Flyway 为每个已执行的迁移脚本计算 CRC32 校验和，并存储在 `flyway_schema_history` 表中。

```sql
-- flyway_schema_history 表结构（简化）
CREATE TABLE flyway_schema_history (
    installed_rank INT NOT NULL,
    version VARCHAR(50),
    description VARCHAR(200) NOT NULL,
    type VARCHAR(20) NOT NULL,      -- SQL, JDBC, SPRING_BATCH
    script VARCHAR(1000) NOT NULL,
    checksum INT,                    -- CRC32 校验和
    installed_by VARCHAR(100) NOT NULL,
    installed_on TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    execution_time INT NOT NULL,     -- 执行耗时（毫秒）
    success BOOLEAN NOT NULL,        -- 是否成功
    PRIMARY KEY (installed_rank)
);
```

校验失败的情况：
- 已执行过的 SQL 脚本被修改后再次运行，Flyway 会抛出 `ValidateException`
- 这保证了迁移脚本的不可篡改性
- 如果确实需要修改，应创建新的迁移脚本而非修改旧的

## 2.4 迁移脚本发现顺序

Flyway 按以下优先级扫描迁移脚本：

1. 文件系统路径（`locations` 配置）
2. classpath 路径（默认 `classpath:db/migration`）
3. 支持多个位置，按配置顺序扫描

版本化迁移的执行顺序严格按照版本号升序；可重复迁移在版本化迁移之后按文件名排序执行。

# 3. 配置详解

## 3.1 Spring Boot 配置

在 `application.yml` 中的完整配置：

```yaml
spring:
  flyway:
    # 是否启用 Flyway
    enabled: true
    
    # 迁移脚本位置（默认 classpath:db/migration）
    locations:
      - classpath:db/migration
      - filesystem:/opt/sql/migrations
    
    # 目标版本（迁移到指定版本为止）
    target: 10
    
    # 基线配置
    baseline-on-migrate: true    # 对已有数据库启用基线
    baseline-version: 1          # 基线版本号
    baseline-description: Base Migration
    
    # 校验控制
    validate-on-migrate: true    # 迁移前校验
    validate-migration-naming: true  # 校验命名规范
    
    # 连接信息（通常复用数据源，可独立配置）
    url: jdbc:mysql://localhost:3306/db
    user: root
    password: secret
    
    # 迁移表名
    table: flyway_schema_history
    
    # Schema 管理
    schemas:
      - public
      - app
    
    # 占位符替换
    placeholder-prefix: "${"
    placeholder-suffix: "}"
    placeholders:
      schema_name: public
    
    # 事务控制
    clean-on-validation-error: false  # 校验失败是否清库（生产慎用！！）
    
    # 乱序迁移
    out-of-order: false
    
    # 连接重试
    connect-retries: 5
    connect-retries-interval: 60
    
    # 编码
    encoding: UTF-8
    
    # 是否创建数据源
    create-data-source: true
    
    # 忽略迁移模式（忽略缺失的迁移）
    ignore-missing-migrations: false
    ignore-ignored-migrations: false
    ignore-pending-migrations: false
    ignore-future-migrations: false
```

## 3.2 核心参数详解

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `spring.flyway.enabled` | `true` | 是否启用 Flyway |
| `spring.flyway.locations` | `classpath:db/migration` | 迁移脚本路径 |
| `spring.flyway.baseline-on-migrate` | `false` | 对已有数据库执行基线 |
| `spring.flyway.baseline-version` | `1` | 基线版本号 |
| `spring.flyway.validate-on-migrate` | `true` | 迁移前校验 |
| `spring.flyway.out-of-order` | `false` | 允许乱序迁移 |
| `spring.flyway.clean-on-validation-error` | `false` | 校验失败是否清库 |
| `spring.flyway.ignore-missing-migrations` | `false` | 忽略缺失脚本 |
| `spring.flyway.table` | `flyway_schema_history` | 历史记录表名 |
| `spring.flyway.schemas` | — | 管理的 Schema 列表 |
| `spring.flyway.placeholder-prefix` | `${` | 占位符前缀 |
| `spring.flyway.placeholder-suffix` | `}` | 占位符后缀 |
| `spring.flyway.connect-retries` | `0` | 连接重试次数 |
| `spring.flyway.encoding` | `UTF-8` | SQL 文件编码 |

## 3.3 基线（Baseline）

基线用于将一个已有数据库纳入 Flyway 管理，而不必从头开始执行所有历史迁移。

```yaml
spring:
  flyway:
    baseline-on-migrate: true
    baseline-version: 5
    baseline-description: "Initial schema as of 2024-01-01"
```

基线的作用：
- 在 `flyway_schema_history` 中插入一条基线记录
- 标记当前数据库状态对应的版本号
- Flyway 只执行版本号大于基线版本的迁移
- 适合将 Flyway 引入现有项目

# 4. 高级特性

## 4.1 回调（Callbacks）

Flyway 提供了生命周期回调机制，允许在迁移的不同阶段执行自定义逻辑。

### 4.1.1 回调类型

```text
beforeMigrate      — 迁移开始前
beforeEachMigrate  — 每个迁移脚本执行前
afterEachMigrate   — 每个迁移脚本执行后
afterMigrate       — 迁移全部完成后
beforeValidate     — 校验前
afterValidate      — 校验后
beforeClean        — 清库前
afterClean         — 清库后
beforeInfo         — Info 命令前
afterInfo          — Info 命令后
beforeRepair       — Repair 命令前
afterRepair        — Repair 命令后
beforeBaseline     — 基线前
afterBaseline      — 基线后
beforeUndo         — 撤销前（Teams）
afterUndo          — 撤销后（Teams）
```

### 4.1.2 SQL 回调

在迁移脚本目录下创建以回调命名的 SQL 文件：

```text
db/migration/
├── beforeMigrate__log_start.sql
├── afterMigrate__send_notification.sql
├── V1__create_user.sql
└── V2__add_index.sql
```

### 4.1.3 Java 回调

实现 `Callback` 接口：

```java
public class MigrationCallback implements Callback {
    
    @Override
    public boolean supports(Event event, Context context) {
        return event == Event.AFTER_MIGRATE;
    }
    
    @Override
    public void handle(Event event, Context context) {
        System.out.println("Migration completed! Sending notification...");
        // 发送通知、记录日志、触发缓存刷新等
    }
    
    @Override
    public String getCallbackName() {
        return "notificationCallback";
    }
}
```

## 4.2 占位符替换（Placeholder Replacement）

Flyway 支持在 SQL 脚本中使用占位符，在迁移时动态替换。

### 4.2.1 配置占位符

```yaml
spring:
  flyway:
    placeholder-prefix: "${"
    placeholder-suffix: "}"
    placeholders:
      schema_name: my_app
      table_prefix: app_
      created_by_column: created_by
```

### 4.2.2 SQL 中使用占位符

```sql
-- V1__create_user.sql
CREATE TABLE ${schema_name}.${table_prefix}user (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    ${created_by_column} VARCHAR(100) NOT NULL
);
```

执行后实际生成的 SQL：

```sql
CREATE TABLE my_app.app_user (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    created_by VARCHAR(100) NOT NULL
);
```

### 4.2.3 禁用占位符替换

如果 SQL 中包含 `${}` 语法（如 Shell 脚本或特定模板），需要禁用：

```yaml
spring:
  flyway:
    placeholder-replacement: false
```

## 4.3 多 Schema 模式

Flyway 支持同时管理多个 Schema：

```yaml
spring:
  flyway:
    schemas:
      - public
      - app
      - reporting
```

多 Schema 时，`flyway_schema_history` 表默认创建在第一个 Schema 中，所有迁移脚本针对每个 Schema 依次执行。

## 4.4 混合模式（Mixed Mode）

混合模式允许同时执行版本化迁移和可重复迁移。默认情况下，一次迁移中如果既有版本化迁移又有可重复迁移，它们会按照合理顺序执行：

1. 先执行所有待执行的版本化迁移（按版本号升序）
2. 再执行所有待执行的可重复迁移（按文件名排序）

## 4.5 干运行（Dry Run）

Flyway Teams 版支持干运行模式，预览将要执行的 SQL 而不实际执行：

```bash
flyway migrate -dryRunOutput=dryrun.sql
```

开源版中可以通过设置 `spring.flyway.dry-run-output` 实现类似效果。

## 4.6 Java 迁移

除了 SQL 迁移，Flyway 还支持基于 Java 的迁移，适合需要在迁移中执行复杂逻辑的场景。

### 4.6.1 实现 Java 迁移

```java
public class V1_2__AddDefaultUsers extends BaseJavaMigration {
    
    @Override
    public void migrate(Context context) throws Exception {
        // 获取 JDBC 连接
        try (PreparedStatement statement = 
                context.getConnection().prepareStatement(
                    "INSERT INTO users (name, email) VALUES (?, ?)")) {
            
            statement.setString(1, "admin");
            statement.setString(2, "admin@example.com");
            statement.executeUpdate();
            
            // 读取外部数据
            List<User> users = loadUsersFromFile();
            for (User user : users) {
                statement.setString(1, user.getName());
                statement.setString(2, user.getEmail());
                statement.executeUpdate();
            }
        }
    }
}
```

### 4.6.2 Spring JDBC 迁移

Flyway 提供了 Spring JDBC 支持：

```java
public class V1_3__InitData extends BaseSpringJavaMigration {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    @Override
    public void migrate(Context context) {
        // 使用 JdbcTemplate 执行复杂逻辑
        jdbcTemplate.execute("INSERT INTO config (key, value) VALUES ('version', '1.0')");
        
        // 调用 Service 层
        userService.initDefaultRoles();
    }
}
```

# 5. 底层原理

## 5.1 Flyway Schema History 表详解

`flyway_schema_history` 表是整个迁移系统的核心元数据表，每次迁移执行后都会插入一条记录。

```sql
-- Flyway 实际创建的完整表结构
CREATE TABLE IF NOT EXISTS flyway_schema_history (
    installed_rank INT NOT NULL,
    version VARCHAR(50),
    description VARCHAR(200) NOT NULL,
    type VARCHAR(20) NOT NULL,          -- SQL, JDBC, SPRING_BATCH
    script VARCHAR(1000) NOT NULL,
    checksum INT,
    installed_by VARCHAR(100) NOT NULL,
    installed_on TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    execution_time INT NOT NULL,
    success BOOLEAN NOT NULL,
    PRIMARY KEY (installed_rank)
);

-- 查询迁移历史
SELECT installed_rank, version, description, type, 
       script, checksum, execution_time, success
FROM flyway_schema_history
ORDER BY installed_rank;
```

字段说明：

| 字段 | 说明 |
|------|------|
| `installed_rank` | 执行顺序，从1递增 |
| `version` | 版本号，可重复迁移为 NULL |
| `description` | 迁移描述 |
| `type` | 迁移类型：SQL/JDBC/SPRING_BATCH |
| `script` | 脚本文件名 |
| `checksum` | CRC32 校验和，用于检测脚本篡改 |
| `installed_by` | 执行迁移的数据库用户 |
| `installed_on` | 执行时间戳 |
| `execution_time` | 执行耗时（毫秒） |
| `success` | 是否执行成功 |

## 5.2 迁移执行流程源码分析

Flyway 迁移的核心逻辑在 `Flyway` 类的 `migrate()` 方法中：

```java
// Flyway 迁移流程（简化伪代码）
public MigrateResult migrate() {
    // 1. 创建 schema_history 表（如果不存在）
    schemaHistory.create();
    
    // 2. 校验当前状态
    if (validateOnMigrate) {
        validate();
    }
    
    // 3. 查找所有可用迁移
    List<MigrationInfo> available = scanner.findMigrations();
    
    // 4. 获取已应用的迁移
    List<MigrationInfo> applied = schemaHistory.allApplied();
    
    // 5. 计算待执行迁移（已应用但 checksum 不匹配 → 报错）
    List<MigrationInfo> pending = calculatePending(available, applied);
    
    // 6. 按版本号排序
    pending.sort(Comparator.comparing(MigrationInfo::getVersion));
    
    // 7. 依次执行
    for (MigrationInfo migration : pending) {
        // 7.1 再次校验（防止并发冲突）
        validatePending(migration);
        
        // 7.2 开始事务
        connection.setAutoCommit(false);
        
        try {
            // 7.3 执行迁移
            ExecutorFactory.createExecutor(migration).execute(connection);
            
            // 7.4 记录 schema_history
            schemaHistory.addApplied(migration);
            
            // 7.5 提交事务
            connection.commit();
        } catch (Exception e) {
            connection.rollback();
            throw new FlywayException("Migration failed!", e);
        }
    }
}
```

## 5.3 锁机制与并发安全

Flyway 使用数据库锁来防止多个实例同时执行迁移：

```sql
-- Flyway 获取表级锁（MySQL 示例）
LOCK TABLES flyway_schema_history WRITE;

-- 或使用行级锁（PostgreSQL）
SELECT * FROM flyway_schema_history FOR UPDATE;
```

锁的作用：
- 防止微服务多实例同时启动时产生并发冲突
- 确保迁移串行执行
- 加锁超时会抛出异常，等待重试

## 5.4 校验机制

校验分为三个粒度：

```java
// 1. 版本校验：检查是否有重复版本
if (applied.version().equals(pending.version())) {
    throw new FlywayException("Duplicate version");
}

// 2. 校验和校验：检查已执行脚本是否被修改
if (applied.checksum() != computeChecksum(pending.script())) {
    throw new FlywayException("Checksum mismatch");
}

// 3. 描述校验：检查描述是否一致
if (!applied.description().equals(pending.description())) {
    throw new FlywayException("Description mismatch");
}
```

# 6. 性能优化

## 6.1 迁移脚本优化

- **合并小版本**：频繁的微小变更尽量合并，减少迁移数量
- **批量操作**：大量数据操作使用批量 DML 而非逐行
- **索引创建优化**：大表建索引时使用 `CONCURRENTLY`（PostgreSQL）

```sql
-- 批量插入优化
INSERT INTO users (name, email) VALUES
('user1', 'u1@example.com'),
('user2', 'u2@example.com'),
('user3', 'u3@example.com');
```

## 6.2 启动时间优化

- **设置目标版本**：用 `target` 限制迁移范围
- **禁用校验**：生产环境稳定后可考虑 `validate-on-migrate: false`
- **异步迁移**：大项目可考虑将迁移独立于应用启动

```yaml
spring:
  flyway:
    target: 50  # 只执行到版本50，跳过后续版本
```

## 6.3 大表迁移策略

对于大表的 DDL 操作：

```sql
-- 分批迁移策略（MySQL 大表加列）
-- Step 1: 创建新表
CREATE TABLE users_new LIKE users;

-- Step 2: 添加新列
ALTER TABLE users_new ADD COLUMN email VARCHAR(100);

-- Step 3: 分批迁移数据（使用 pt-online-schema-change 等工具）
INSERT INTO users_new SELECT id, name, email FROM users WHERE id BETWEEN ? AND ?;

-- Step 4: 替换表
RENAME TABLE users TO users_old, users_new TO users;
```

# 7. 与 Spring Boot 集成

## 7.1 基本集成

Spring Boot 集成 Flyway 极其简单：

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>

<!-- MySQL 需要额外依赖 -->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-mysql</artifactId>
</dependency>
```

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb
    username: root
    password: secret
  
  flyway:
    enabled: true
    locations: classpath:db/migration
```

## 7.2 迁移脚本目录结构

```text
src/main/resources/
└── db/
    └── migration/
        ├── V1__create_user_table.sql
        ├── V2__add_email_column.sql
        ├── V2_1__add_email_index.sql
        ├── V3__create_order_table.sql
        ├── V3_1__add_order_item_table.sql
        └── R__create_user_view.sql
```

## 7.3 多数据源配置

```java
@Configuration
public class FlywayConfig {
    
    @Bean
    @ConfigurationProperties(prefix = "datasource.user")
    public DataSource userDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    @ConfigurationProperties(prefix = "datasource.order")
    public DataSource orderDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    public FlywayConfigurationCustomizer userFlywayCustomizer() {
        return configuration -> {
            configuration.setDataSource(userDataSource());
            configuration.setLocations("classpath:db/migration/user");
            configuration.setSchemas("user_db");
        };
    }
    
    @Bean
    public FlywayConfigurationCustomizer orderFlywayCustomizer() {
        return configuration -> {
            configuration.setDataSource(orderDataSource());
            configuration.setLocations("classpath:db/migration/order");
            configuration.setSchemas("order_db");
        };
    }
}
```

## 7.4 编程式集成

如果需要更精细的控制，可以编程式配置：

```java
@Configuration
public class FlywayManualConfig {
    
    @Bean
    public Flyway flyway(DataSource dataSource) {
        return Flyway.configure()
            .dataSource(dataSource)
            .locations("classpath:db/migration")
            .baselineOnMigrate(true)
            .outOfOrder(false)
            .encoding(StandardCharsets.UTF_8)
            .load();
    }
    
    @EventListener(ApplicationReadyEvent.class)
    public void migrate() {
        flyway.migrate();
    }
}
```

## 7.5 Spring Boot 自动配置原理

Spring Boot 通过 `FlywayAutoConfiguration` 实现自动配置：

```java
@AutoConfiguration
@ConditionalOnClass(Flyway.class)
@ConditionalOnProperty(prefix = "spring.flyway", name = "enabled", 
                       matchIfMissing = true)
public class FlywayAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean(Flyway.class)
    public Flyway flyway(FlywayProperties properties, 
                         DataSource dataSource,
                         List<FlywayConfigurationCustomizer> customizers) {
        
        FluentConfiguration configuration = Flyway.configure()
            .dataSource(dataSource)
            .locations(properties.getLocations().toArray(new String[0]))
            .baselineOnMigrate(properties.isBaselineOnMigrate())
            .encoding(properties.getEncoding());
        
        // 应用自定义配置
        customizers.forEach(c -> c.customize(configuration));
        
        return configuration.load();
    }
    
    // 监听 ContextRefreshedEvent 事件触发迁移
    @EventListener
    public void migrate(Flyway flyway) {
        flyway.migrate();
    }
}
```

# 8. 最佳实践与常见问题

## 8.1 最佳实践

### 8.1.1 迁移脚本规范

- **每个脚本只做一个变更**：单一职责原则，方便排查问题
- **脚本不可逆**：一旦提交就不能修改，必须通过新脚本修复
- **版本号连续不跳跃**：避免版本冲突，方便 Code Review
- **描述清晰**：用描述说明变更目的

### 8.1.2 团队协作规范

```text
1. 分支开发流程
   - 每个 Feature 分支在 db/migration 下创建迁移脚本
   - 版本号使用时间戳避免冲突：V202401151200__add_field.sql
   - 合并到主分支时统一修改版本号

2. Code Review 要点
   - 检查 SQL 语法正确性
   - 检查是否包含 DROP 操作（生产环境需谨慎）
   - 检查索引创建是否合理
   - 检查数据迁移是否可回滚

3. 发布流程
   - 先在预发布环境验证迁移
   - 生产环境部署时确保有数据库备份
   - 监控迁移执行时间，超长迁移需提前规划
```

### 8.1.3 生产环境注意事项

- **永远不要在生产环境使用 `clean`**：会清空所有数据
- **大表迁移选在低峰期**：避免锁表影响业务
- **预留回退方案**：准备回滚脚本或数据备份
- **监控迁移耗时**：超过阈值发出告警
- **只增不改**：「向前兼容」是最高准则

## 8.2 常见问题

### 8.2.1 校验失败

```
ERROR: Validate failed: Migrations have failed validation
Detected applied migration not resolved locally: V3
```

解决方案：
```yaml
spring:
  flyway:
    ignore-missing-migrations: true  # 忽略缺失的迁移
```

### 8.2.2 Checksum 不匹配

原因：已执行的迁移脚本被修改。
解决方案：
1. 恢复原始脚本
2. 使用 `flyway repair` 修复校验和
3. 创建新的迁移脚本

```bash
# 修复校验和不匹配（重置 checksum 为当前值）
flyway repair
```

### 8.2.3 乱序迁移

场景：分支 A 创建了 V5，分支 B 创建了 V6，但 A 先合并发布，B 后合并。

解决方案：
```yaml
spring:
  flyway:
    out-of-order: true  # 允许乱序
```

但长期建议通过版本号管理避免乱序。

### 8.2.4 迁移耗时过长

处理方法：
- 使用 `target` 分批迁移
- 大表 DDL 使用在线工具（pt-osc, gh-ost）
- 考虑异步迁移方案

### 8.2.5 多实例并发启动

场景：Kubernetes 环境中多个 Pod 同时启动。
处理方式：Flyway 的数据库锁机制会自动处理，无需额外配置。

## 8.3 与 Liquibase 对比

| 维度 | Flyway | Liquibase |
|------|--------|-----------|
| 核心理念 | 约定优于配置（SQL优先） | XML/YAML/JSON/JSON5 描述变更 |
| 学习曲线 | 低 | 中 |
| 变更格式 | 原生 SQL | XML/JSON/YAML/SQL |
| 版本控制 | 文件名版本号 | XML/JSON/YAML 中的 changeSet ID |
| 回滚支持 | Teams 版支持 | 开源版支持 |
| 数据库兼容性 | 广泛 | 广泛 |
| 社区活跃度 | 高 | 高 |
| Spring Boot 集成 | 原生支持 | 原生支持 |
| CI/CD 友好度 | 高（原生 SQL） | 中 |

选择建议：
- 团队熟悉 SQL 且追求简洁：选择 Flyway
- 需要精细控制变更顺序和回滚：选择 Liquibase
- 需要多数据库平台支持：两者均可

# 9. 面试要点

## 9.1 基础问题

**Q: 为什么需要数据库迁移工具？**
A: 解决团队开发中数据库版本不一致、环境同步困难、变更加历史可追溯的问题，让数据库变更像代码一样可控。

**Q: Flyway 迁移脚本有哪几种类型？命名规则是什么？**
A: 三种：版本化迁移（V1__desc.sql）、可重复迁移（R__desc.sql）、撤销迁移（U1__desc.sql，Teams版）。

**Q: Flyway 的校验和机制是什么？**
A: 对已执行的迁移脚本计算 CRC32 checksum 并存入 schema_history 表。下次运行时重新计算对比，若不一致则抛出异常，防止脚本被篡改。

**Q: `flyway_schema_history` 表有哪些关键字段？**
A: installed_rank(执行顺序), version(版本号), description(描述), type(类型), script(脚本名), checksum(校验和), execution_time(耗时), success(是否成功)。

**Q: 如何将 Flyway 引入已有数据库的项目？**
A: 配置 `baseline-on-migrate: true` 和 `baseline-version`，Flyway 会创建基线记录，后续只执行高于基线版本的迁移。

## 9.2 进阶问题

**Q: Flyway 如何保证多实例并发安全？**
A: 通过数据库锁（如 MySQL LOCK TABLES 或 PostgreSQL SELECT FOR UPDATE）实现，确保同一时间只有一个实例执行迁移。

**Q: 如果一个迁移脚本执行到一半失败了，Flyway 如何处理？**
A: 整个迁移在事务中执行，失败时自动回滚。schema_history 表不会记录失败的迁移。但如果是 DDL 语句（MySQL 隐式提交），则无法回滚。

**Q: 如何处理团队开发中分支合并导致的版本冲突？**
A: 三种方式：1）使用时间戳作为版本号 2）设置 `out-of-order: true` 允许乱序 3）使用可重复迁移代替版本化迁移。

**Q: Java 迁移和 SQL 迁移各有什么优缺点？**
A: SQL 迁移：简单、数据库无关、易于 Review。Java 迁移：适合复杂逻辑（数据处理、调用外部API）、可编程性更强，但需要编译和部署。

## 9.3 设计思路题

**Q: 设计一个支持多环境的数据库迁移系统需要考虑哪些因素？**

A:
1. **版本管理**：统一版本号规范，避免冲突
2. **环境隔离**：开发/测试/预发布/生产环境使用不同配置
3. **安全控制**：生产环境禁止 DROP、TRUNCATE 等危险操作
4. **回滚策略**：准备回退脚本或利用备份恢复
5. **监控告警**：迁移失败及时告警
6. **权限管理**：数据库账号最小权限原则
7. **自动化**：集成 CI/CD 流水线自动执行迁移
8. **文档同步**：ER 图、数据字典随迁移自动更新

**Q: Flyway 和 Liquibase 的设计理念有何不同？**

A: Flyway 遵循"约定优于配置"，以原生 SQL 为基础，学习成本低，强调简单直接。Liquibase 使用抽象描述（XML/YAML），通过 changeSet 管理变更，提供更丰富的数据库无关性和变更追踪能力，但也带来了更高的复杂度。选择取决于团队偏好和项目需求。
