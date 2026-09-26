
# 1. 异步调用概述

Dubbo 提供全链路异步化能力，从 Consumer 发起调用到 Provider 处理返回，全程支持异步模型。

```
同步模型:
  Consumer 线程发起调用 → 阻塞等待 → 收到响应 → 继续执行

异步模型:
  Consumer 线程发起调用 → 立即返回 Future → 继续执行
  Provider 处理完成后 → Netty IO 线程设置 Future 结果 → 回调通知
```

# 2. Future 异步调用

## 2.1 基于 RpcContext 的异步（传统方式）

```java
@DubboReference
private UserService userService;

public void asyncCall() {
    // 发起异步调用
    userService.getUser(1L);
    
    // 获取 Future
    Future<User> future = RpcContext.getServiceContext().getFuture();
    
    // 做其他事情
    doOtherWork();
    
    // 阻塞获取结果
    User user = future.get(3000, TimeUnit.MILLISECONDS);
}
```

**限制**：Future 模式每次调用后必须立即获取 `getFuture()`，不能在多层嵌套中使用。

## 2.2 基于 AsyncContext 的异步（推荐）

```java
@DubboReference(async = true)
private UserService userService;

public void asyncCall() {
    // 发起异步调用
    userService.getUser(1L);
    
    // 获取 Future
    Future<User> future = RpcContext.getServiceContext().getFuture();
    
    // 设置回调
    ((CompletableFuture<User>) future).whenComplete((user, ex) -> {
        if (ex != null) {
            log.error("async call error", ex);
        } else {
            log.info("async result: {}", user);
        }
    });
}
```

# 3. CompletableFuture 异步（Dubbo 3 推荐）

## 3.1 接口定义返回 CompletableFuture

```java
// 服务接口直接返回 CompletableFuture
public interface UserService {
    CompletableFuture<User> getUser(Long id);
    CompletableFuture<List<User>> listUsers();
}

// Provider 实现
@DubboService
public class UserServiceImpl implements UserService {

    @Override
    public CompletableFuture<User> getUser(Long id) {
        return CompletableFuture.supplyAsync(() -> {
            return new User(id, "user-" + id, 25);
        });
    }

    @Override
    public CompletableFuture<List<User>> listUsers() {
        return CompletableFuture.supplyAsync(() -> {
            return userRepository.findAll();
        }, businessThreadPool); // 指定线程池
    }
}
```

## 3.2 Consumer 端使用

```java
@DubboReference
private UserService userService;

public void demo() {
    CompletableFuture<User> future = userService.getUser(1L);
    
    future.thenApply(User::getName)
          .thenAccept(name -> System.out.println("User name: " + name))
          .exceptionally(ex -> {
              log.error("Error", ex);
              return null;
          });
}
```

## 3.3 链式调用

```java
@DubboReference
private UserService userService;

@DubboReference
private OrderService orderService;

public void chainCall() {
    userService.getUser(1L)
        .thenCompose(user -> {
            // 获取用户后，再查订单
            return orderService.getOrdersByUserId(user.getId());
        })
        .thenAccept(orders -> {
            System.out.println("Orders: " + orders.size());
        });
}
```

# 4. 异步源码分析

## 4.1 AsyncToSyncInvoker（同步转异步桥接）

```java
public class AsyncToSyncInvoker<T> implements Invoker<T> {
    private final Invoker<T> invoker;

    @Override
    public Result invoke(Invocation invocation) throws RpcException {
        Result asyncResult = invoker.invoke(invocation);
        
        try {
            // 同步阻塞等待结果
            if (InvokeMode.SYNC == invocation.getInvokeMode()) {
                asyncResult.get(Integer.MAX_VALUE); // 阻塞等待
            }
        } catch (InterruptedException e) {
            throw new RpcException(e);
        }
        
        return asyncResult;
    }
}
```

## 4.2 AsyncRpcResult —— 异步结果封装

```java
public class AsyncRpcResult implements Result {
    private final CompletableFuture<Object> responseFuture;
    private final Invocation invocation;

    @Override
    public Object getValue() {
        return getResponseFuture()
            .get(timeout, TimeUnit.MILLISECONDS);
    }

    @Override
    public CompletableFuture<Object> getResponseFuture() {
        return responseFuture;
    }

    // Netty IO 线程收到响应时调用
    public static AsyncRpcResult newDefaultAsyncResult(
            Object value, Throwable t, Invocation invocation) {
        
        CompletableFuture<Object> future = new CompletableFuture<>();
        if (t != null) {
            future.completeExceptionally(t);
        } else {
            future.complete(value);
        }
        return new AsyncRpcResult(future, invocation);
    }
}
```

## 4.3 DefaultFuture → CompletableFuture 演进

```java
// Dubbo 2.x: DefaultFuture (自定义伪 Future)
// Dubbo 3.x: CompletableFuture (JDK 8 标准 Future)

// DubboInvoker 中
@Override
protected Result doInvoke(final Invocation invocation) {
    ExchangeClient currentClient = clients[0];
    
    // 请求
    CompletableFuture<Object> responseFuture = 
        currentClient.request(invocation, timeout);
    
    // 包装为 AsyncRpcResult
    AsyncRpcResult asyncResult = new AsyncRpcResult(
        responseFuture, invocation);
    
    return asyncResult;
}
```

# 5. Provider 端异步

## 5.1 业务线程池异步处理

```yaml
dubbo:
  protocol:
    dispatcher: message   # 只将请求消息分发到业务线程池
    threadpool: cached    # 缓存线程池
    threads: 200
```

## 5.2 Provider 端异步处理示例

```java
@DubboService
public class UserServiceImpl implements UserService {

    @Override
    public String processLargeData(byte[] data) {
        // 直接返回确认
        CompletableFuture.runAsync(() -> {
            // 异步处理大数据
            heavyProcessing(data);
        }, heavyWorkExecutor);
        
        return "accepted";
    }
}
```

# 6. 全链路异步架构

```
Consumer 线程:
  │
  ├─→ 构建 RpcInvocation (invokeMode=ASYNC)
  ├─→ 发送请求到 Netty
  ├─→ CompletableFuture<Result> 立即返回
  │
  └─→ 业务线程继续执行其他任务
        │
        └─→ future.thenAccept(result -> { ... })

Netty IO 线程 (Consumer):
  │
  └─→ 收到 Provider 响应 → CompletableFuture.complete(result)
        → 触发 thenAccept 回调

Netty IO 线程 (Provider):
  │
  └─→ 收到 Consumer 请求 → Dispatcher → 业务线程池
        → 执行 ServiceImpl → CompletableFuture.complete(result)
```

# 7. 响应式（Reactive Streams）

Dubbo 3.1+ 支持 Reactive Streams 编程模型：

```java
// 接口定义
public interface UserService {
    // 返回 Mono / Flux (Project Reactor)
    Mono<User> getUser(Long id);
    Flux<User> listUsers();
}

// Consumer 使用
@DubboReference
private UserService userService;

public void reactiveDemo() {
    userService.getUser(1L)
        .map(User::getName)
        .subscribe(name -> System.out.println("Name: " + name));
}
```

# 8. 异步配置

```yaml
dubbo:
  consumer:
    async: true            # 全局开启异步
```

```java
// 接口级异步
@DubboReference(async = true)
private UserService userService;

// 方法级异步（@DubboReference 的 methods 属性中配置暂不支持）
// 建议使用 CompletableFuture 接口方式
```

# 9. 注意事项

1. **Context 传递**：异步场景下 RpcContext 需要特殊处理，使用 `RpcContext.getServerContext()` 获取调用端上下文
2. **线程池隔离**：异步回调默认在 Netty IO 线程执行，耗时操作需指定独立线程池
3. **线程安全性**：`CompletableFuture` 是线程安全的，但回调中访问共享变量需要同步
4. **异常处理**：异步模式下的异常需要显式处理，否则会被吞掉

```java
future.whenComplete((result, exception) -> {
    if (exception != null) {
        // 必须处理异常
        log.error("Async call failed", exception);
        return;
    }
    // 处理正常结果
    process(result);
});
```
