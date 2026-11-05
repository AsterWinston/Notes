# 1. 异步请求模型

同步 Controller 在容器线程里一次执行完，响应随即返回；异步 Controller 先把请求挂起，交给另一个线程处理，处理完成后再触发一次 dispatch 写出响应。MockMvc 支持这套模型，但测试写法与同步请求不同：必须先用 `asyncStarted()` 确认请求已挂起，再用 `asyncDispatch()` 完成第二次分发。

## 1.1 三种异步返回类型

| 返回类型 | 完成方式 | 典型场景 |
|----------|----------|----------|
| `Callable<T>` | Spring 提交到 `TaskExecutor` 执行 | 简单异步计算 |
| `DeferredResult<T>` | 业务线程调用 `setResult` | 事件驱动、长轮询 |
| `WebAsyncTask<T>` | 可配置超时与执行器 | 需要超时控制的异步 |

三者在 MockMvc 里的测试套路一致：先断言 `asyncStarted`，再 `asyncDispatch`。

# 2. 异步 Controller 测试

## 2.1 生产代码

```java
@RestController
@RequestMapping("/api/reports")
public class ReportController {

    private final ReportService reportService;

    public ReportController(ReportService reportService) {
        this.reportService = reportService;
    }

    @GetMapping("/{id}")
    public Callable<Report> generate(@PathVariable Long id) {
        return () -> reportService.generate(id);   // 交给异步线程执行
    }
}
```

`Callable` 被提交后，容器线程立即释放，响应进入异步模式。

## 2.2 基础测试

```java
@WebMvcTest(ReportController.class)
class ReportControllerAsyncTest {

    @Autowired
    private MockMvc mockMvc;

    @MockitoBean
    private ReportService reportService;

    @Test
    void should_return_report_after_async_processing() throws Exception {
        when(reportService.generate(1L)).thenReturn(new Report(1L, "ready"));

        MvcResult result = mockMvc.perform(get("/api/reports/1"))
                .andExpect(request().asyncStarted())
                .andReturn();

        mockMvc.perform(asyncDispatch(result))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.status").value("ready"));
    }
}
```

两个关键点：

- `request().asyncStarted()` 是必须的。没有它，第一次 `perform` 只拿到"已挂起"的中间状态，响应体还是空的。
- `asyncDispatch(result)` 触发第二次 dispatch，此时才真正执行 `Callable` 并写出响应。
- `request().asyncResult(...)` 可以直接断言异步计算的结果类型，但它不覆盖最终 HTTP 序列化。

## 2.3 DeferredResult 测试

`DeferredResult` 由业务代码主动设置结果。Controller 通常直接返回 Service 创建的实例：

```java
@GetMapping("/deferred/{id}")
public DeferredResult<Report> deferred(@PathVariable Long id) {
    return reportService.createDeferred(id);
}
```

测试里 mock 出同一个引用，先 `asyncStarted`，再调用 `setResult`，最后 `asyncDispatch`：

```java
@Test
void should_complete_deferred_result() throws Exception {
    DeferredResult<Report> deferred = new DeferredResult<>();
    when(reportService.createDeferred(1L)).thenReturn(deferred);

    MvcResult result = mockMvc.perform(get("/api/reports/deferred/1"))
            .andExpect(request().asyncStarted())
            .andReturn();

    deferred.setResult(new Report(1L, "ready"));   // 模拟业务线程完成

    mockMvc.perform(asyncDispatch(result))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.status").value("ready"));
}
```

## 2.4 超时处理

`Callable` 和 `DeferredResult` 都支持超时。超时后 Spring 会走异常处理链，通常返回 503 或自定义错误结构。测试超时不要依赖真实等待，应该主动设置错误结果：

```java
@Test
void should_return_503_when_async_times_out() throws Exception {
    DeferredResult<Report> deferred = new DeferredResult<>(50L);
    when(reportService.createDeferred(1L)).thenReturn(deferred);

    MvcResult result = mockMvc.perform(get("/api/reports/deferred/1"))
            .andExpect(request().asyncStarted())
            .andReturn();

    deferred.setErrorResult(new TimeoutException("report generation timed out"));

    mockMvc.perform(asyncDispatch(result))
            .andExpect(status().isServiceUnavailable())
            .andExpect(jsonPath("$.code").value("ASYNC_TIMEOUT"));
}
```

对应的 `@ExceptionHandler(TimeoutException.class)` 返回 503 和 `ASYNC_TIMEOUT` 错误码即可。

**注意**：MockMvc 不会真的等 50 毫秒后再触发超时。`setErrorResult` 是手动模拟"异步处理失败"的手段。要验证真实的超时调度，应使用 `@SpringBootTest` 或真实端口测试。

# 3. 统一异常处理

## 3.1 @RestControllerAdvice

集中式异常处理让 Controller 只关心正常路径，错误结构由 advice 统一生成：

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(OrderNotFoundException.class)
    public ResponseEntity<ApiError> handleNotFound(OrderNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(new ApiError("ORDER_NOT_FOUND", ex.getMessage()));
    }
}
```

`@WebMvcTest` 会自动加载 `@ControllerAdvice`，所以只测 Controller 也能覆盖错误映射：

```java
@Test
void should_map_missing_order_to_404() throws Exception {
    when(orderService.find(99L)).thenThrow(new OrderNotFoundException(99L));

    mockMvc.perform(get("/api/orders/99"))
            .andExpect(status().isNotFound())
            .andExpect(jsonPath("$.code").value("ORDER_NOT_FOUND"));
}
```

如果使用 `standaloneSetup`，记得显式 `setControllerAdvice(...)`，否则异常会直接抛出测试。

## 3.2 使用 ProblemDetail

Spring 6 支持 RFC 9457 的 `ProblemDetail`，返回标准的 `application/problem+json`：

```java
@ExceptionHandler(OrderNotFoundException.class)
public ProblemDetail handleNotFound(OrderNotFoundException ex) {
    ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.NOT_FOUND, ex.getMessage());
    problem.setTitle("Order Not Found");
    problem.setProperty("code", "ORDER_NOT_FOUND");
    problem.setProperty("orderId", ex.getOrderId());
    return problem;
}
```

测试时断言标准字段与自定义属性：

```java
mockMvc.perform(get("/api/orders/99"))
        .andExpect(status().isNotFound())
        .andExpect(content().contentTypeCompatibleWith(MediaType.APPLICATION_PROBLEM_JSON))
        .andExpect(jsonPath("$.title").value("Order Not Found"))
        .andExpect(jsonPath("$.status").value(404))
        .andExpect(jsonPath("$.code").value("ORDER_NOT_FOUND"));
```

> **行业实践**：新项目优先使用 `ProblemDetail`，错误结构天然符合标准，前端和网关都能按统一字段解析。老项目如果已有自定义 `ApiError`，不必为了标准而全量迁移，但应在文档里固定字段含义。

## 3.3 参数校验异常

`@Valid` 校验失败会抛 `MethodArgumentNotValidException`，需要转成 400 和字段级错误：

```java
@ExceptionHandler(MethodArgumentNotValidException.class)
public ProblemDetail handleValidation(MethodArgumentNotValidException ex) {
    ProblemDetail problem = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
    problem.setTitle("Validation Failed");

    Map<String, String> errors = new LinkedHashMap<>();
    for (FieldError fieldError : ex.getBindingResult().getFieldErrors()) {
        errors.putIfAbsent(fieldError.getField(), fieldError.getDefaultMessage());
    }
    problem.setProperty("errors", errors);
    return problem;
}
```

测试：

```java
mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                        {"itemId": null, "quantity": 0}
                        """))
        .andExpect(status().isBadRequest())
        .andExpect(jsonPath("$.errors.itemId").exists())
        .andExpect(jsonPath("$.errors.quantity").exists());
```

`@RequestParam`、`@PathVariable` 上的约束失败抛的是 `ConstraintViolationException`，需要单独处理，否则会落到 500。

## 3.4 异常处理的匹配顺序

Spring 选择异常处理器时遵循"最具体优先"：`OrderNotFoundException` 会先匹配它自己的处理器，其次是 `RuntimeException`，最后才是 `Exception` 兜底。如果同一个异常被多个 advice 声明，`@Order` 或 advice 的注册顺序决定优先级。**设计考量**：兜底的 `Exception` 处理器一定要记录日志并返回通用错误码，不要把异常堆栈直接暴露给客户端。

# 4. 错误响应的断言

## 4.1 错误结构约定

一个可用的错误响应通常包含以下字段：

| 字段 | 含义 | 示例 |
|------|------|------|
| `code` | 稳定错误码 | `ORDER_NOT_FOUND` |
| `message` | 面向用户的说明 | `订单不存在` |
| `detail` | 面向开发者的细节 | `orderId=99` |
| `traceId` | 链路追踪标识 | `trace-001` |
| `timestamp` | 发生时间 | `2026-11-06T10:00:00Z` |
| `errors` | 字段级校验错误 | `{"email":"格式不正确"}` |

测试应至少断言 `code` 和状态码，`message` 可以断言包含关系，`timestamp` 和 `traceId` 只断言存在。

## 4.2 常见错误场景

无效 JSON 对应 400，方法不支持对应 405，Content-Type 不匹配对应 415。这三类错误都应被 advice 转成统一错误结构，测试时至少断言状态码和 `code` 字段，避免前端拿到无法解析的默认错误页。

# 5. 异步与异常的交叉场景

## 5.1 异步抛异常

`Callable` 内部抛出的异常会进入异步 dispatch 的异常处理流程。测试时先 `asyncStarted`，再 `asyncDispatch`，异常处理器会在第二次 dispatch 时生效，最终断言 500 和对应错误码。

## 5.2 异步超时配置

全局超时可以通过 `spring.mvc.async.request-timeout=5s` 设置，`DeferredResult` 的构造器参数优先级更高。测试里不要用 `Thread.sleep` 等待超时，既慢又不稳定；用 `setErrorResult` 主动触发，或用 Awaitility 等待状态变化。

# 6. 常见错误

| 现象 | 原因 | 修正 |
|------|------|------|
| 响应体为空 | 缺少 `asyncDispatch` | 先 `asyncStarted`，再 `asyncDispatch` |
| `asyncDispatch` 报状态错误 | 异步结果尚未设置 | 设置 `DeferredResult` 后再 dispatch |
| 异常处理器不生效 | standalone 未注册 advice | `setControllerAdvice(...)` |
| 校验异常返回 500 | 未处理 `MethodArgumentNotValidException` | 增加对应 `@ExceptionHandler` |
| 超时测试很慢 | 使用 `Thread.sleep` | 用 `setErrorResult` 模拟 |
| 404 没有 JSON 错误体 | `@WebMvcTest` 不加载 `BasicErrorController` | 测试自定义 advice 或改用完整上下文 |
| 异步线程拿不到用户 | ThreadLocal 未传播 | 使用上下文包装器或显式传参 |

# 7. 练习

1. 写一个返回 `Callable<Report>` 的接口，用 `asyncDispatch` 验证 200 和 JSON 字段。
2. 为 `OrderNotFoundException` 写一个返回 `ProblemDetail` 的 advice，并测试 404、`title` 与自定义 `code`。
3. 用 `DeferredResult.setErrorResult` 模拟超时，验证 503 和错误码。

# 8. 小结

异步请求测试分两步：先确认 `asyncStarted`，再 `asyncDispatch` 完成分发。`Callable`、`DeferredResult` 都遵循这个模式，超时通过设置错误结果模拟，不要依赖真实等待。异常处理则围绕 `@RestControllerAdvice` 展开，`ProblemDetail` 提供了标准化的错误结构。

| 主题 | 要点 |
|------|------|
| 异步两步 | `asyncStarted` + `asyncDispatch` |
| 异步结果 | `request().asyncResult(...)` |
| 超时 | `setErrorResult` 主动模拟 |
| 统一异常 | `@RestControllerAdvice` + `@ExceptionHandler` |
| 标准错误 | `ProblemDetail` + `application/problem+json` |
| 校验错误 | `MethodArgumentNotValidException` 与 `ConstraintViolationException` |
| 断言重点 | 状态码、错误码、字段级错误 |

下一篇 `8_security_and_best_practices.md` 收束整个模块，讨论 Spring Security 测试、CSRF、用户模拟，以及 MockMvc 的常见反模式和最佳实践。
