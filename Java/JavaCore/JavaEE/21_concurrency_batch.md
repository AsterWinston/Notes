
# 1. Jakarta Concurrency

## 1.1 概述

Jakarta Concurrency（原 JSR-236）为 JavaEE 环境提供**受容器管理的并发 API**，让应用可以在容器内安全地执行异步任务、定时任务，而无需直接操作线程。

```
┌──────────────────────────────────────────┐
│  应用（EJB/CDI/Servlet）                  │
│        │ @Asynchronous / 注入 Executor   │
│        ▼                                 │
│  ManagedExecutorService                  │
│  容器管理线程池（受监控/可交易/可安全）   │
│        │                                 │
│        ▼                                 │
│  业务异步任务                             │
└──────────────────────────────────────────┘
```

## 1.2 为什么不直接用 `new Thread()`

- JavaEE 容器要求资源受控：直接用原生线程**绕过容器上下文**（事务、安全上下文、JNDI 命名空间）。
- 容器管理的线程池可复用、可监控、保证 Thread Context 传播。

## 1.3 核心接口

| 接口 | 说明 |
|------|------|
| `ManagedExecutorService` | 受管执行器（类似 ExecutorService） |
| `ManagedScheduledExecutorService` | 受管定时执行器（类似 ScheduledExecutorService） |
| `ManagedThreadFactory` | 受管线程工厂 |
| `ContextService` | 创建上下文代理对象 |
| `ManagedExecutorDefinition` / `ManagedScheduledExecutorDefinition` | 注解式定义执行器 |

## 1.4 注入执行器

```java
@Stateless
public class ReportService {

    @Resource(name = "java:comp/DefaultManagedExecutorService")
    private ManagedExecutorService executor;

    public Future<String> generateReport() {
        return executor.submit(() -> {
            // 异步执行，容器维护事务/安全上下文
            return "report-data";
        });
    }
}
```

## 1.5 注解式定义执行器

```java
@ManagedExecutorDefinition(
        name = "java:module/concurrency/MyExecutor",
        maxAsync = 10,
        context = "java:comp/DefaultContextService")
@Stateless
public class AsyncTaskService {

    @Resource(lookup = "java:module/concurrency/MyExecutor")
    private ManagedExecutorService executor;
}
```

## 1.6 定时任务（ManagedScheduledExecutorService）

```java
@Resource(name = "java:comp/DefaultManagedScheduledExecutorService")
private ManagedScheduledExecutorService scheduler;

public void schedule() {
    scheduler.schedule(() -> System.out.println("延迟1秒"),
           1, TimeUnit.SECONDS);

    scheduler.scheduleAtFixedRate(() -> System.out.println("每5秒"),
           0, 5, TimeUnit.SECONDS);
}
```

## 1.7 ContextService（上下文代理）

```java
@Resource
private ContextService contextService;

public void runWithContext(Runnable task) {
    Runnable proxy = contextService.createContextualProxy(
            task, Runnable.class);   // 用容器上下文包装任务
    new Thread(proxy).start();       // 原生线程也能获得容器上下文
}
```

## 1.8 与 EJB `@Asynchronous` 的关系

- EJB `@Asynchronous` 底层使用容器受管执行器。
- Jakarta Concurrency 提供更细粒度、更灵活的控制。

# 2. Jakarta Batch

## 2.1 概述

Jakarta Batch（原 JSR-352）提供**标准化的批处理框架**，支持分块处理、监听器、跳过/重试、检查点等。参考实现：**JBeret**（WildFly）等。

```
Job（作业）
  ├─ Step（步骤）1
  │    ├─ ItemReader（读取）
  │    ├─ ItemProcessor（处理，可选）
  │    └─ ItemWriter（写出）
  └─ Step（步骤）2
       ├─ ...
```

## 2.2 核心概念

| 概念 | 说明 |
|------|------|
| Job | 一个批处理作业 |
| JobInstance / JobExecution | 作业实例与执行 |
| Step | 一个步骤（chunk / batchlet） |
| ItemReader / ItemProcessor / ItemWriter | 分块处理三件套 |
| Batchlet | 简单任务（读→全部处理→写回） |
| JobOperator | 启动作业/查看执行 |
| 检查点（Checkpoint） | 记录进度，失败可断点续跑 |

## 2.3 作业定义（job.xml）

```xml
<job id="importJob" xmlns="http://xmlns.jcp.org/xml/ns/javaee"
     xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
     xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee
                         http://xmlns.jcp.org/xml/ns/javaee/jobXML_1_0.xsd"
     version="1.0">

    <step id="importStep">
        <chunk item-count="100">
            <reader ref="fileReader"/>
            <processor ref="dataProcessor"/>
            <writer ref="dbWriter"/>
        </chunk>
    </step>
</job>
```

## 2.4 Reader / Processor / Writer 实现

```java
@Named
public class FileReader extends AbstractItemReader {
    private BufferedReader reader;

    @Inject
    @BatchProperty
    private String filePath;

    @Override
    public void open(Serializable checkpoint) throws Exception {
        reader = new BufferedReader(new FileReader(filePath));
    }

    @Override
    public Object readItem() throws Exception {
        String line = reader.readLine();
        if (line == null) {
            return null;   // 返回 null 表示结束
        }
        return line.split(",");
    }
}

@Named
public class DataProcessor implements ItemProcessor {
    @Override
    public Object processItem(Object item) {
        String[] cols = (String[]) item;
        User u = new User();
        u.setName(cols[0]);
        u.setAge(Integer.parseInt(cols[1]));
        return u;
    }
}

@Named
public class DbWriter extends AbstractItemWriter {
    @PersistenceContext
    private EntityManager em;

    @Override
    public void writeItems(List<Object> items) {
        for (Object item : items) {
            em.persist(item);
        }
    }
}
```

## 2.5 启动作业

```java
@Stateless
public class JobLauncher {

    @Resource(name = "java:comp/DefaultManagedExecutorService")
    private ManagedExecutorService executor;

    public long startImport() {
        JobOperator operator = BatchRuntime.getJobOperator();
        Properties props = new Properties();
        props.setProperty("filePath", "/data/users.csv");
        return operator.start("importJob", props);
    }

    public JobExecution status(long executionId) {
        return BatchRuntime.getJobOperator().getJobExecution(executionId);
    }
}
```

## 2.6 检查点与断点续跑

- chunk 按 `item-count` 间隔做检查点。
- 失败后 `JobOperator.restart(executionId)` 从最后检查点继续。

## 2.7 监听器

```java
@Named
public class JobListener implements jakarta.batch.api.listener.JobListener {
    @Override
    public void beforeJob() { }
    @Override
    public void afterJob() { }
}
```

## 2.8 跳过与重试

```xml
<chunk item-count="100" retry-count="3" skip-limit="50">
    <retryable-exception-classes>
        <include class="java.sql.SQLException"/>
    </retryable-exception-classes>
    <skippable-exception-classes>
        <include class="com.example.BadDataException"/>
    </skippable-exception-classes>
</chunk>
```

# 3. 与 Spring 生态对照

| Jakarta Concurrency / Batch | Spring 等价 |
|------------------------------|-------------|
| `ManagedExecutorService` | `ThreadPoolTaskExecutor` + `@Async` |
| `ManagedScheduledExecutorService` | `@Scheduled` |
| ContextService | `TaskDecorator` 上下文传播 |
| Jakarta Batch（JBeret） | Spring Batch |
| 作业定义 job.xml | Spring Batch Job 配置（Java 配置） |
| ItemReader/Processor/Writer | `ItemReader`/`ItemProcessor`/`ItemWriter`（同名同思路） |
