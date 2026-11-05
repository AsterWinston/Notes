# 1. 反模式与最佳实践

学完 API 之后，真正决定测试质量的是**使用方式**。同一个 `verify`，可以验证关键副作用，也可以把实现细节焊死；同一个 `@Mock`，可以隔离外部系统，也可以让测试变成"验证 Mock 的 Mock"。本篇收束整个模块，先拆解常见反模式，再给出可执行的决策清单。

## 1.1 过度 Mock

过度 Mock 的典型症状是：测试里几乎每个依赖都是 Mock，连值对象、集合、`Optional` 都要造一个替身。测试看起来覆盖了很多分支，实际上只验证了"代码按我写的顺序调用了这些 Mock"。

```java
// 反模式：把值对象也 Mock 掉，测试失去真实语义
@Mock
private User user;
@Mock
private Address address;

@Test
void should_build_display_name() {
    when(user.getName()).thenReturn("Ada");
    when(user.getAddress()).thenReturn(address);
    when(address.getCity()).thenReturn("London");

    assertEquals("Ada (London)", formatter.format(user));
}
```

这段测试有两个问题。第一，`User` 和 `Address` 是纯数据，直接 `new` 出来更真实、更简单。第二，所有字段都被写死，测试无法发现"用户没有地址"或"城市为空"的真实分支。

| 过度 Mock 的信号 | 后果 | 修正 |
|------------------|------|------|
| Mock 值对象/DTO | 测试与对象结构耦合 | 直接构造真实对象 |
| Mock 返回 Mock | 测试难以阅读 | 构造完整对象图，或使用 Fake |
| 只 Mock 不验证行为 | 测试变成"桩配置表演" | 断言被测方法的结果 |
| 每个方法都 `verify` | 任何重构都失败 | 只验证跨边界协作 |
| 为了覆盖率 Mock 私有逻辑 | 测试脆弱且无意义 | 通过公共行为覆盖 |
| 用 `RETURNS_DEEP_STUBS` 串联 | 隐藏对象图耦合 | 显式构造需要的依赖 |

**设计考量**：Mock 应该放在**系统边界**上：数据库、HTTP、消息队列、文件系统、时钟、第三方 SDK。领域对象、值对象、集合、`Optional` 这些属于系统内部，应该使用真实实例。边界越清晰，Mock 越少，测试越稳。

## 1.2 验证实现细节

验证实现细节是 Mockito 最容易产生的反模式。它通常长这样：

```java
// 反模式：把内部步骤当成业务契约
@Test
void should_create_order() {
    service.createOrder(command);

    verify(cache).get("product:1");
    verify(repository).findById(1L);
    verify(discountCalculator).calculate(command);
    verify(orderFactory).create(command, product);
    verify(repository).save(any(Order.class));
    verify(eventPublisher).publish(any(OrderCreated.class));
    verifyNoMoreInteractions(repository);
}
```

这段测试断言了六个内部步骤。只要把缓存从 `cache.get` 换成 `cache.getOptional`，或者把工厂改成构造器，测试就会失败，但订单创建行为可能完全没变。测试从此变成"实现快照"，而不是"行为规格"。

更合理的写法是只验证**外部可观察结果**：

```java
@Test
void should_create_order_with_expected_total() {
    when(repository.findById(1L)).thenReturn(Optional.of(product));
    when(repository.save(any(Order.class)))
            .thenAnswer(invocation -> invocation.getArgument(0));

    Order result = service.createOrder(command);

    assertEquals(new BigDecimal("198.00"), result.total());
    assertEquals(OrderStatus.CREATED, result.status());
    // 只有"发布事件"是跨边界副作用，值得单独验证
    verify(eventPublisher).publish(any(OrderCreated.class));
}
```

**判断标准**：如果测试失败，业务用户会不会感知到？会，说明测试在保护行为；不会，说明测试在保护实现。后者的价值要低得多。

| 实现细节 | 更适合的断言 |
|----------|--------------|
| 先查缓存还是先查库 | 不验证 |
| 用了哪个 Mapper | 断言最终 DTO |
| 内部方法调用顺序 | 只在业务有强顺序时用 `InOrder` |
| 日志内容 | 单独测试日志配置，或不断言 |
| 指标名称 | 用监控系统测试，不用单元测试 |
| 私有方法调用 | 通过公共方法间接覆盖 |

## 1.3 Mock 第三方库

直接 Mock 第三方库的类，是另一个高风险习惯。第三方类型的方法签名、final 修饰、内部状态都可能在升级时变化；Mock 它们会让测试与库的实现强绑定。

```java
// 反模式：直接 Mock 第三方 SDK
@Mock
private S3Client s3Client;

@Test
void should_upload() {
    when(s3Client.putObject(any(PutObjectRequest.class), any(RequestBody.class)))
            .thenReturn(PutObjectResponse.builder().build());

    storageService.upload(file);
}
```

这段代码的问题不是"不能工作"，而是：

- SDK 升级后构造器、Builder、返回类型变化，测试全部要改。
- 你验证的是 SDK 调用细节，不是自己的存储契约。
- 真实网络行为（超时、重试、签名）完全没有覆盖。

更好的做法是**包一层自己的端口接口**：

```java
public interface FileStorage {
    String upload(String key, byte[] content);
    byte[] download(String key);
}
```

业务代码依赖 `FileStorage`，单元测试 Mock 这个接口；真实 S3 行为用 LocalStack 或 Testcontainers 验证。这样第三方库被隔离在适配器里，升级影响面小，测试也更稳定。

| 方案 | 适用场景 | 成本 |
|------|----------|------|
| Mock 自己定义的端口接口 | 业务单元测试 | 低 |
| WireMock | HTTP 第三方服务 | 中 |
| Testcontainers | 数据库、消息队列、真实中间件 | 中高 |
| Mock 第三方 SDK | 遗留代码临时过渡 | 低但脆弱 |
| 真实沙箱环境 | 端到端验收 | 高 |

> **行业实践**："Don't mock types you don't own" 是测试领域的常见原则。它的核心不是禁止一切第三方 Mock，而是提醒你：**第三方类型不是你的契约**，应该先建立自己的适配层。

## 1.4 测试脆弱性

脆弱的测试有几个共同特征：依赖时间、随机、执行顺序、线程调度，或者对参数和调用次数过度精确。它们平时通过，CI 上偶发失败，最后被团队打上 `@Disabled`。

| 脆弱来源 | 典型表现 | 改进方式 |
|----------|----------|----------|
| 真实时间 | `LocalDateTime.now()` 导致跨天失败 | 注入 `Clock` |
| 随机数 | UUID、随机 ID 无法断言 | 注入 `IdGenerator` |
| 执行顺序 | 测试依赖前一个测试的数据 | 每个测试独立准备数据 |
| 线程调度 | `Thread.sleep` 后验证异步结果 | 使用 Awaitility、Latch |
| 精确参数 | 对象多一个字段就失败 | 用 `argThat` 只断言关键字段 |
| 调用次数 | 内部实现变化导致次数变化 | 用 `atLeastOnce` 或只验证结果 |
| 静态状态 | 单例缓存跨测试污染 | 每个测试重置，或改用依赖注入 |
| 本地化/时区 | 不同机器结果不同 | 固定 `Locale`、`ZoneId` |

### 1.4.1 一个脆弱测试的改造

```java
// 改造前：依赖真实时间和固定 sleep
@Test
void should_expire_token() throws Exception {
    Token token = tokenService.create(user);
    Thread.sleep(1000);
    assertTrue(tokenService.isExpired(token));
}
```

```java
// 改造后：注入可控时钟，测试瞬间完成
@Test
void should_expire_token() {
    MutableClock clock = new MutableClock(Instant.parse("2026-01-01T00:00:00Z"));
    TokenService service = new TokenService(clock, Duration.ofMinutes(30));

    Token token = service.create(user);
    clock.advance(Duration.ofMinutes(31));

    assertTrue(service.isExpired(token));
}
```

这个改造的重点不是"用 Mockito 替换时间"，而是**把时间变成可注入的依赖**。Mockito 只是实现手段，设计才是根因。

## 1.5 测试金字塔与 Mockito 的位置

Mockito 主要服务于金字塔底部的单元测试，也可以在中层切片测试中替换外部依赖。它不应该成为所有测试层的万能胶水。

```text
                    ┌──────────────┐
                    │     E2E      │  少量，验证关键用户旅程
                    └──────┬───────┘
                 ┌─────────┴─────────┐
                 │   集成/切片测试     │  中等，验证组件协作
                 └─────────┬─────────┘
            ┌──────────────┴──────────────┐
            │          单元测试             │  大量，快速、隔离
            │   JUnit + Mockito + AssertJ  │
            └─────────────────────────────┘
```

| 层级 | 关注点 | Mockito 的角色 |
|------|--------|----------------|
| 单元测试 | 单个类/方法的行为 | 隔离外部依赖 |
| 切片测试 | Web 层、JPA 层 | 替换缺失 Bean |
| 集成测试 | 多组件协作 | 尽量少用，保留真实依赖 |
| 端到端测试 | 用户旅程 | 不使用 Mock |

**设计考量**：测试金字塔不是"单元测试越多越好"的教条，而是"越靠底层，反馈越快、定位越准、成本越低"的经验规律。Mockito 让单元测试更容易写，但不能因此把所有集成场景都拆成 Mock 测试，否则会失去对真实协作的验证。

## 1.6 最佳实践清单

下面这份清单可以直接用于代码审查或团队规范：

| 实践 | 说明 |
|------|------|
| 只 Mock 边界 | 数据库、HTTP、消息、文件、时钟、随机源 |
| 优先构造器注入 | 依赖显式，测试可直接 new |
| Mock 接口而非实现 | 降低对具体类型的耦合 |
| 断言行为而非步骤 | 返回值、持久化状态、消息内容优先 |
| 慎用 `verifyNoMoreInteractions` | 它会把新增合法调用判为失败 |
| 默认严格模式 | 让未使用的桩尽早暴露 |
| 慎用 `lenient()` | 只在迁移期或公共夹具中使用 |
| 用 `ArgumentCaptor` 做结构化断言 | 捕获后交给 AssertJ |
| 静态/构造 Mock 用 try-with-resources | 防止作用域泄漏 |
| 异步测试用确定性等待 | 避免 `Thread.sleep` |
| 每个测试独立准备数据 | 不依赖执行顺序 |
| 测试名描述业务行为 | `should_reject_expired_coupon` 优于 `test1` |

## 1.7 常见错误与排错

| 现象 | 原因 | 修正 |
|------|------|------|
| 测试通过但线上仍出错 | 只验证了 Mock 调用，没验证真实协作 | 补契约测试或集成测试 |
| 重构后大量测试失败 | 断言绑定了内部实现步骤 | 改为断言返回值与关键副作用 |
| 偶发失败 | 依赖真实时间、随机数或线程调度 | 注入时钟/随机源，使用 Awaitility |
| 单个测试启动很慢 | 每个测试都创建完整 Spring 上下文 | 用测试切片或纯单元测试 |
| 覆盖率很高但缺陷多 | 过度 Mock 掩盖了真实集成问题 | 减少 Mock，补充真实依赖测试 |
| 测试维护成本高 | 重复的桩、过宽的断言、共享状态 | 提取夹具，每个测试独立准备数据 |

## 1.8 全模块总结

八篇笔记从接入、打桩、匹配器、验证、Spy、静态 Mock、Spring 集成一路走到反模式，核心脉络可以压缩成一张决策图：

```text
需要隔离依赖？
├── 否 → 不用 Mockito，直接 new + 断言
└── 是 → 依赖是接口/可注入吗？
    ├── 是 → @Mock + 构造器注入
    │        ├── 需要部分真实 → @Spy + doReturn
    │        └── 需要验证副作用 → verify / ArgumentCaptor
    └── 否 → 是静态方法/内部 new 吗？
        ├── 是 → 先尝试抽接口/工厂
        │        └── 遗留代码临时用 mockStatic / mockConstruction
        └── 否 → final 类/方法 → Mockito 5 inline 可 Mock
```

| 篇目 | 主题 | 一句话要点 |
|------|------|------------|
| `1_overview_and_setup.md` | 定位与接入 | JUnit 调度，Mockito 隔离，AssertJ 断言 |
| `2_stubbing.md` | 打桩 | 控制返回值与异常，Spy 用 `doReturn` |
| `3_argument_matchers.md` | 参数匹配 | 全匹配器或全原始值，`argThat` 保持纯函数 |
| `4_verification.md` | 行为验证 | 验证副作用，不验证实现步骤 |
| `5_spy_and_inject_mocks.md` | Spy 与注入 | 优先构造器注入，Spy 谨慎使用 |
| `6_static_final_and_constructor.md` | 静态/final/构造 | 局部作用域，资源必须关闭 |
| `7_spring_integration.md` | Spring 集成 | `@WebMvcTest` + `@MockitoBean` + `MockMvc` |
| `8_anti_patterns_and_summary.md` | 反模式与总结 | 少 Mock、断行为、保稳定 |

# 2. 练习与自检

1. 找出一个"验证实现细节"的测试，把它改写成只断言返回值和关键副作用。
2. 为什么不建议直接 Mock 第三方 SDK？请给出端口接口、WireMock、Testcontainers 三种替代方案中的两种。
3. 用一句话概括 Mockito 的适用边界，并说明什么情况下应该改用集成测试。

# 3. 小结

Mockito 的价值不在于"能 Mock 多少东西"，而在于帮助测试聚焦**被测对象的行为**。一旦 Mock 开始掩盖设计问题、绑定实现细节，或者让测试比生产代码更难维护，就应该停下来重新审视边界。

| 原则 | 落地做法 |
|------|----------|
| 少 Mock | 只 Mock 系统边界，值对象用真实实例 |
| 断行为 | 优先断言返回值、状态和消息内容 |
| 保稳定 | 注入时钟/随机源，避免 sleep 和顺序依赖 |
| 控范围 | 单元测试用 `MockitoExtension`，Web 层用切片 |
| 留真实 | 集成测试保留数据库和中间件，不要全 Mock |
| 常重构 | 测试也是代码，重复和脆弱同样需要治理 |

至此 Mockito 模块结束。下一步可以进入同目录的 `SpringBootTest` 模块，把本篇学到的替身思想放到更完整的 Web 与容器测试场景中验证。测试工具会变，但"隔离边界、断言行为、保持可维护"这三条原则不会变。
