# 1. 数据库测试策略

## 1.1 选型维度

| 方案 | 优点 | 缺点 |
|------|------|------|
| 内嵌数据库 | 快、轻量 | 与生产差异大 |
| 专用测试库 | 与生产接近 | 维护成本高 |
| 容器化数据库 | 环境一致 | 启动慢（可选） |

在不使用外部容器的前提下，推荐：

- 单元测试不依赖数据库
- Repository 测试使用内嵌数据库
- 集成测试使用专用测试库

---

# 2. `@DataJpaTest`

## 2.1 特性

- 自动配置 JPA 与数据库相关 Bean
- 默认回滚事务
- 提供 `TestEntityManager`

```java
@DataJpaTest
class UserRepositoryTest {
    @Autowired
    private TestEntityManager em;
}
```

## 2.2 实体扫描

若实体不在默认扫描路径，可使用：

```java
@DataJpaTest
@EntityScan("com.example.domain")
class EntityScanTest {}
```

---

# 3. `@AutoConfigureTestDatabase`

默认会替换真实数据库为内嵌数据库：

```java
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
```

适用于需要使用真实测试库的场景。

---

# 4. 数据库迁移与初始化

## 4.1 Flyway / Liquibase

- 可在测试中自动执行迁移
- 保持与生产一致的 Schema 版本

## 4.2 SQL 初始化

```java
@Sql({"classpath:schema.sql", "classpath:data.sql"})
class SqlInitTest {}
```

---

# 5. Repository 测试实践

## 5.1 Flush 与断言

```java
User saved = repo.save(user);
em.flush();
em.clear();
User loaded = repo.findById(saved.getId()).orElseThrow();
```

通过 `flush/clear` 避免一阶缓存影响断言。

## 5.2 分页与排序

重点验证分页条件、排序字段是否与业务一致。

---

# 6. 性能与稳定性

- 避免在测试中依赖真实网络数据库
- 保持数据清理策略一致（回滚或清表）
- 测试间不要共享可变数据状态
