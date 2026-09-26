# 1. 测试切片概览

## 1.1 为什么需要切片

完整启动 `@SpringBootTest` 成本高，切片测试通过**只加载某一层相关 Bean**实现：

- 启动更快
- 边界更清晰
- 问题定位更容易

## 1.2 切片的本质

切片是对自动配置的一种裁剪：

- 默认只加载特定自动配置
- 通过过滤器限制组件扫描
- 不会加载与该层无关的 Bean

---

# 2. `@WebMvcTest`

## 2.1 默认加载内容

- `@Controller`、`@ControllerAdvice`
- `@JsonComponent`
- `@WebMvcConfigurer`
- `MockMvc` 与相关基础设施

**不会加载**：`@Service`、`@Repository`、数据库配置

## 2.2 典型用法

```java
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UserService userService;
}
```

## 2.3 扩展方式

- `@Import` 引入自定义配置
- `@MockBean` 替换 Service 依赖

---

# 3. `@WebFluxTest`

用于响应式 WebFlux 场景，只加载 `@Controller` 与 WebFlux 相关配置。

```java
@WebFluxTest
class ReactiveControllerTest {}
```

---

# 4. `@DataJpaTest`

专注 JPA Repository 测试：

- 自动配置 JPA、DataSource、EntityManager
- 默认回滚事务
- 提供 `TestEntityManager`

```java
@DataJpaTest
class UserRepositoryTest {}
```

---

# 5. `@JdbcTest`

用于纯 JDBC 场景：

- 自动配置 `JdbcTemplate`
- 不加载 JPA 相关 Bean

---

# 6. `@JsonTest`

用于 JSON 序列化与反序列化测试：

- 提供 `JacksonTester`
- 只加载 JSON 相关 Bean

```java
@JsonTest
class JsonSerializationTest {}
```

---

# 7. `@RestClientTest`

测试 `RestTemplate` 或 `WebClient` 的客户端组件：

- 提供 `MockRestServiceServer`
- 自动配置 JSON 映射

---

# 8. 其他常见切片

| 注解 | 说明 |
|------|------|
| `@DataMongoTest` | MongoDB Repository 测试 |
| `@DataRedisTest` | Redis 测试 |
| `@GraphQlTest` | GraphQL 控制器测试 |
| `@DataJdbcTest` | Spring Data JDBC 测试 |

---

# 9. 切片还是 `@SpringBootTest`

## 9.1 选择原则

- 只测某一层 → 切片
- 多层协作/配置验证 → `@SpringBootTest`

## 9.2 常见误区

- 切片中引入过多 `@Import` 反而让测试变慢
- 切片失败时，优先检查“缺失依赖”而不是业务逻辑
