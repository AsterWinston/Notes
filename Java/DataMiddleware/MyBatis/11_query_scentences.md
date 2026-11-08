# 1. 返回Bean

**查询结果用什么类型接收，取决于结果的"形状"**。有对应实体类、且能保证只有一条记录（主键/唯一键查询）时，可以直接用实体类接收，也可以用 List 集合接收。

要警惕反向情形：**方法声明为实体类，SQL 却查出多条**，MyBatis 会抛 TooManyResultsException。另外查不到数据时返回的是 null 而不是空对象，调用方必须判空。

# 2. 返回List

当查询的记录条数多于一条时，**必须使用集合接收**，用单个实体类接收会抛 TooManyResultsException。反直觉的约定：**结果集为空时 MyBatis 返回空 List（size=0）而不是 null**，`cars.size()` 可放心调用，`cars.get(0)` 必须先判 isEmpty。

# 3. 返回Map

当返回的数据没有合适的实体类对应时——典型如统计报表、字段未定型的聚合查询——可以采用 Map 接收：**字段名做 key，字段值做 value**，resultType 直接写 `map`。能保证只有一条数据时返回一个 Map 即可。代价是放弃类型安全：key 拼错不会编译报错，只在运行期取出 null；DECIMAL 列映射成 BigDecimal 而非 Double，直接强转抛 ClassCastException。

> **注意**：Map 接收适合报表/导出等一次性场景，业务主链路应尽早沉淀 DTO；数据库列名变更时 Map 的 key 静默改变，所有取值点失效且不报错。

# 4. 返回List

查询结果条数大于等于一条时，可以返回一个存储 Map 的 List 集合（`List<Map<String, Object>>`，resultType 仍写 `map`），类型丢失问题同第 3 节。

# 5. 返回Map<String,Map>（@MapKey）

想把某列的值当外层 key、整行当 value、按 key 直接取行时，用 **@MapKey 指定 key**。拿 Car 的 id 做 key，取出对应 Map 时更方便。

```java
/**
 * 获取所有的Car，返回一个Map集合。
 * Map集合的key是Car的id，value是对应行的Map数据。
 * 需要在接口上添加@MapKey注解，指定id作为map的key
 */
@MapKey("id")
Map<Long, Map<String, Object>> selectAllRetMap();
```

```xml
<select id="selectAllRetMap" resultType="map">
    select id,
           car_num      carNum,
           brand,
           guide_price  guidePrice,
           produce_time produceTime,
           car_type     carType
    from t_car
</select>
```

其中 MapKey 属于 `import org.apache.ibatis.annotations.MapKey;`。易混点：**@MapKey 的 key 类型由列的实际值决定**，BIGINT 列映射出 Long，写成 `Map<Integer, ...>` 会在取值时炸；`Map<String, Car>` 也是合法签名（@MapKey + resultType="car"），外层 key 取的是实体属性值。

# 6. resultMap结果映射

查询结果的列名和 Java 属性名对应不上时（`car_num` 对不上 `carNum`，该列就是 null），三种解法：**as 起别名**、**resultMap 结果映射**、**开启驼峰命名自动映射**。

## 6.1 使用as起别名

在 XxxMapper.xml 的 SQL 中 `car_num as carNum` 显式对齐。优点是局部、直观；缺点是每个查询重复写列清单，可配合 `<sql>` 片段复用。

## 6.2 使用resultMap进行结果映射

```xml
<!--
    resultMap:
        id:这个结果映射的标识，作为select标签的resultMap属性的值。
        type:结果集要映射的类。可以使用别名。
        property：POJO类的属性名
        column：数据库表的列名
-->
<resultMap id="carResultMap" type="${bean全限定类名}">
    <!--对象的唯一标识，如果有主键，建议使用id标签，官方解释是：为了提高mybatis的性能，建议写上，虽然说也可以用普通的result标签进行映射-->
    <id property="id" column="id"/>
    <result property="carNum" column="car_num"/>
    <!--当属性名和数据库列名一致时，可以省略，但是建议都写上-->
    <!--javaType用来指定属性类型。jdbcType用来指定列类型，一般可以省略-->
    <result property="brand" column="brand" javaType="string" jdbcType="VARCHAR"/>
    <result property="guidePrice" column="guide_price"/>
    <result property="produceTime" column="produce_time"/>
    <result property="carType" column="car_type"/>
</resultMap>

<!--resultMap属性的值必须和resultMap标签中id属性值一致。-->
<select id="selectAllByResultMap" resultMap="carResultMap">
    select * from t_car
</select>
```

**注意：resultMap 和 resultType 不要同时出现，resultMap 优先级大于 resultType**。resultMap 是声明式的"列→属性"契约，也是嵌套映射（association/collection，见 13 篇）的唯一载体；表结构变更改一处 resultMap 全体受益。`<id>` 不改变映射结果，但决定哪些行合并为同一对象，主键场景建议写上。

## 6.3 驼峰命名自动映射

前提：**属性名遵循 Java 小驼峰，列名遵循 SQL 全小写下划线**。在 mybatis-config.xml 中开启：

```xml
<settings>
    <setting name="mapUnderscoreToCamelCase" value="true"/>
</settings>
```

// underscore：下划线，强调

settings 放在 properties 之后、typeAlias 之前（DTD 顺序约束）。开启后 `car_num` 自动映射 `carNum`，绝大多数 SQL 不用再写别名或 resultMap。

**设计考量**——三种方式取舍：

| 方式 | 作用范围 | 维护成本 | 适用 |
|------|----------|----------|------|
| as 别名 | 单条 SQL | 每条都要写 | 个别列对不上、临时查询 |
| resultMap | 声明它的语句 | 声明一次多处引用 | 列名怪异、嵌套映射、复杂结果 |
| 驼峰开关 | 全局 | 零 | 命名规范统一的项目（**推荐默认开**） |

> **行业实践**：新项目全局开驼峰并约定命名规范；存量项目里列名不规范的表用局部 resultMap 打补丁，不要为个别烂列名关掉全局开关。

# 7. 返回总记录条数

分页前置动作：查总数。

```xml
<select id="selectTotal" resultType="long">
    select count(*) from t_car
</select>
```

```java
@Test
public void testSelectTotal() {
    CarMapper carMapper = SqlSessionUtil.openSession().getMapper(CarMapper.class);
    Long total = carMapper.selectTotal();
    System.out.println(total);
}
```

细节：接口返回类型建议用 Long 而不是 long，更防御；InnoDB 的 `count(*)` 没有存计数，是真实扫描（优化器选最小二级索引），几亿行的大表要评估耗时——这也是 PageHelper 某些场景建议手写 count 的原因。

# 8. 多表关联：join 一条 SQL vs 分步两条 SQL

两种基本形态：**join 出一条宽行 + resultMap 嵌套映射（association）**，与**两条简单 SQL + select/column 分步查询**（完整语法见 13 篇）。

```xml
<!-- 分步形态：主SQL只查主表，association用select+column触发第二条SQL -->
<resultMap id="stuByStep" type="Student">
    <id property="sid" column="sid"/>
    <result property="sname" column="sname"/>
    <association property="clazz"
                 select="com.ziv.mybatis.mapper.ClazzMapper.selectByIdStepTwo"
                 column="cid" fetchType="lazy"/>
</resultMap>
<select id="selectByIdStep" resultMap="stuByStep">
    select sid, sname, cid from t_stu where sid = #{sid}
</select>
```

| 维度 | join 嵌套映射 | 分步 select |
|------|---------------|-------------|
| SQL 条数 | 1 条（宽行） | 1 + N 条（N=主结果行数） |
| N+1 问题 | 无 | 不配懒加载时**有** |
| 一对多膨胀 | 行重复，主对象列冗余传输 | 无膨胀 |
| 懒加载 | 不支持 | 支持（fetchType="lazy"） |
| 跨表过滤/排序/分页 | join 后直接 order by + limit | 只能对主表分页 |

**判断口径**：一对一/多对一且需要按关联表字段过滤、排序、分页时用 join；一对多基数大、关联对象可能用不到（懒加载收益）、子查询复用度高时用分步。分步在"列表 20 行 × 每行一次子查询"下就是 21 条 SQL——**它把 join 的压力转移到了网络往返上**。

> **设计争议**：要不要在应用层做 join（两条 SQL 在 Java 里 Map 拼接）？支持方：分库分表后跨片 join 不可行、缓存友好；反对方：多一次数据搬运、手工拼 key 易错。分库分表场景下应用层组装往往才是唯一选项。

# 9. 分页：手写 limit 与 PageHelper

标准公式 `startIndex = (pageNum - 1) * pageSize`，手写版要把参数传进 SQL（见 16 篇）。PageHelper 把它收敛成一行：

```java
PageHelper.startPage(2, 2);            // 分页参数写入 ThreadLocal
List<Car> cars = mapper.selectAll();   // 被拦截器改写：先count再拼LIMIT
PageInfo<Car> pageInfo = new PageInfo<>(cars, 5);
```

**原理**：PageInterceptor 拦截 Executor.query()，从 ThreadLocal 取参数，先执行 `select count(0) from (原SQL) tmp_count` 查总数，再拼 LIMIT 并反射改 BoundSql 的 sql 字段执行（源码分析见 16 篇）。

高频坑位：

1. **startPage 与目标查询之间不能插入其他查询**：ThreadLocal 参数被"下一条查询"消费，中间多执行一条无关 SQL 就分页错对象；它也只对紧随的第一条 select 生效。
2. **count 优化是双刃剑**：遇到 `count(distinct ...)` 等复杂 SQL 优化失败退化为子查询包裹，大表极慢；可在 XML 里手写同 id + `_Count` 后缀的 count 语句覆盖。
3. **深分页依然深**：`limit 1000000, 10` 要先扫过一百万行，插件救不了，只能延迟关联（子查询先按覆盖索引翻页取 id，再 join 回表）。
4. **线程池/异步环境下 ThreadLocal 会串**，异步分支里 startPage 后记得 finally clearPage()。
# 10. 批量 insert 的三种写法

| 写法 | 形态 | 性能要点 |
|------|------|----------|
| foreach 多值拼接 | 一条 SQL N 组 values | 最快，分片防 max_allowed_packet |
| ExecutorType.BATCH | 逐行 addBatch，统一 executeBatch | MySQL 需 `rewriteBatchedStatements=true` |
| 循环单条 insert | N 次网络往返 | 慢，仅小数据量可接受 |

foreach 多值拼接即 12 篇的 insertBatch 写法，一次网络往返 N 行；BATCH 执行器写法：

```java
// ExecutorType.BATCH：攒够一批再发
try (SqlSession session = factory.openSession(ExecutorType.BATCH, false)) {
    CarMapper mapper = session.getMapper(CarMapper.class);
    int i = 0;
    for (Car car : cars) {
        mapper.insert(car);                            // 只 addBatch
        if (++i % 500 == 0) session.flushStatements(); // 分片提交
    }
    session.flushStatements();
    session.commit();
}
```

> **注意**：BATCH 模式返回的受影响行数不可信（MySQL 返回 -2），不要用返回值判断每行成败；foreach 分片细节见 9 篇第 8 节。

# 11. 游标流式查询：fetchSize

全量导出百万行时，默认驱动会把整个结果集拉进内存。**流式的本质：服务端按 fetchSize 分批吐行，客户端逐行消费**。

```xml
<!-- MySQL 真流式：fetchSize 必须是 Integer.MIN_VALUE，结果集前缀只读 -->
<select id="streamAll" resultType="car"
        fetchSize="-2147483648" resultSetType="FORWARD_ONLY">
    select * from t_car
</select>
```

```java
// ResultHandler 逐行回调，边读边写，不落全量List
mapper.streamAll(ctx -> excelWriter.write((Car) ctx.getResultObject()));
```

要点与限制：**MySQL Connector/J 只有 fetchSize=Integer.MIN_VALUE（-2147483648）才启用逐行流式**，设成 1000、5000 无效，依旧全量缓冲；Oracle/PostgreSQL 按正常数值生效（PG 还需关闭 autocommit）。**同一连接上流未读完不能发起第二条查询**，边消费边写入得另开会话。游标长时间占用连接，**失败只能从头再来**，大导出优先"按 id 区间切片并行分页"；且与 PageHelper 互斥。

# 12. 复杂动态条件拼装

多条件搜索（品牌、价格区间、id 集合、排序任意组合）用 where + if + choose 拼装（标签机制见 12 篇）：

```xml
<select id="selectByCondition" resultType="car">
    select <include refid="carColumnName"/>
    from t_car
    <where>
        deleted = 0
        <if test="brand != null and brand != ''">
            and brand like concat('%', #{brand}, '%')
        </if>
        <if test="minPrice != null">and guide_price &gt;= #{minPrice}</if>
        <if test="maxPrice != null">and guide_price &lt;= #{maxPrice}</if>
        <if test="ids != null and ids.size() > 0">
            and id in
            <foreach collection="ids" item="id" open="(" separator="," close=")">#{id}</foreach>
        </if>
        <choose>
            <when test="orderBy == 'price'">order by guide_price desc</when>
            <otherwise>order by id desc</otherwise>
        </choose>
    </where>
</select>
```

拼装四条纪律：**恒真条件打底（where 1=1）已过时**，`<where>` 自动吃掉首个 and/or；**排序白名单放 choose 而不是 ${}**——`order by #{orderBy}` 按常量排序无效，`order by ${orderBy}` 可注入，choose 枚举合法分支两全；**in 列表必须判空**，空集合会生成 `id in ()` 语法报错；**LIKE 用 concat**，`${}` 模糊拼接有注入（见 9 篇第 1 节）。

> **行业实践**：条件继续膨胀（运营后台几十个字段的组合搜索）时 XML 维护成本急剧上升，常见替代是 MyBatis Dynamic SQL（Java 构建条件、编译期检查）或 MyBatis-Plus 的 Wrapper（见 25 篇）。

# 13. 慢 SQL 定位

查询写得对不代表跑得快。**定位链路：开关采集 → 看执行计划 → 回到映射层改 SQL**。

```bash
# my.cnf 打开慢查询日志：slow_query_log=1，long_query_time=0.5（超过500ms记录）
# 分析：mysqldumpslow -s t -t 10 mysql-slow.log
# 计划：explain select ... ;  重点看 type（ALL<range<ref）、key、rows、Extra里的filesort/temporary
```

MyBatis 侧配套手段：

- **看最终执行 SQL**：对 `org.apache.ibatis.logging` 开 DEBUG，日志里的 Preparing 就是改写后的语句；用 p6spy 拿到参数已回填的完整 SQL，排查隐式类型转换导致的索引失效。
- **分页慢先怀疑 count**：手写 `_Count` 覆盖（见第 9 节）；**别 select \***，join 宽行在大结果集下重复列的传输与合并开销都会放大。

> **行业实践**：慢 SQL 治理通常前移到发布阶段：测试环境压低 long_query_time 收集超标语句纳入 code review；框架层能做的是让每条 SQL 可追踪——traceId 与 SQL 日志同行输出，定位到具体 Mapper 方法。

# 14. 小结

- **接收类型跟着结果形状走**：单条实体、多条 List（空结果是空集合不是 null）、无实体用 Map，行转索引字典用 @MapKey；Map 系是报表兜底，主链路要沉淀 DTO。
- **列名对不上三种解法**：as（局部）、resultMap（嵌套映射唯一载体，优先级高于 resultType）、全局驼峰（规范项目默认开）。
- **多表映射二选一**：小基数、需跨表过滤排序分页选 join；大一对多、可懒加载、子查询要复用选分步，警惕分步 N+1。
- **分页**：PageHelper 吃 ThreadLocal，startPage 后必须紧跟目标查询；深分页靠延迟关联。**批量**按量级选 foreach 分片/BATCH/流式；**动态拼装**守 where 标签、choose 白名单、in 判空、concat 模糊四条纪律；**慢 SQL**走慢日志 + explain + p6spy。
