# 1. 分页插件

Mybatis-Plus 3.4.0+ 使用 `MybatisPlusInterceptor` 插件体系，分页由 `PaginationInnerInterceptor` 实现。

## 1.1 配置分页插件

```java
@Configuration
public class MybatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();

        // 1. 分页插件
        PaginationInnerInterceptor paginationInterceptor = new PaginationInnerInterceptor(DbType.MYSQL);
        // 设置最大单页限制（防止恶意查询全量数据）
        paginationInterceptor.setMaxLimit(500L);
        // 溢出总页数后是否回到第一页（默认 false）
        paginationInterceptor.setOverflow(false);
        // 是否优化 count 查询（默认 true，左连接时会优化）
        paginationInterceptor.setOptimizeJoin(true);

        interceptor.addInnerInterceptor(paginationInterceptor);

        // 2. 乐观锁插件（见第3节）
        // interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());

        // 3. 多租户插件（见第5节）
        // interceptor.addInnerInterceptor(new TenantLineInnerInterceptor(...));

        // 4. 防全表更新与删除插件
        interceptor.addInnerInterceptor(new BlockAttackInnerInterceptor());

        return interceptor;
    }
}
```

> **注意：** 旧版的 `PaginationInterceptor` 已在 3.4.0 废弃，必须使用 `MybatisPlusInterceptor` + `PaginationInnerInterceptor`。

## 1.2 使用分页

```java
@SpringBootTest
class PageTest {
    @Autowired
    private UserMapper userMapper;

    // ===== Mapper 层分页 =====
    @Test
    void testPage() {
        // 创建分页对象：第1页，每页10条
        Page<User> page = new Page<>(1, 10);

        // 无条件分页查询
        Page<User> result = userMapper.selectPage(page, null);

        System.out.println("总记录数：" + result.getTotal());
        System.out.println("总页数：" + result.getPages());
        System.out.println("当前页：" + result.getCurrent());
        System.out.println("每页大小：" + result.getSize());
        System.out.println("数据列表：" + result.getRecords());
    }

    @Test
    void testPageWithCondition() {
        // 条件分页
        LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<>();
        wrapper.gt(User::getAge, 18);

        Page<User> page = new Page<>(1, 10);
        Page<User> result = userMapper.selectPage(page, wrapper);
    }

    @Test
    void testPageWithOrder() {
        // 分页 + 排序
        Page<User> page = new Page<>(1, 10);
        LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<>();
        wrapper.orderByDesc(User::getCreateTime);

        Page<User> result = userMapper.selectPage(page, wrapper);
    }

    // ===== Service 层分页 =====
    @Autowired
    private UserService userService;

    @Test
    void testServicePage() {
        Page<User> page = new Page<>(1, 10);
        LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<>();
        wrapper.eq(User::getStatus, 1);

        // IService 提供的方法
        IPage<User> result = userService.page(page, wrapper);
    }
}
```

## 1.3 自定义分页 SQL

当需要多表关联分页时，MP 的分页插件会自动处理分页，只需在 Mapper 中编写关联 SQL 即可：

```java
// Mapper 接口
@Mapper
public interface UserMapper extends BaseMapper<User> {
    // Page 必须是第一个参数，wrapper 通过 @Param 传入
    IPage<UserVO> selectUserWithOrders(Page<UserVO> page, @Param("ew") Wrapper<User> wrapper);
}
```

```xml
<!-- UserMapper.xml -->
<select id="selectUserWithOrders" resultType="com.example.vo.UserVO">
    SELECT u.*, COUNT(o.id) as order_count, SUM(o.amount) as total_amount
    FROM t_user u
    LEFT JOIN t_order o ON u.id = o.user_id
    ${ew.customSqlSegment}
    GROUP BY u.id
</select>
```

```java
// 使用
Page<UserVO> page = new Page<>(1, 10);
LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<>();
wrapper.eq(User::getStatus, 1);
IPage<UserVO> result = userMapper.selectUserWithOrders(page, wrapper);
```

## 1.4 分页返回 Map

```java
// 查询结果不映射实体，直接返回 List<Map>
Page<Map<String, Object>> page = new Page<>(1, 10);
LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<>();
wrapper.select(User::getId, User::getUsername, User::getAge);
IPage<Map<String, Object>> result = userMapper.selectMapsPage(page, wrapper);
```

---

# 2. 通用分页实体

## 2.1 Page 对象详解

```java
// Page<T> 核心属性
Page<User> page = new Page<>(1, 10);   // current=1, size=10

// 链式设置
Page<User> page = new Page<User>()
        .setCurrent(1)                  // 当前页（默认 1）
        .setSize(10)                    // 每页大小（默认 10）
        .addOrder(OrderItem.desc("create_time"))  // 排序
        .addOrder(OrderItem.asc("id"));

// 查询结果
IPage<User> result = userMapper.selectPage(page, null);
result.getCurrent();    // 当前页码
result.getSize();       // 每页大小
result.getTotal();      // 总记录数
result.getPages();      // 总页数
result.getRecords();    // 当前页数据列表
result.hasNext();       // 是否有下一页
result.hasPrevious();   // 是否有上一页
```

## 2.2 分页结果 DTO 封装

```java
// 统一分页响应对象
@Data
public class PageResult<T> {
    private Long total;          // 总记录数
    private Long pages;          // 总页数
    private Long current;        // 当前页
    private Long size;           // 每页大小
    private List<T> records;     // 数据列表

    public static <T> PageResult<T> of(IPage<T> page) {
        PageResult<T> result = new PageResult<>();
        result.setTotal(page.getTotal());
        result.setPages(page.getPages());
        result.setCurrent(page.getCurrent());
        result.setSize(page.getSize());
        result.setRecords(page.getRecords());
        return result;
    }
}

// Controller 中使用
@RestController
@RequestMapping("/users")
public class UserController {
    @Autowired
    private UserService userService;

    @GetMapping
    public PageResult<User> list(@RequestParam(defaultValue = "1") Integer page,
                                  @RequestParam(defaultValue = "10") Integer size) {
        Page<User> p = new Page<>(page, size);
        IPage<User> result = userService.page(p);
        return PageResult.of(result);
    }
}
```

## 2.3 不进行 count 查询的分页

某些场景下不需要知道总记录数，可以跳过 count 查询提升性能：

```java
// 只需要当前页数据，不需要总记录数
Page<User> page = new Page<>(1, 10, false);  // 第三个参数为 searchCount
IPage<User> result = userMapper.selectPage(page, null);

// 或者
Page<User> page = new Page<>(1, 10);
page.setSearchCount(false);
IPage<User> result = userMapper.selectPage(page, null);
```

---

# 3. 乐观锁插件

乐观锁（Optimistic Lock）用于解决并发更新时的数据冲突问题。Mybatis-Plus 通过版本号 `version` 字段实现。

## 3.1 配置乐观锁插件

```java
@Configuration
public class MybatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();

        // 分页插件
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));

        // 乐观锁插件
        interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());

        return interceptor;
    }
}
```

## 3.2 实体类使用 @Version

```java
@Data
@TableName("t_product")
public class Product {
    private Long id;
    private String name;
    private Integer stock;      // 库存

    @Version                    // 标记为乐观锁版本号字段
    private Integer version;    // 版本号
}
```

> **数据库对应字段：**
> ```sql
> CREATE TABLE t_product (
>     id BIGINT PRIMARY KEY,
>     name VARCHAR(100),
>     stock INT,
>     version INT DEFAULT 1   -- 默认版本号为 1
> );
> ```

## 3.3 使用效果

```java
// 场景：并发扣库存
// 线程 A 和 线程 B 同时查询同一个商品
Product productA = productMapper.selectById(1L);   // version = 1, stock = 100
Product productB = productMapper.selectById(1L);   // version = 1, stock = 100

// 线程 A 先扣库存
productA.setStock(productA.getStock() - 10);
productMapper.updateById(productA);
// 实际执行：UPDATE t_product SET stock=90, version=2 WHERE id=1 AND version=1
// 更新成功，version 变为 2

// 线程 B 后扣库存
productB.setStock(productB.getStock() - 10);
productMapper.updateById(productB);
// 实际执行：UPDATE t_product SET stock=90, version=2 WHERE id=1 AND version=1
// 此时 version 已经变成 2，WHERE version=1 不匹配，更新 0 行，更新失败！
```

**更新失败处理：**

```java
@Service
public class ProductServiceImpl extends ServiceImpl<ProductMapper, Product> {

    @Transactional
    public boolean deductStock(Long productId, int quantity) {
        Product product = getById(productId);
        if (product.getStock() < quantity) {
            throw new RuntimeException("库存不足");
        }

        product.setStock(product.getStock() - quantity);
        boolean updated = updateById(product);

        if (!updated) {
            // 乐观锁更新失败（被其他线程抢先修改），重试或抛异常
            throw new RuntimeException("更新失败，请重试");
        }

        return true;
    }
}
```

> **注意：**
> - `@Version` 仅支持 `updateById(entity)` 和 `update(entity, wrapper)`，不支持 `update(wrapper)`（不带实体）
> - 整数类型的 version 字段，每次更新自动 +1
> - 日期类型的 version 字段，每次更新自动设置为当前时间

---

# 4. 自动填充

Mybatis-Plus 的 `MetaObjectHandler` 可以自动填充 `createTime`、`updateTime` 等字段，避免每次手动设置。

## 4.1 实体类配置

```java
@Data
@TableName("t_user")
public class User {
    private Long id;
    private String username;

    @TableField(fill = FieldFill.INSERT)          // 插入时自动填充
    private LocalDateTime createTime;

    @TableField(fill = FieldFill.INSERT_UPDATE)   // 插入和更新时自动填充
    private LocalDateTime updateTime;

    @TableField(fill = FieldFill.INSERT)          // 插入时自动填充
    private Long createUser;

    @TableField(fill = FieldFill.INSERT_UPDATE)   // 插入和更新时自动填充
    private Long updateUser;
}
```

| fill 枚举值 | 触发时机 |
|------------|---------|
| `DEFAULT` | 不自动填充 |
| `INSERT` | 插入时自动填充 |
| `UPDATE` | 更新时自动填充 |
| `INSERT_UPDATE` | 插入和更新时自动填充 |

## 4.2 实现 MetaObjectHandler

```java
@Component
public class MyMetaObjectHandler implements MetaObjectHandler {

    @Override
    public void insertFill(MetaObject metaObject) {
        // 插入时自动填充
        this.strictInsertFill(metaObject, "createTime", LocalDateTime.class, LocalDateTime.now());
        this.strictInsertFill(metaObject, "updateTime", LocalDateTime.class, LocalDateTime.now());
        this.strictInsertFill(metaObject, "createUser", Long.class, getCurrentUserId());
        this.strictInsertFill(metaObject, "updateUser", Long.class, getCurrentUserId());

        // 如果实体类中已有值（手动设置过），是否覆盖？
        // 使用 strictInsertFill：字段已有值时不再填充
        // 使用 fillStrategy：不判断已有值，强制填充
    }

    @Override
    public void updateFill(MetaObject metaObject) {
        // 更新时自动填充
        this.strictUpdateFill(metaObject, "updateTime", LocalDateTime.class, LocalDateTime.now());
        this.strictUpdateFill(metaObject, "updateUser", Long.class, getCurrentUserId());
    }

    private Long getCurrentUserId() {
        // 从 SecurityContext、ThreadLocal 或 Request 中获取当前登录用户 ID
        // return SecurityUtils.getCurrentUserId();
        return 1L;  // 示例
    }
}
```

## 4.3 覆盖策略

```java
// 方式一：使用 fillStrategy 无论字段是否有值都强制填充
@Override
public void insertFill(MetaObject metaObject) {
    this.fillStrategy(metaObject, "createTime", LocalDateTime.now());
}

// 方式二：使用 strictInsertFill，仅当字段无值时才填充
@Override
public void insertFill(MetaObject metaObject) {
    this.strictInsertFill(metaObject, "createTime", LocalDateTime.class, LocalDateTime.now());
}
```

---

# 5. 多租户插件

多租户（Multi-Tenant）是指在一个应用实例中隔离不同租户（如不同企业、不同组织）的数据。Mybatis-Plus 通过 `TenantLineInnerInterceptor` 在 SQL 执行前自动拼接租户条件。

## 5.1 定义租户处理器

```java
@Component
public class MyTenantLineHandler implements TenantLineHandler {

    /**
     * 获取当前租户 ID
     */
    @Override
    public Expression getTenantId() {
        // 从当前请求上下文获取租户 ID
        // 通常从 JWT Token、请求头、ThreadLocal 中获取
        Long tenantId = TenantContextHolder.getCurrentTenantId();
        if (tenantId == null) {
            throw new RuntimeException("无法获取租户信息");
        }
        return new LongValue(tenantId);
    }

    /**
     * 获取租户 ID 对应的数据库字段名
     */
    @Override
    public String getTenantIdColumn() {
        return "tenant_id";
    }

    /**
     * 忽略多租户的表（不需要过滤租户的表）
     */
    @Override
    public boolean ignoreTable(String tableName) {
        // 系统配置表、字典表不区分租户
        List<String> ignoreTables = Arrays.asList("t_sys_config", "t_dict");
        return ignoreTables.contains(tableName);
    }
}
```

## 5.2 租户上下文工具（ThreadLocal）

```java
public class TenantContextHolder {
    private static final ThreadLocal<Long> CONTEXT = new ThreadLocal<>();

    public static void setCurrentTenantId(Long tenantId) {
        CONTEXT.set(tenantId);
    }

    public static Long getCurrentTenantId() {
        return CONTEXT.get();
    }

    public static void clear() {
        CONTEXT.remove();   // 必须清理，防止内存泄露
    }
}
```

## 5.3 配置多租户插件

```java
@Configuration
public class MybatisPlusConfig {

    @Autowired
    private MyTenantLineHandler tenantLineHandler;

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();

        // 分页插件
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));

        // 多租户插件
        interceptor.addInnerInterceptor(new TenantLineInnerInterceptor(tenantLineHandler));

        return interceptor;
    }
}
```

## 5.4 过滤器中设置租户 ID

```java
@Component
public class TenantFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;

        // 从请求头获取租户 ID
        String tenantIdStr = httpRequest.getHeader("X-Tenant-Id");
        try {
            if (StringUtils.hasText(tenantIdStr)) {
                TenantContextHolder.setCurrentTenantId(Long.valueOf(tenantIdStr));
            }
            chain.doFilter(request, response);
        } finally {
            // 请求结束后清理，防止内存泄露
            TenantContextHolder.clear();
        }
    }
}
```

## 5.5 多租户 SQL 效果

```java
// 查询所有用户
userMapper.selectList(null);

// 自动拼接租户条件后的 SQL：
// SELECT id, username FROM t_user WHERE tenant_id = 1

// 按 ID 查询
userMapper.selectById(1L);
// SQL: SELECT id, username FROM t_user WHERE id = 1 AND tenant_id = 1

// 分页查询
userMapper.selectPage(page, wrapper);
// SQL: SELECT id, username FROM t_user WHERE status = 1 AND tenant_id = 1 LIMIT ?
```

---

# 6. 防全表更新与删除插件（BlockAttackInnerInterceptor）

防止开发时误操作导致全表更新或删除，仅拦截 Mapper 层。

```java
@Configuration
public class MybatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();

        // 分页插件
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));

        // 防全表更新与删除插件（必须放在最后）
        interceptor.addInnerInterceptor(new BlockAttackInnerInterceptor());

        return interceptor;
    }
}
```

**效果：**
```java
// 不带条件的更新
userMapper.update(new User(), null);
// 抛出异常：Prohibition of full table update

// 不带条件的删除
userMapper.delete(null);
// 抛出异常：Prohibition of full table deletion

// 带条件的正常操作不受影响
userMapper.delete(new LambdaQueryWrapper<User>().eq(User::getId, 1L));
// 正常执行
```

---

# 7. 插件配置完整示例

```java
@Configuration
@MapperScan("com.example.mapper")     // 扫描 Mapper 接口
public class MybatisPlusConfig {

    @Autowired
    private MyTenantLineHandler tenantLineHandler;

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();

        // 1. 多租户插件（最先执行）
        interceptor.addInnerInterceptor(new TenantLineInnerInterceptor(tenantLineHandler));

        // 2. 分页插件
        PaginationInnerInterceptor pagination = new PaginationInnerInterceptor(DbType.MYSQL);
        pagination.setMaxLimit(500L);
        interceptor.addInnerInterceptor(pagination);

        // 3. 乐观锁插件
        interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());

        // 4. 防全表更新与删除插件（最后执行）
        interceptor.addInnerInterceptor(new BlockAttackInnerInterceptor());

        return interceptor;
    }
}
```

> **执行顺序：** 多个 InnerInterceptor 按添加顺序依次执行，before 阶段按添加顺序、after 阶段按逆序执行。多租户建议最先添加，BlockAttack 建议最后添加。
