
# 1. 线程模型概述

Dubbo 的线程模型基于 Netty 的 Reactor 模型，通过 Dispatcher 策略将 IO 线程和业务线程分离。

```
Netty Boss Thread
    │ (接受 TCP 连接)
    ▼
Netty Worker Thread (IO 线程, 默认 CPU+1 个)
    │ (读写网络数据, 编解码, 心跳)
    │
    ├── Dispatcher (根据策略分发)
    │
    ▼
ThreadPool (业务线程池)
    │ (执行 Filter Chain → 反射调用 ServiceImpl)
    ▼
返回结果 → Worker Thread → 编码 → 发送
```

# 2. Dispatcher —— 线程分发策略

```java
@SPI(AllDispatcher.NAME)  // 默认 all
public interface Dispatcher {
    @Adaptive({DISPATCHER_KEY, "dispather", "channel.handler"})
    ChannelHandler dispatch(ChannelHandler handler, URL url);
}
```

## 2.1 五种策略

| 策略 | 常量 | 说明 |
|------|------|------|
| **all** (默认) | `AllDispatcher` | 所有消息：线程池处理 |
| **direct** | `DirectDispatcher` | 所有消息：IO 线程处理 |
| **message** | `MessageDispatcher` | 请求/响应消息：线程池；连接事件：IO 线程 |
| **execution** | `ExecutionDispatcher` | 请求消息：线程池；响应和事件：IO 线程 |
| **connection** | `ConnectionDispatcher` | 连接事件：IO 线程顺序执行；其他：线程池 |

## 2.2 AllDispatcher（默认）

```java
public class AllDispatcher implements Dispatcher {
    public static final String NAME = "all";

    @Override
    public ChannelHandler dispatch(ChannelHandler handler, URL url) {
        return new AllChannelHandler(handler, url);
    }
}

public class AllChannelHandler extends WrappedChannelHandler {

    @Override
    public void received(Channel channel, Object message) {
        // 所有消息都提交到业务线程池
        ExecutorService executor = getPreferredExecutorService(message);
        executor.execute(new ChannelEventRunnable(channel, handler, 
            ChannelState.RECEIVED, message));
    }

    @Override
    public void connected(Channel channel) {
        // 连接事件也提交到线程池
        ExecutorService executor = getSharedExecutorService();
        executor.execute(new ChannelEventRunnable(channel, handler, 
            ChannelState.CONNECTED, null));
    }
}
```

## 2.3 DirectDispatcher（IO 线程直接处理）

```java
public class DirectDispatcher implements Dispatcher {
    @Override
    public ChannelHandler dispatch(ChannelHandler handler, URL url) {
        return new DirectChannelHandler(handler, url);
    }
}

public class DirectChannelHandler extends WrappedChannelHandler {

    @Override
    public void received(Channel channel, Object message) {
        // 直接在 IO 线程中处理，不提交到线程池
        handler.received(channel, message);
    }
}
```

## 2.4 ExecutionDispatcher（仅请求到线程池）

```java
public class ExecutionChannelHandler extends WrappedChannelHandler {

    @Override
    public void received(Channel channel, Object message) {
        if (message instanceof Request) {
            // 请求消息提交到线程池
            ExecutorService executor = getPreferredExecutorService(message);
            executor.execute(new ChannelEventRunnable(channel, handler, 
                ChannelState.RECEIVED, message));
        } else {
            // 响应消息和事件在 IO 线程处理
            handler.received(channel, message);
        }
    }
}
```

## 2.5 策略选择建议

```
高并发业务处理  → all      (默认，最安全)
极低延迟        → direct   (无线程切换开销，但阻塞 IO 线程)
连接密集型      → connection
标准场景        → message  (连接事件 IO 线程处理，业务线程池处理)
IO 密集         → execution
```

# 3. 业务线程池

## 3.1 线程池类型

```yaml
dubbo:
  protocol:
    threadpool: fixed      # 固定大小（默认）
    # 可选: fixed / cached / limited / eager
```

| 线程池 | 类名 | 特点 |
|--------|------|------|
| **fixed** | `FixedThreadPool` | 固定大小线程池（默认），默认为 200 线程 |
| **cached** | `CachedThreadPool` | 缓存线程池，自动回收空闲线程 |
| **limited** | `LimitedThreadPool` | 可伸缩线程池，但上限不超核心数（防止 OOM） |
| **eager** | `EagerThreadPool` | 优先创建线程，队列满时创建线程 |

## 3.2 FixedThreadPool（默认）

```java
public class FixedThreadPool implements ThreadPool {
    
    @Override
    public Executor getExecutor(URL url) {
        String name = url.getParameter("threadname", "Dubbo");
        int threads = url.getParameter("threads", 200);
        int queues = url.getParameter("queues", 0); // 队列大小，0 = SynchronousQueue
        
        return new ThreadPoolExecutor(
            threads, threads,
            0L, TimeUnit.MILLISECONDS,
            queues == 0 ? new SynchronousQueue<>() : 
                new LinkedBlockingQueue<>(queues),
            new NamedThreadFactory(name, true),
            new AbortPolicyWithReport(name, url)
        );
    }
}
```

**FixedThreadPool 特性**：
- 核心线程数 = 最大线程数（线程数固定）
- `queues=0` 时使用 `SynchronousQueue`（无缓冲，直接拒绝）
- `queues>0` 时使用 `LinkedBlockingQueue`（积压队列）

## 3.3 CachedThreadPool

```java
public class CachedThreadPool implements ThreadPool {
    
    @Override
    public Executor getExecutor(URL url) {
        String name = url.getParameter("threadname", "Dubbo");
        int cores = url.getParameter("corethreads", 0);
        int threads = url.getParameter("threads", Integer.MAX_VALUE);
        int queues = url.getParameter("queues", 0);
        int alive = url.getParameter("alive", 60000);
        
        return new ThreadPoolExecutor(
            cores, threads,
            alive, TimeUnit.MILLISECONDS,
            queues == 0 ? new SynchronousQueue<>() : 
                new LinkedBlockingQueue<>(queues),
            new NamedThreadFactory(name, true),
            new AbortPolicyWithReport(name, url)
        );
    }
}
```

## 3.4 EagerThreadPool

```java
public class EagerThreadPool implements ThreadPool {
    
    @Override
    public Executor getExecutor(URL url) {
        String name = url.getParameter("threadname", "Dubbo");
        int cores = url.getParameter("corethreads", 0);
        int threads = url.getParameter("threads", Integer.MAX_VALUE);
        int queues = url.getParameter("queues", 0);
        int alive = url.getParameter("alive", 60000);
        
        // 使用自定义 TaskQueue + Eager 拒绝策略
        // 优先创建新线程而不是放入队列
        TaskQueue<Runnable> taskQueue = new TaskQueue<>(queues <= 0 ? 1 : queues);
        EagerThreadPoolExecutor executor = new EagerThreadPoolExecutor(
            cores, threads, alive, TimeUnit.MILLISECONDS,
            taskQueue,
            new NamedThreadFactory(name, true),
            new AbortPolicyWithReport(name, url));
        taskQueue.setExecutor(executor);
        return executor;
    }
}
```

# 4. AbortPolicyWithReport —— 自定义拒绝策略

当线程池忙且队列满时，默认使用 `AbortPolicyWithReport` 抛出异常并记录线程栈信息：

```java
public class AbortPolicyWithReport extends AbortPolicy {
    protected static final Logger logger = 
        LoggerFactory.getLogger(AbortPolicyWithReport.class);

    @Override
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        String msg = String.format(
            "Thread pool is EXHAUSTED! " +
            "Pool: Size=%d, Active=%d, Pending=%d, Completed=%d, " +
            "Task: %d (Scheduled: %d)",
            e.getPoolSize(), e.getActiveCount(),
            e.getQueue().size(), e.getCompletedTaskCount(),
            e.getTaskCount(), e.getQueue().size() + e.getActiveCount()
        );
        
        logger.warn(msg);
        dumpJStack();       // 打印当前所有线程的 StackTrace
        throw new RejectedExecutionException(msg);
    }

    private void dumpJStack() {
        Map<Thread, StackTraceElement[]> stacks = 
            Thread.getAllStackTraces();
        for (Map.Entry<Thread, StackTraceElement[]> entry : 
                stacks.entrySet()) {
            logger.warn(entry.getKey().getName());
            for (StackTraceElement element : entry.getValue()) {
                logger.warn("    " + element);
            }
        }
    }
}
```

# 5. IO 线程池

```yaml
dubbo:
  protocol:
    iothreads: 8      # IO 线程数（Netty Worker Thread），默认 CPU + 1
```

```java
// NettyClient/NettyServer 中的配置
protected void doOpen() {
    EventLoopGroup workerGroup = 
        new NioEventLoopGroup(iothreads, 
            new DefaultThreadFactory("NettyClientWorker", true));
    
    bootstrap.group(workerGroup);
}
```

# 6. Consumer 端线程模型

```
Consumer 业务线程 (Tomcat/自定义)
    │ 调用 userService.getUser(1L)
    ├─→ 代理拦截
    ├─→ Filter Chain
    ├─→ ClusterInvoker.invoke()
    │   (在当前线程中执行)
    ├─→ ExchangeClient.request() → Netty Worker 线程发送 TCP 数据
    │
    └─→ DefaultFuture.get() 阻塞等待响应
         │
         └─→ Netty Worker 线程收到响应
              └─→ DefaultFuture.received() 唤醒 Consumer 线程
                   └─→ 结果返回
```

关键点：Consumer 端**没有额外的业务线程池**，调用在调用者线程中执行（与 Tomcat 线程共享）。

# 7. Provider 端线程模型

```
Netty Boss Thread
    │ accept
    ▼
Netty Worker Thread (IO 线程)
    │ 接收 TCP 数据 → 解码 → Request
    │
    ├── Dispatcher.all
    │   └── AllChannelHandler.received()
    │       └── 提交到业务线程池
    │
    ▼
业务线程池 (ThreadPool)
    │
    ├── Filter Chain (Provider Filters)
    ├── AbstractProxyInvoker.invoke()
    │   └── Wrapper.invokeMethod()
    │       └── ServiceImpl.getUser()
    │
    └── 返回 Result → 编码 → Netty Worker Thread 发送
```

# 8. 线程模型配置实战

## 8.1 高并发场景

```yaml
dubbo:
  protocol:
    dispatcher: all           # 全部到线程池
    threadpool: fixed         # 固定线程池
    threads: 500              # 核心线程数
    iothreads: 16             # IO 线程数
    queues: 1000              # 有界队列 1000
    accepts: 500              # 最大连接数
```

## 8.2 低延迟场景

```yaml
dubbo:
  protocol:
    dispatcher: message       # 只有请求消息到线程池
    threadpool: eager         # 优先创建线程
    threads: 200
    corethreads: 50
    iothreads: 8
    queues: 0                 # 无缓冲队列
```

## 8.3 注解配置

```java
@DubboService(
    threadpool = "fixed",
    threads = 300,
    dispatcher = "all",
    queues = 100
)
public class OrderServiceImpl implements OrderService {
}
```

# 9. 线程池隔离

## 9.1 方法级线程池隔离

```java
@DubboService(
    threads = 200,
    methods = {
        @Method(name = "heavyProcess", threads = 50, executes = 10),
        @Method(name = "quickQuery", threads = 500, queues = 200)
    }
)
public class MixedServiceImpl implements MixedService {
    // heavyProcess 和 quickQuery 使用独立的线程池
}
```

## 9.2 自定义线程池

```yaml
dubbo:
  protocol:
    threadpool: mypool     # 自定义线程池名称
```

```java
public class MyThreadPool implements ThreadPool {
    @Override
    public Executor getExecutor(URL url) {
        // 自定义线程池实现
        return new ThreadPoolExecutor(...);
    }
}
```

SPI 注册：

```
# META-INF/dubbo/org.apache.dubbo.common.threadpool.ThreadPool
mypool=com.example.MyThreadPool
```

# 10. 线程模型调优指南

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| `RejectedExecutionException` | 线程池满 + 队列满 | 增大 `threads` 或 `queues` |
| 响应时间波动大 | 线程池不足导致排队 | `eager` 模式 + 增大 `threads` |
| 线程数很高但 CPU 不高 | 长时间阻塞等待（下游缓慢） | 下游调优 or `cached` 自动回收 |
| 内存 OOM | `queues` 未设置导致无限排在 LinkedBlockingQueue | 设置有界 `queues` |
| IO 线程 CPU 100% | IO 线程太少或 `direct` 模式下业务阻塞 | 增大 `iothreads` 或换 `all` |

# 11. 线程栈排查

当出现 `RejectedExecutionException` 时，`AbortPolicyWithReport` 会自动打印所有线程的 StackTrace 到日志文件：

```
输出路径: {user.dir}/dubbo-{timestamp}.jstack
```

包含：
- 线程名称
- 线程状态（RUNNABLE/BLOCKED/WAITING）
- 完整的调用栈
