# 1. 测试替身（Test Double）分类

## 1.1 常见类型

| 类型 | 目的 | 特点 |
|------|------|------|
| Dummy | 占位 | 不参与断言 |
| Stub | 固定返回 | 关注结果 |
| Mock | 关注交互 | 验证调用 |
| Spy | 代理真实对象 | 局部替换 |
| Fake | 简化实现 | 真实可运行 |

**核心原则**：替身越多，测试越快，但可能偏离真实行为。

---

# 2. Mockito 核心用法

## 2.1 Stubbing（打桩）

```java
when(userRepo.findById(1L)).thenReturn(Optional.of(user));
```

对于 void 方法：

```java
doThrow(new IllegalStateException()).when(service).close();
```

## 2.2 验证交互

```java
verify(userRepo, times(1)).save(any(User.class));
```

## 2.3 参数捕获

```java
ArgumentCaptor<User> captor = ArgumentCaptor.forClass(User.class);
verify(userRepo).save(captor.capture());
assertThat(captor.getValue().getName()).isEqualTo("tom");
```

## 2.4 严格度与宽松模式

JUnit 5 可使用 MockitoExtension 控制严格度：

```java
@ExtendWith(MockitoExtension.class)
class PureUnitTest {}
```

严格模式可帮助发现“多余 stubbing”。

---

# 3. Spring Boot 集成

## 3.1 `@MockBean`

`@MockBean` 会把 mock 注入到 Spring 容器，并替换同类型的真实 Bean。

```java
@SpringBootTest
class OrderServiceTest {

    @MockBean
    private PaymentGateway paymentGateway;
}
```

## 3.2 `@SpyBean`

用于包装真实 Bean，仅覆盖部分方法。

```java
@SpyBean
private PriceService priceService;
```

## 3.3 `@Mock` 与 `@MockBean` 区别

| 注解 | 作用范围 | 是否进入容器 |
|------|----------|--------------|
| `@Mock` | 纯 Mockito | 否 |
| `@MockBean` | Spring Test | 是 |

---

# 4. 常见问题与陷阱

## 4.1 过度 Mock

过度使用 `@MockBean` 会导致：

- 真实配置无法验证
- 集成问题被隐藏

## 4.2 Final 类与静态方法

- Mockito 默认不支持静态方法 mock
- 如需 mock 静态方法需额外配置（谨慎使用）

## 4.3 深度 Stubbing

```java
when(order.getCustomer().getAddress().getCity()).thenReturn("SZ");
```

过深链路通常是设计问题，应考虑重构或使用 Fake。

---

# 5. 实践建议

- 单元测试优先使用 `@Mock`
- 切片测试使用 `@MockBean` 隔离外部依赖
- 集成测试避免过多替身，优先验证真实配置
