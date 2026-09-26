
# 1. RootContext — xid 的 ThreadLocal 容器

```java
public class RootContext {

    public static final String KEY_XID = "TX_XID";

    private static final ThreadLocal<String> CONTEXT_HOLDER = new ThreadLocal<>();

    public static String getXID() {
        return CONTEXT_HOLDER.get();
    }

    public static void bind(String xid) {
        if (StringUtils.isEmpty(xid)) {
            throw new IllegalArgumentException("xid cannot be empty");
        }
        // 不允许嵌套绑定
        if (CONTEXT_HOLDER.get() != null) {
            if (!CONTEXT_HOLDER.get().equals(xid)) {
                throw new TransactionException(
                    "Duplicated xid binding: " + CONTEXT_HOLDER.get()
                    + " vs " + xid);
            }
        }
        CONTEXT_HOLDER.set(xid);
    }

    public static String unbind() {
        String xid = CONTEXT_HOLDER.get();
        if (xid != null) {
            CONTEXT_HOLDER.remove();
        }
        return xid;
    }

    // 在不绑定新 xid 的情况下检查
    public static boolean inGlobalTransaction() {
        return CONTEXT_HOLDER.get() != null;
    }
}
```

# 2. xid 在 Seata 流程中的完整路径

```
TM (@GlobalTransactional 方法)
  │
  ├── TransactionalTemplate.execute()
  │     └── GlobalTransaction.begin() → TC 返回 xid
  │           └── RootContext.bind(xid) ← 绑定到当前线程
  │
  ├── 本地 DataSourceProxy 检测到 xid → 注册分支1
  │
  ├── Dubbo/Feign RPC 调用
  │     ├── Dubbo: SeataTransactionPropagationFilter
  │     │     └── invocation.setAttachment("TX_XID", xid)
  │     │
  │     └── Feign: SeataFeignRequestInterceptor
  │           └── requestTemplate.header("TX_XID", xid)
  │
  ├── RM (Provider) 接收
  │     ├── Dubbo: SeataTransactionPropagationProviderFilter
  │     │     └── RootContext.bind(rpcXid)
  │     │
  │     └── HTTP: SeataHandlerInterceptor
  │           └── RootContext.bind(headerXid)
  │
  │     └── DataSourceProxy 检测到 xid → 注册分支2
  │
  └── 清退时 RootContext.unbind()
```

# 3. Dubbo Filter 传播

```java
// Consumer 端
@Activate(group = CommonConstants.CONSUMER, order = 100)
public class SeataTransactionPropagationFilter implements Filter {

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation)
            throws RpcException {
        String xid = RootContext.getXID();
        if (xid != null) {
            // 同时设置大小写两个 key（兼容不同版本）
            invocation.setAttachment(RootContext.KEY_XID, xid);
            invocation.setAttachment(RootContext.KEY_XID.toLowerCase(), xid);
        }
        return invoker.invoke(invocation);
    }
}

// Provider 端
@Activate(group = CommonConstants.PROVIDER, order = -100)
public class SeataTransactionPropagationProviderFilter implements Filter {

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation)
            throws RpcException {
        // 尝试从 attachment 获取 xid
        String xid = invocation.getAttachment(RootContext.KEY_XID);
        if (xid == null) {
            xid = invocation.getAttachment(RootContext.KEY_XID.toLowerCase());
        }

        if (xid != null) {
            RootContext.bind(xid);  // 绑定到当前线程
        }

        try {
            return invoker.invoke(invocation);
        } finally {
            // 清理，防止内存泄漏
            if (xid != null) {
                RootContext.unbind();
            }
        }
    }
}
```

# 4. Feign 拦截器传播

```java
// SeataFeignClient 实现的 RequestInterceptor
public class SeataFeignRequestInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate template) {
        String xid = RootContext.getXID();
        if (xid != null) {
            template.header(RootContext.KEY_XID, xid);
        }
    }
}

// HTTP 服务端拦截
public class SeataHandlerInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request,
            HttpServletResponse response, Object handler) {
        String xid = request.getHeader(RootContext.KEY_XID);
        if (xid != null) {
            RootContext.bind(xid);
        }
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request,
            HttpServletResponse response, Object handler, Exception ex) {
        RootContext.unbind();
    }
}
```

# 5. 多线程场景的 xid 传播

```java
// 场景：TM 内部启动异步线程调用 RM
@GlobalTransactional
public void processOrder(Order order) {
    orderMapper.insert(order);  // 分支1: 当前线程

    // 异步线程调用
    CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
        // ⚠️ 新线程的 RootContext 中没有 xid！
        // 需要手动传递
        accountClient.debit(userId, amount);  // 不会参与全局事务
    });

    future.join();
}

// ✓ 正确做法：手动传递 xid
@GlobalTransactional
public void processOrder(Order order) {
    orderMapper.insert(order);

    // 获取当前线程的 xid
    String xid = RootContext.getXID();

    CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
        // 在新线程中手动绑定
        RootContext.bind(xid);
        try {
            accountClient.debit(userId, amount);
        } finally {
            RootContext.unbind();
        }
    });

    future.join();
}
```

# 6. 线程池 xid 透传

```java
// 自定义线程池：自动透传 xid
public class SeataThreadPoolExecutor extends ThreadPoolExecutor {

    public SeataThreadPoolExecutor(int corePoolSize, int maxPoolSize,
            long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> queue) {
        super(corePoolSize, maxPoolSize, keepAliveTime, unit, queue);
    }

    @Override
    public void execute(Runnable command) {
        // 获取父线程的 xid
        final String xid = RootContext.getXID();
        
        super.execute(() -> {
            // 子线程绑定 xid
            if (xid != null) {
                RootContext.bind(xid);
            }
            try {
                command.run();
            } finally {
                if (xid != null) {
                    RootContext.unbind();
                }
            }
        });
    }
}
```

# 7. xid 传播的注意事项

1. **ThreadLocal 隔离**：xid 基于 ThreadLocal，异步/多线程场景需手动传递
2. **Context 清理**：Provider 端必须在 finally 中 unbind()
3. **嵌套事务**：不允许在同一线程绑定不同 xid（防止 xid 覆盖）
4. **Filter 顺序**：xid 传播 Filter 应优先于业务 Filter 执行
5. **HTTP 与 Dubbo 的 Key 差异**：建议使用 `RootContext.KEY_XID` 统一常量
