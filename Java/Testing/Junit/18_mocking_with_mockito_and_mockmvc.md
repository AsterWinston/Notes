# 1. Mockito 基础

Mockito 常用于隔离外部依赖，关注被测对象的行为。

## 1.1 依赖与扩展

JUnit 5 推荐使用 `MockitoExtension`：

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    @Mock
    private UserRepository repo;

    @InjectMocks
    private UserService service;
}
```

## 1.2 常用 API

- `when(...).thenReturn(...)`
- `doThrow(...).when(...)`
- `verify(...)`

```java
when(repo.findById(1L)).thenReturn(Optional.of(user));
verify(repo).findById(1L);
```

## 1.3 BDD 风格

```java
given(repo.findById(1L)).willReturn(Optional.of(user));
then(repo).should().findById(1L);
```

# 2. MockMvc 与 Web 层测试

`MockMvc` 适合测试 Controller 与过滤器链。

## 2.1 典型用法

```java
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void get_user() throws Exception {
        mockMvc.perform(get("/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").value(1));
    }
}
```

## 2.2 常见注意点

- `@WebMvcTest` 只加载 MVC 相关 Bean
- 对业务层使用 `@MockBean` 注入

# 3. Mockito 使用误区

- 过度 Mock 导致测试验证实现细节
- 验证次数过多，降低可维护性
- 忽略行为驱动的断言，只验证调用次数
