本篇覆盖 MyBatis-Plus（MP）的高频面试题，按"主题块 → 详细分块 → 题目 → 解答"组织。五个主题块：定位与整合、通用 CRUD 与条件构造器、主键/自动填充与并发控制、插件拦截器体系、工程实践。MyBatis 本体见 6_mybatis.md，MySQL 与分库分表见 16_mysql.md、22_sharding_sphere.md。

# 1. 定位与整合

## 1.1 定位与选型

### 1.1.1 MyBatis-Plus 是什么？怎么理解"只做增强不做改变"？

**结论**：MP 是 MyBatis 的**增强工具**——不改写 MyBatis 任何既有行为，在其之上叠加"开箱即用"的能力：单表 CRUD 零 SQL（BaseMapper）、链式条件构造器（Wrapper）、分页/多租户/乐观锁等插件、雪花主键、自动填充、逻辑删除、代码生成器。

"只做增强不做改变"的落地形态：

1. **底层对象不变**：SqlSessionFactory、MapperProxy、Executor 四大对象、MappedStatement 注册表全是 MyBatis 原生那套（见 6_mybatis.md 1.1.3），MP 只是启动时**多注册**了一批 MappedStatement（见 1.2.1）、运行时**多挂**了一个拦截器 MybatisPlusInterceptor；
2. **原有代码原样运行**：已写的 XML、注解 SQL、PageHelper 继续生效，新旧共存、可渐进引入；
3. **行为可回退**：所有增强都由注解/配置显式声明（@TableId、@TableLogic、插件注册），不用即无感。

> **易错**：MP 不是"另一套 ORM"，面试不要把两者对立——它是 MyBatis 的超集生态；排查 MP 行为异常（分页没生效、逻辑删除没拼接）最终都要回到 MyBatis 的 MappedStatement 与拦截器机制去看。

### 1.1.2 MyBatis-Plus 和 MyBatis 有什么区别？如何选型？

| 维度 | MyBatis | MyBatis-Plus |
|------|---------|--------------|
| 单表 CRUD | 手写 SQL 或接 MBG/通用 Mapper | BaseMapper 零 SQL |
| 条件构造 | 手写动态 SQL 或 Example | Wrapper/LambdaWrapper 链式 |
| 分页 | 手写 LIMIT 或 PageHelper | PaginationInnerInterceptor 内置 |
| 主键策略 | 手动（useGeneratedKeys/selectKey） | ASSIGN_ID 雪花/AUTO/UUID 声明式 |
| 逻辑删除 | 手写 UPDATE + 每条查询加条件 | @TableLogic 自动改写 |
| 乐观锁 | 手写 WHERE version 与 SET version+1 | @Version + 插件自动处理 |
| 公共字段填充 | 手工 set | MetaObjectHandler 自动填充 |
| 多租户/动态表名 | SQL 手工拼 | 插件自动注入 |
| 代码生成 | MBG 外部工具 | 内置 Generator |
| 学习/维护成本 | 低（就是 MyBatis） | 引入一层抽象，排查问题要多懂 MP 机制 |

**选型**：新项目默认选 MP（单表场景开发效率提升立竿见影，团队几乎零迁移成本）；存量 MyBatis 项目渐进引入（只加依赖即可用 BaseMapper）；对 SQL 有极致管控、审计严格的核心报表仍可手写 XML——MP **不禁止**任何原生写法，两者混用是常态。多表复杂关联查询仍是手写 SQL 的主战场，MP 不生成 join。

### 1.1.3 MP、tk 通用 Mapper、JPA 怎么选？

三者解决同一个问题——"消灭单表 CRUD 样板代码"，路线不同：

- **tk.mybatis 通用 Mapper**：接口继承 `Mapper<T>` 获得 CRUD + Example 查询，**无分页/无乐观锁/无逻辑删除**，需要再配 PageHelper 等（见 6_mybatis.md 3.3.3）；注入方式是启动时动态生成方法实现，思路与 MP 同源。
- **MyBatis-Plus**：通用 Mapper 的能力 + 全家桶（分页、逻辑删除、乐观锁、多租户、填充、生成器、动态表名），Wrapper 表达力远超 Example，社区活跃度国内断层第一。
- **Spring Data JPA**：面向领域对象自动生成 SQL，省掉的是"映射层"而不是"SQL 层"——模型稳定、以对象操作为主的系统效率最高；复杂 SQL/执行计划调优要下沉 NativeQuery/QueryDSL，学习曲线陡。

**国内实践**：互联网业务系统 MP 为主流答案；老项目里见到 tk.mybatis 不奇怪（MP 出现之前的通用方案）；JPA 多见于海外团队与领域建模驱动的项目。

## 1.2 架构与配置

### 1.2.1 BaseMapper 不写 XML 就有 SQL，MP 是怎么注入的？

**结论**：容器启动阶段，MP 用 **ISqlInjector（注入器）** 把内置方法按"实体元数据 → SQL 模板 → MappedStatement"注册进 MyBatis 的 Configuration——和手写 XML 殊途同归（对应 MyBatis 的 MappedStatement 注册表，见 6_mybatis.md 1.2.4）。

流程拆解：

1. `TableInfoHelper` 解析实体：`@TableName/@TableId/@TableField/@TableLogic/@Version` + 驼峰规则 → 缓存成 **TableInfo**（表名、主键、字段列表、逻辑删除配置等）；
2. `DefaultSqlInjector.inspectInject()` 遍历 Mapper 接口方法，把 BaseMapper 声明的 17 个方法名（insert/deleteById/selectById/selectList/updateById/selectPage…）与 **AbstractMethod 实现**（Insert、SelectById…每个内置一个 SQL 模板类）匹配；
3. 每个命中方法生成 MappedStatement：SQL 由模板 + TableInfo 拼出（如 `SELECT %s FROM %s WHERE %s=#{id}`），参数/结果映射同样程序化构建；
4. **同名方法跳过**——你在 XML/注解里自己写了 `selectById`，MP 不再注入（自定义覆盖默认的官方机制）。

**扩展**：全局替换/追加注入方法 → 自定义 `ISqlInjector` Bean（继承 DefaultSqlInjector 重写 `getMethodList`，塞入自己的 AbstractMethod 子类）——MP 生态的"多数据源审计字段注入器""逻辑删除不查库插件"都从这里长出来。

> **易错**：注入只发生在**启动**时——实体改注解要重启；"Method not found in BaseMapper"报错多是接口没继承 BaseMapper 或注入器被自定义后漏方法。

### 1.2.2 Spring Boot 整合 MP 的步骤与关键配置？

```yaml
mybatis-plus:
  mapper-locations: classpath*:/mapper/**/*.xml
  type-aliases-package: com.example.entity
  configuration:
    map-underscore-to-camel-case: true   # MP starter 默认已开
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl  # 开发期打印 SQL
  global-config:
    banner: false
    db-config:
      id-type: assign_id                 # 全局主键策略
      table-prefix: t_                   # 实体省略前缀
      logic-delete-field: deleted        # 全局逻辑删除字段
      logic-delete-value: 1
      logic-not-delete-value: 0
spring:
  datasource:  # 普通 DataSource 配置即可（Hikari）
```

**步骤**：依赖 `mybatis-plus-boot-starter`（Boot3 用 `mybatis-plus-spring-boot3-starter`，3.5.5+）→ 上述 yml → 启动类 `@MapperScan` → 实体注解 + Mapper 继承 BaseMapper。自动配置链路与 MyBatis 整合同构（SqlSessionFactoryBean 被替换为 MybatisSqlSessionFactoryBean——**这就是 MP 能在启动期做方法注入的原因**，见 6_mybatis.md 3.3.1、10_springboot.md）。

**版本注意**：3.5.x 后 Generator、注解包独立（mybatis-plus-annotation / mybatis-plus-generator 单独依赖）；混用不同小版本（starter 3.4 + core 3.5）会出现"找不到类/方法签名不匹配"——统一 BOM 版本。

# 2. 通用 CRUD 与条件构造器

## 2.1 CRUD 分层封装

### 2.1.1 BaseMapper 提供了哪些方法？

按操作分组（全部零 SQL）：

| 类别 | 方法 |
|------|------|
| 插入 | `insert(T)` |
| 删除 | `deleteById(id)` / `deleteBatchIds(ids)` / `deleteByMap(Map)` / `delete(Wrapper)` |
| 更新 | `updateById(T)` / `update(T entity, Wrapper)` / `update(Wrapper)` |
| 查单条 | `selectById` / `selectOne(Wrapper)`（**多条命中会抛 TooManyResultsException**） |
| 查列表 | `selectList` / `selectBatchIds` / `selectByMap` / `selectObjs`（只取第一列） |
| 计数/存在 | `selectCount` / `exists(Wrapper)`（3.5.4+，生成 `SELECT 1 ... LIMIT 1`） |
| 分页 | `selectPage(IPage, Wrapper)` / `selectMapsPage` |
| Map 形态 | `selectMaps`（列名→值，适合聚合结果） |

**追问**：`selectOne` 查出多条？——底层 MyBatis `selectOne` 语义（见 6_mybatis.md），要么保证条件唯一（唯一索引），要么 `.last("LIMIT 1")`，要么改 selectList 取首条；用 Wrapper 的 `orderBy` 加 `LIMIT` 时注意 last 位置固定拼在 SQL 末尾。

### 2.1.2 IService/ServiceImpl 封装了什么？链式调用怎么用？

**`IService<T>`** = BaseMapper 全套 + 批量方法（saveBatch/updateBatchById/removeByIds/saveOrUpdateBatch，默认 1000 条一批 flush，见 5.1.1）+ 条件 count/exists + **链式入口**；**`ServiceImpl<M extends BaseMapper<T>, T>`** 提供默认实现（内部持有 baseMapper）。

链式 API（Service 与 Wrapper 融合，省去中间变量）：

```java
userService.lambdaQuery()
        .eq(User::getName, "张三")
        .gt(User::getAge, 18)
        .orderByDesc(User::getCreateTime)
        .last("LIMIT 10")
        .list();

userService.lambdaUpdate()
        .set(User::getStatus, 0)
        .eq(User::getId, id)
        .update();
```

**工程约定**：Controller 直接打 Service、Service 之间横向调用；**不要把 IService 当"贫血 CRUD 透传层"一路生成到 Controller**——业务规则、事务边界（@Transactional 标 Service 方法，见 7_spring.md 4.1.1）都收在 Service。

### 2.1.3 ActiveRecord 模式是什么？推荐使用吗？

实体继承 `Model<T>` 后自带 `insert()/updateById()/deleteById()/selectById()`：

```java
public class User extends Model<User> { ... }
new User().setName("张三").insert();
```

原理：Model 内部经 `SqlHelper` 静态持有 SqlSessionFactory 执行——**表对象自己会操作数据库**。

**结论：不推荐**。① 数据访问与领域对象耦合，违背分层（Mapper/Service 职责混在实体里）；② 静态持有 + 全局 SqlSessionFactory，单测不可 mock；③ 实体进 JSON 序列化、跨层传递时行为诡异；④ 官方态度也是"仅适合快速原型"。生产用 Mapper + Service（2.1.2）。

## 2.2 条件构造器 Wrapper

### 2.2.1 QueryWrapper 和 LambdaQueryWrapper 有什么区别？为什么推荐 Lambda？

| 对比 | QueryWrapper | LambdaQueryWrapper |
|------|--------------|--------------------|
| 字段引用 | 字符串 `"user_name"` | `User::getUserName` |
| 列名拼错 | 运行时 SQL 报错 | **编译期报错** |
| 属性改名重构 | 字符串漏改 | IDE 联动一起改 |
| 动态列名场景 | 支持（列名来自配置/参数） | 不便（需反射拼装） |
| 运行性能 | 相同 | 相同（Lambda 解析结果有缓存） |

```java
new QueryWrapper<User>().eq("name", "张三").gt("age", 18);
new LambdaQueryWrapper<User>().eq(User::getName, "张三").gt(User::getAge, 18);
```

Lambda 通过 **SerializedLambda** 反解方法引用 → 查 TableInfo 映射成真实列名（`@TableField("user_name")` 自动生效）。默认选 Lambda，仅"列名运行时才知道"的场景（动态报表、配置化查询）用字符串 Wrapper。UpdateWrapper/LambdaUpdateWrapper 对应 UPDATE 的 set + where。

### 2.2.2 Wrapper 常用方法有哪些？apply/or/last 有什么坑？

**条件族**：eq/ne/gt/ge/lt/le、between/notBetween、like/notLike/likeLeft(左%)/likeRight(右%)、isNull/isNotNull、in/notIn、ge/le 各支持 `column + value + mapping` 三参（如 `apply("create_time >= {0}", start)`）。

**结构控制（高频坑位）**：

```java
// 1. and/or 嵌套：or 只影响相邻一段，复杂组合显式套 and/nested
w.eq(User::getStatus, 1)
 .and(x -> x.like(User::getName, kw).or().like(User::getPhone, kw));
// 错误直觉：eq(status).like(name).or().like(phone) ——or 之后全链语义被改

// 2. 条件布尔重载：动态查询免 if-else
w.like(StringUtils.isNotBlank(kw), User::getName, kw)
 .eq(age != null, User::getAge, age);

// 3. orderBy/groupBy/having/select(指定列)/set(UpdateWrapper)
// 4. apply 原生片段：`apply("date_format(create_time,'%Y-%m') = {0}", month)`
//    —— 用 {0} 占位符走参数绑定；直接字符串拼用户输入 = SQL 注入（见 5.2.2）
// 5. last 只允许一次，追加在 SQL 最末尾（limit/for update），用户输入严禁进 last
```

**Wrapper 与实体混用**：`update(entity, wrapper)` 时 SET 部分取 entity 非 null 字段（受 updateStrategy，见 2.2.3），WHERE 取 wrapper——两套条件互不干扰；纯 Wrapper `set(User::getRemark, null)` 是唯一"稳妥把列更成 NULL"的方式（实体 null 默认被忽略）。

### 2.2.3 FieldStrategy（null 字段策略）是怎么设计的？

**问题场景**：`updateById(user)` 传了个 remark=null 的实体——"我不想更新这列"还是"我要把这列清空"？MP 的答案是 **FieldStrategy**：

| 策略 | 语义 |
|------|------|
| NOT_NULL（默认） | 字段为 null 就**不出现在 SQL**（INSERT 不插列、UPDATE 不进 SET、WHERE 不加条件） |
| NOT_EMPTY | null 和空串都忽略（String 有效） |
| IGNORED | 不判断，null 也照拼——`SET remark = null` |
| NOT_IGNORED / NEVER 等变体 | 按版本存在细微差异，理解"判不判、何时判"即可 |

三个维度独立配置：`insertStrategy / updateStrategy / whereStrategy`，@TableField 字段级或 global-config.db-config 全局级。

**清空字段的三种正确姿势**（默认 NOT_NULL 下 updateById 带 null 清不掉）：

1. `LambdaUpdateWrapper.set(User::getRemark, null)` ——**推荐**，意图显式；
2. 该字段 `@TableField(updateStrategy = IGNORED)` ——全局字段语义改"null 即清空"，影响所有更新；
3. 自定义 XML SQL。

**易错**：`wrapper.eq("name", null)` 不等价于"忽略"——行为随版本变化（旧版拼出 `name = null` 永假，新版直接校验抛异常），查空值永远写 `isNull()`；"更新丢字段"的第一排查项永远是 updateStrategy。

# 3. 主键、自动功能与并发控制

## 3.1 主键生成

### 3.1.1 MP 的主键策略有哪些？怎么选？

`@TableId(type = IdType.X)`（字段级优先于全局 db-config.id-type）：

| IdType | 语义 | 适用 |
|--------|------|------|
| ASSIGN_ID（默认） | **雪花算法**生成 Long（应用侧，插入前赋好） | 分布式系统首选，分库分表友好 |
| ASSIGN_UUID | 32 位 UUID 字符串（去横线） | 字段是 varchar 主键、接受无序随机 |
| AUTO | 数据库自增（JDBC getGeneratedKeys 回填，见 6_mybatis.md 2.1.6） | 单库单表、习惯自增 |
| INPUT | 业务自己 set id | 外部号段/老系统迁移 |
| NONE | 走全局配置 | —— |

**选型口径**：

- 互联网新项目默认 **ASSIGN_ID**：不依赖 DB 即可生成（写入前知道 id、分库分表路由可用 id、批量场景无回填竞态）；Long 型对索引与网络传输都友好；
- **AUTO 的隐性成本**：分库分表下"各库各自自增"会撞号（要靠号段/步长方案，见 22_sharding_sphere.md）；且强依赖 DB 交互拿 id；
- **UUID 作主键的性能代价**：无序 → InnoDB 主键页分裂、写放大（见 16_mysql.md），除非业务天然要求字符串主键，否则用雪花别用 UUID。

### 3.1.2 雪花算法的原理？MP 的实现有什么改良？时钟回拨怎么办？

**结构（64bit Long）**：

```text
0 | 41bit 毫秒时间戳(相对纪元) | 5bit datacenterId | 5bit workerId | 12bit 序列号
```

- 41bit ≈ **69 年**（MP 自定义纪元，默认沿用 2010-11-04）；
- 10bit 机器位 → 1024 节点；
- 12bit 序列号 → 同毫秒同节点 4096 个，超出**自旋等下一毫秒**；
- 整体**趋势递增**（时间为主序），对 B+ 树插入友好；纯内存生成、无网络开销、不依赖 DB——对比 UUID 无序、DB 自增有单点瓶颈、号段模式要发号服务（22_sharding_sphere.md、34_distributed_theory.md 有完整对比）。

**MP 改良（Sequence 类）**：workerId/datacenterId 不要求手工配置——由 **MAC 地址 + 进程 PID + 启动时间**哈希自动推导，容器化部署多副本"零配置不撞号"（理论上哈希有碰撞概率，副本数大时建议显式 `worker-id`/env 注入）。

**时钟回拨**：NTP 校时/虚拟机迁移可能让系统时钟倒退。MP Sequence 的处理：回拨**≤5ms** 自旋等待追平；回拨过大直接抛 `IllegalStateException: Clock moved backwards`——宁可失败不产重复号。工程兜底：机房 NTP 平滑步进（step 改 slew）、发号服务多时钟源校验；重度依赖方（订单号）再叠一层"启动时校验历史最大时间戳（持久化 lastTimestamp）"。

## 3.2 自动功能与并发

### 3.2.1 自动填充如何实现？如何拿到当前操作人？

**两步**：实体字段声明时机 + 全局填充器。

```java
@TableField(fill = FieldFill.INSERT)        private LocalDateTime createTime;
@TableField(fill = FieldFill.INSERT)        private String createBy;
@TableField(fill = FieldFill.INSERT_UPDATE) private LocalDateTime updateTime;
@TableField(fill = FieldFill.INSERT_UPDATE) private String updateBy;

@Component
public class MyMetaObjectHandler implements MetaObjectHandler {
    @Override
    public void insertFill(MetaObject metaObject) {
        this.strictInsertFill(metaObject, "createTime", LocalDateTime.class, LocalDateTime.now());
        this.strictInsertFill(metaObject, "createBy", String.class, UserContext.getUserId());
    }
    @Override
    public void updateFill(MetaObject metaObject) {
        this.strictUpdateFill(metaObject, "updateTime", LocalDateTime.class, LocalDateTime.now());
        // updateBy：注意 strict 只在"原值为空"时填；强制覆盖用 setFieldValByName
    }
}
```

**触发点**：填充发生在 MP 的参数处理阶段（MybatisParameterHandler），对**注入的内置方法**（insert/updateById 等）生效——**手写 XML 的 UPDATE 不会自动填充**（高频事故：加了 fill 注解但走自定义 SQL，字段没填）。

**操作人从哪来**：登录拦截器/JWT 过滤器把用户放 **ThreadLocal**（UserContext），fill 方法里读——注意线程池/MQ 消费/定时任务场景 ThreadLocal 为空要有兜底（"system"），异步链路要显式传递上下文（1_java.md 3.4）。也可 RequestContextHolder 拿 request（仅限同步 Web 线程）。

### 3.2.2 逻辑删除如何实现？有哪些注意事项？

**机制**：`@TableLogic`（或全局 logic-delete-field）后，**MP 注入的内置方法**自动改写 SQL：

```text
deleteById(1)   → UPDATE user SET deleted=1 WHERE id=1 AND deleted=0
selectById(1)   → SELECT ... WHERE id=1 AND deleted=0
updateById(...) → UPDATE ... WHERE id=1 AND deleted=0
```

删除后实体"查不到"；需要恢复时手工把 deleted 改回 0 或走自定义 SQL。

**注意事项（每条都是事故源）**：

1. **改写只覆盖注入方法与 Wrapper 查询**——手写 XML/注解 SQL **不会**自动加 `deleted=0`，过滤条件要自己写（与自动填充同理）；
2. **唯一索引冲突**：逻辑删除后行还在，同业务键重新注册会撞唯一约束。解法：联合唯一索引 `(biz_key, deleted)` 只解决"最多留一条已删"——常见强化是删除时把 deleted 置为 **id 或时间戳**（0 存活/非 0 且唯一），或 `(biz_key, deleted_at)`；
3. **索引与查询计划**：deleted 低基数（几乎全是 0）别单独建索引；高频查询用 `(tenant_id, deleted, create_time)` 类联合索引；
4. **数据量膨胀**：被"删"的数据永久占表——归档策略（定期搬历史表）与分表规模评估要提前做；
5. **审计需求**：记录删除人/时间用自动填充（updateBy/updateTime 在 deleteById 改写成的 UPDATE 中同样会填，逻辑删除的"更新"也走 fill）。

### 3.2.3 乐观锁如何实现？失败后业务怎么办？

**三步接入**：表加 version 列 → 实体 `@Version private Integer version` → 注册 `OptimisticLockerInnerInterceptor`。

```java
User u = userMapper.selectById(1L);      // version=2，必须"先查带出 version"
u.setName("新名字");
int rows = userMapper.updateById(u);     // UPDATE user SET name=?, version=3 WHERE id=? AND version=2
if (rows == 0) { /* 冲突：重查重试 / 提示刷新 / 记录失败 */ }
```

**关键机制与误区**：

1. **MP 不抛异常**——updateById 返回 0 就是冲突，**判定与处置是业务代码的责任**（旧资料"自动抛 OptimisticLockerException"是讹传）；
2. version 为 null 的实体走 updateById **不加乐观锁条件**（等于没开）——所以必须查询带回 version；UpdateWrapper 纯 set 更新同样绕过 @Version 语义；
3. 重试策略："有限次重读重算"适合低冲突（用户资料）；高冲突（秒杀库存）应改用**原子 UPDATE**（`SET stock=stock-? WHERE id=? AND stock>=?`，见 27_redis.md 库存题）——乐观锁循环重试在高竞争下是性能毒药；
4. 对比悲观锁 `SELECT ... FOR UPDATE`：读多写少用乐观、写冲突密集用悲观，判定标准是**冲突概率 × 回滚成本**（见 16_mysql.md 锁章节）。

## 3.3 类型映射

### 3.3.1 MP 的枚举如何映射到数据库？

**注解方案（推荐，3.5.x）**：

```java
public enum OrderStatus {
    PAID(1, "已支付"), REFUND(2, "已退款");
    @EnumValue private final int code;   // 库里存这个值
    private final String desc;
}
```

或实现 `IEnum<Integer>`（getValue() 返回落库值）。MP 把 **MybatisEnumTypeHandler** 设为默认枚举处理器：含 @EnumValue/IEnum 的枚举自动按"值字段"读写；纯 JDK 枚举退化为 EnumTypeHandler（按 name 存字符串）/EnumOrdinalTypeHandler（按序号）——**改枚举顺序会错位**，落库别用 ordinal。

**配置方案（旧）**：`type-enums-package` 批量扫描注册——3.5.x 已废弃移除，升级项目见到该配置要迁移到注解方案。

**配套细节**：Wrapper 条件 `.eq(User::getStatus, OrderStatus.PAID)` 走同一 handler 参数绑定；返回 Map/聚合查询里的枚举列按 JDBC 类型出来，不保证是枚举对象；前端契约用 code 还是 name 由序列化层统一（Jackson 配 `@JsonValue` 思路一致）。

### 3.3.2 JSON 字段怎么存？TypeHandler 在 MP 里怎么用？

**三段式配置**：

```java
@TableName(value = "t_order", autoResultMap = true)   // ① 必须！注入方法的 resultMap 才带 handler
public class Order {
    @TableField(typeHandler = JacksonTypeHandler.class) // ② 字段级处理器
    private List<Item> items;                            // MySQL JSON/TEXT 列
}
```

- 原理：JacksonTypeHandler 是 MyBatis TypeHandler 的 MP 内置实现（对象 ↔ JSON 字符串，见 6_mybatis.md 2.1.5），写入序列化、读取反序列化；
- **autoResultMap 忘开是第一大坑**——insert/update 正常、select 回来字段全 null；
- MP 自带：JacksonTypeHandler、FastjsonTypeHandler（历史）、GsonTypeHandler 可选，自定义（加密列、逗号分隔串 ↔ List）照 MyBatis 规范实现后同样挂 @TableField；
- **查询能力边界**：JSON 列条件要走数据库函数（MySQL `JSON_EXTRACT/JSON_CONTAINS`，apply 拼进 Wrapper；生成列 + 索引解决高频过滤）——"JSON 存对象"牺牲了该字段的 SQL 可查性，只用于"存取整块、少作条件"的扩展字段，别把主查询条件塞进 JSON。

# 4. 插件拦截器体系

## 4.1 插件架构

### 4.1.1 MybatisPlusInterceptor 是什么？InnerInterceptor 的执行顺序怎么安排？

**架构**：MP 3.4.0+ 把散装的旧插件（PaginationInterceptor、OptimisticLockerInterceptor…）统一成一个 **MybatisPlusInterceptor**——它本身是标准 MyBatis Interceptor（拦截 Executor 与 StatementHandler，织入方式见 6_mybatis.md 3.2.1），内部维护 `List<InnerInterceptor>`，在 `willDoQuery/willDoUpdate/beforeQuery/beforePrepare/beforeFinish` 等钩子上**按注册顺序逐个调用**。

**官方顺序约束（高频追问，答错直接暴露没写过配置）**：

1. **多租户、动态表名插件必须在分页插件之前**——两者的 SQL 改写（JSqlParser 加条件/换表名）必须先于分页拼 LIMIT 与生成 COUNT，否则 COUNT 统计的是"改写前"的 SQL；
2. **BlockAttackInnerInterceptor 放最后**——它是"守门员"，校验的应是最终 SQL 形态；
3. 乐观锁在 update 阶段独立生效，位置相对自由，惯例放中间。

```java
@Bean
public MybatisPlusInterceptor mpInterceptor() {
    MybatisPlusInterceptor it = new MybatisPlusInterceptor();
    it.addInnerInterceptor(new TenantLineInnerInterceptor(handler));     // 1 多租户
    it.addInnerInterceptor(new DynamicTableNameInnerInterceptor(map));   // 2 动态表名
    it.addInnerInterceptor(new OptimisticLockerInnerInterceptor());      // 3 乐观锁
    it.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL)); // 4 分页（改写链末尾）
    it.addInnerInterceptor(new BlockAttackInnerInterceptor());           // 5 全表攻击阻断
    return it;
}
```

**追问**：和 PageHelper 能共存吗？——都能拦截改写，但会互相干扰（PageHelper 在 StatementHandler.prepare 改写、MP 分页在 Executor 层改写），**二选一**；MP 项目直接用 PaginationInnerInterceptor（见 6_mybatis.md 3.2.2 对两者层次的对比）。

## 4.2 分页插件

### 4.2.1 PaginationInnerInterceptor 如何配置使用？原理是什么？

**配置**：`new PaginationInnerInterceptor(DbType.MYSQL)`，可设 `maxLimit`（单页上限，防 `pageSize=100000` 拖库）、`overflow`（翻过总页数返回空/回首页）。

**使用**：

```java
Page<User> page = new Page<>(1, 10);          // current, size
IPage<User> r = userMapper.selectPage(page,
        new LambdaQueryWrapper<User>().gt(User::getAge, 18));
// r.getRecords() / getTotal() / getCurrent() / getPages()
```

自定义 Mapper 方法同样支持：**第一个参数是 IPage** 即可被分页（XML 手写 SQL + Page 参数一样生效）。

**原理**（拦截 `Executor.query`，依赖 JSqlParser）：

1. `willDoQuery`：`page.searchCount=true` 时先执行 **COUNT 查询**（原 SQL 改写：去 ORDER BY、按需把 select 列换成 count(0)，join 可优化时只留最小表——optimizeCountSql/optimizeJoin）；total=0 直接短路不再查数据；
2. `beforeQuery`：按 **IDialect**（MySqlDialect/OracleDialect…）拼物理分页——MySQL `LIMIT ?,?`、Oracle ROWNUM 包装、PG `LIMIT ? OFFSET ?`；
3. 数据查询执行完把 records 与 total 回填 Page 返回。

> **易错**：DbType 不显式指定时靠 jdbc 连接串自动识别——连接池前置代理（如某些云代理）识别失败会拼错方言；分页与 RowBounds 逻辑分页的区别、深分页问题见 6_mybatis.md 3.2.3、16_mysql.md。

### 4.2.2 多表 JOIN 如何分页？COUNT 很慢怎么办？

```java
@Select("SELECT u.*, o.order_no FROM t_user u LEFT JOIN t_order o ON u.id=o.user_id " +
        "WHERE u.age > #{age}")
IPage<UserOrderVO> selectJoinPage(Page<UserOrderVO> page, @Param("age") Integer age);
// XML 同理：参数里带 IPage 即生效
```

要点：

1. JOIN 分页与单表机制相同（IPage 参数触发），**一对多结果集**行数被放大——分页语义是"数据库行数"不是"父对象数"，要按父对象分页请改**两段式**：主表分页查 id 列表 → IN 批量查子表 → 内存组装（见 6_mybatis.md 2.2.3）；
2. **COUNT 优化三板斧**：① `optimizeCountSql`（默认开）自动剥离 ORDER BY/简化 select 列；② `page.setSearchCount(false)` 关自动 count，用缓存/预估总数（瀑布流不需要精确 total）；③ 复杂 join 手写专用 COUNT SQL（`page.setSearchCount(false)` + 自己查总数，或 `countId` 指定 Mapper 里的 count 方法）；
3. `maxLimit` 兜底，禁止业务传超大 size 变相全表拉取。

### 4.2.3 分页插件解决不了什么问题？

**一句话**：它只自动化"写分页 SQL"，不改变 `LIMIT offset, n` 的 **O(offset) 代价**——MySQL 仍要扫过并丢弃前 offset 行（二级索引场景还要回表）。深分页方案（延迟关联/游标分页/ES，见 16_mysql.md）；以及 4.2.2 的行数语义、count 代价问题。插件之外还要治理：禁 SELECT *（分页大宽表行宽放大网络与内存）、管理端分页与 C 端分页策略不同（C 端滚动用游标/时间戳 keyset）。

## 4.3 其他内置插件与动态数据源

### 4.3.1 多租户插件如何实现数据隔离？

**方案谱系**：独立库（隔离最强、成本最高）→ 共享库独立 Schema → **共享表 + tenant_id 列**（主流 SaaS，MP 的 TenantLineInnerInterceptor 即此方案）→ 混合。

```java
public class MyTenantHandler implements TenantLineHandler {
    @Override public Expression getTenantId() {
        return new LongValue(TenantContext.getTenantId());   // ThreadLocal/请求上下文
    }
    @Override public String getTenantIdColumn() { return "tenant_id"; }
    @Override public boolean ignoreTable(String t) {
        return "sys_dict".equalsIgnoreCase(t);                // 公共表放行
    }
}
// 注册见 4.1.1；SELECT/UPDATE/DELETE 自动 AND tenant_id=?，INSERT 自动补列
```

**机制**：JSqlParser 解析每条 SQL 的表引用，注入租户条件。注意三点：

1. **手写 XML/注解 SQL 同样被改写**（与逻辑删除不同！租户/分页在 Executor 层对最终 SQL 动刀）——但复杂 SQL（子查询嵌套、UNION）解析失败会抛异常，上线前全量扫一遍自定义 SQL；
2. 租户上下文丢失 = 查错租户数据，**MQ 消费/定时任务**场景要显式按租户遍历（多租户批处理标准模式：外层循环租户、内层业务），异步传递上下文（参照 1_java.md 3.4）；
3. 与 27_redis.md 缓存 key、22_sharding_sphere.md 分片键组合成"租户维度全景隔离"设计。

### 4.3.2 如何防止误发全表 UPDATE/DELETE？

`BlockAttackInnerInterceptor`：JSqlParser 判定——UPDATE/DELETE **无 WHERE** 或 WHERE 恒真（`1=1`）→ 直接抛 `MybatisPlusException: 阻止全表操作`。

```java
it.addInnerInterceptor(new BlockAttackInnerInterceptor());  // 放链尾
```

**边界与配套**（只答插件不完整）：① 拦不住"WHERE 业务列 = 低区分度值"的准全表更新（`WHERE status=0` 覆盖 99% 行）——靠 SQL 审核平台（见 33_cicd.md 的 SQL 门禁）与影响行数上限告警；② 数据库账号权限收敛（应用账号无 DDL/DROP）；③ binlog 备份 + 延迟从库兜底误删恢复；④ 生产 DML 强制走工单审核。

### 4.3.3 动态表名插件是做什么的？

`DynamicTableNameInnerInterceptor` 按运行时上下文把 SQL 里的表名替换——典型场景**按月/按租户分表**（日志表 `t_log_202611`）、灰度影子表。

```java
DynamicTableNameInnerInterceptor dyn = new DynamicTableNameInnerInterceptor();
dyn.setTableNameHandler((sql, name) ->
    "t_log".equals(name) ? "t_log_" + MonthContext.current() : name);
```

**定位提醒**：它是"轻量表名路由"，不是分库分表中间件——没有跨库路由、没有聚合、没有分布式事务；真正分片用 ShardingSphere（见 22_sharding_sphere.md），本插件只服务"同库换表"的小场景。

### 4.3.4 @DS 注解如何实现多数据源与读写分离？有什么坑？

依赖 `dynamic-datasource-spring-boot-starter`（baomidou 出品，常与 MP 搭配）：yml 配 master/slave1/slave2 → `DynamicRoutingDataSource` 作为唯一 DataSource 注入（对 MyBatis 透明，见 10_springboot.md）→ `@DS("slave")` 标类或方法（方法优先）。

**原理**：AOP 环绕切面把数据源 key 压入 **ThreadLocal**（DynamicDataSourceContextHolder），`determineCurrentLookupKey()` 取 key 路由；支持 SPEL、Header 动态传（网关打标）、事务内整段锁定首个 key。

**高频坑（每条都炸过生产）**：

1. **与 @Transactional 同时标一个方法**：@DS 切面 order 默认高于事务——同方法上"先切库再开事务"没问题；但**事务已开启后**再调带 @DS 的内层方法**不生效**（连接已绑定 ThreadLocal，见 7_spring.md 4.1.1），要"一个事务内跨库"需 `@DSTransactional`（本地事务链表 + best-effort 提交，非 XA）——强一致走 Seata（见 21_seata.md）；
2. **主从复制延迟**：写后立读从库查不到（读己之写问题）——关键读强制走主（事务内自动走写库/`@DS("master")`/hint 强制路由），或等 GTID 同步；
3. 类上 @DS 与 Mapper 注入的 SqlSessionFactory 若多套并存，注意"数据源切换"作用在**连接**层而非会话层；
4. 连接池 per-datasource 独立配置，从库池太小 = 读放大打爆主库的错觉（容量规划）。

# 5. 工程实践

## 5.1 批量操作

### 5.1.1 saveBatch 和 foreach 批量插入有什么区别？怎么选？

| 维度 | saveBatch（IService） | foreach 多值 INSERT |
|------|----------------------|---------------------|
| SQL 形态 | 同一条 INSERT **PreparedStatement.addBatch**，JDBC 批量提交（BatchExecutor，见 6_mybatis.md 1.2.3/3.2.4） | 一条 SQL 拼 N 组 VALUES |
| 网络往返 | 默认 1 次批量 flush（1000 条/批） | 1 次 |
| SQL 长度风险 | 无单条超长（逐条参数） | 受 `max_allowed_packet` 限制 |
| 主键回填/复杂逻辑 | 每条独立走注入 SQL 与自动填充 | 同样支持 useGeneratedKeys 连续回填 |
| 性能差距 | **MySQL 不开 rewrite 时退化逐条发送，慢数倍**（5.1.2） | 天然最快 |

**选择**：常规万级以内、实体带复杂类型处理器/雪花 id → saveBatch 省心；几十万行导入 → foreach 分批（500~1000 行/批）或 JDBC Loader；"saveBatch 慢"先查驱动参数再换姿势。

### 5.1.2 saveBatch 为什么慢？如何调优？

**根因**：MySQL Connector/J 的 `addBatch()` 默认**并不真正打包**——executeBatch 仍逐条发送（每条一个 RTT）。批量提速的前提是 URL 加参数：

```text
jdbc:mysql://.../db?rewriteBatchedStatements=true
```

开启后驱动按批量大小改写：纯 INSERT 拼成多值 `INSERT ... VALUES (...),(...)`（接近 foreach 性能，吞吐常提升 **5~10 倍**）；UPDATE 打包成 multi-statement（需 `allowMultiQueries=true` 协同评估）。

**调优清单**：

1. `rewriteBatchedStatements=true`（批量场景必开）；
2. 分批 flush：saveBatch 默认 1000，超大导入配合 ExecutorType.BATCH 手控 `flushStatements + clearCache`（一级缓存膨胀，见 6_mybatis.md 3.2.4）；
3. **rewrite 的副作用**：`INSERT ... ON DUPLICATE KEY UPDATE` 改写策略不同（旧驱动有坑，版本回归测试）；批内异常粒度变粗（整条改写 SQL 失败回滚一批）——批量幂等要自己保证（唯一键 + 忽略重复）；
4. 更大数据量：LOAD DATA INFILE（16_mysql.md）或业务队列削峰。

## 5.2 工具与安全

### 5.2.1 代码生成器怎么用？生成哪些内容？

3.5.3+ 用 FastAutoGenerator（旧 AutoGenerator 逐步淘汰）：

```java
FastAutoGenerator.create("jdbc:mysql://localhost:3306/db", "root", "****")
    .globalConfig(b -> b.author("dev").outputDir("D:/gen").dateType(DateType.TIME_PACK))
    .packageConfig(b -> b.parent("com.example").entity("entity").mapper("mapper"))
    .strategyConfig(b -> b.addInclude("t_user", "t_order")
            .addTablePrefix("t_")
            .entityBuilder().enableLombok().enableTableFieldAnnotation()
                  .logicDeleteColumnName("deleted").versionColumnName("version")
            .mapperBuilder().enableBaseResultMap().enableBaseColumnList()
            .controllerBuilder().enableRestStyle())
    .templateEngine(new FreemarkerTemplateEngine())   // 默认 Velocity
    .execute();
```

产物：Entity（@TableName/@TableId/@TableField + 逻辑删除/乐观锁/填充注解）、Mapper 接口与 XML（BaseResultMap/Base_Column_List）、Service/Impl、Controller（RESTful 骨架）。**策略**：生成器负责"从表结构到可运行骨架"，二次开发在生成后手工迭代；重跑覆盖问题用"生成到独立目录 → diff 合并"的工程惯例；模板可自定义（公司规范注解、统一 Result 包装直接进模板）。

### 5.2.2 MyBatis-Plus 有哪些 SQL 注入攻击面？如何防？

**结论**：`#{}` 预编译是 MyBatis 层的安全（见 6_mybatis.md 2.1.1），但 MP/Wrapper 开了几个"原生 SQL 片段通道"，**每一个都要审**：

| 攻击面 | 危险写法 | 安全写法 |
|--------|---------|---------|
| `apply` | `apply("name = '" + input + "'")` | `apply("date_format(t,'%Y') = {0}", year)` 占位符 |
| `last` | `last("limit " + input)` | 数值先 parse + 范围校验再拼 |
| `orderBy` | 前端排序字段直通 `orderByAsc(input)` | **白名单映射**（枚举/Map：字段名→列名） |
| `setSql` | `setSql(input)` 任意片段 | 只接受常量字符串（内部拼装，非用户输入） |
| XML `${}` | 动态表名/排序拼接 | 白名单校验（6_mybatis.md 2.1.1） |
| 字段名入参 | Wrapper 列名来自前端 | 列名永远不来自用户输入 |

**纵深防御**：MP 对 Wrapper 的 order/group 列做了 JSqlParser 校验 + 新版 `SqlInjectionUtils.check`（命中关键词抛异常），但**白名单才是正解**；配合数据库账号最小权限、SQL 审计、WAF；代码评审清单：`grep -rn "apply\|last(\|orderBy" --include=*.java` 人工过一遍输入来源。