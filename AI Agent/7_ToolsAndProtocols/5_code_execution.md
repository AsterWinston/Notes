# 1. 为什么需要代码执行

结论先行：大语言模型（Large Language Model，LLM）不擅长精确计算与确定性逻辑，却擅长写代码，因此把"算"交给代码、把"想"交给模型是最佳分工。

## 1.1 确定性计算的必要性

### 1.1.1 适用场景

| 适用 | 例子 |
|------|------|
| 数学计算 | 统计、聚合、公式 |
| 数据处理 | CSV/JSON 转换 |
| 图表生成 | matplotlib 绘图 |
| 逻辑验证 | 验证算法正确性 |

### 1.1.2 对照示例

```
问："计算 2024 年 3 月到 2026 年 9 月的天数"
LLM 直接算：可能出错
让 LLM 写代码执行：精确可靠
```

## 1.2 代码执行与直接计算

### 1.2.1 对比表

| 维度 | 直接让模型算 | 让模型写代码再执行 |
|------|--------------|--------------------|
| 精确性 | 低（易错） | 高（确定性） |
| 可验证 | 难 | 易（有输出） |
| 成本 | 低 | 中（多一次执行） |
| 安全 | 无风险 | 有风险，需沙箱 |

### 1.2.2 边界与坑

- 简单算术不必上代码执行：多一次容器启动反而更慢。
- 代码执行并非万能：涉及实时数据仍需外部工具。
- 结果正确不代表逻辑正确：要能解释代码做了什么。

### 1.2.3 典型调用链

```
[User]  统计这份 CSV 的月度均值
[Model] tool_calls: executePython(code="...pandas...")
[Tool]  执行成功，返回均值表
[Model] 按结果生成自然语言结论
```

# 2. 执行方式

结论先行：执行方式按隔离强度分为容器沙箱、进程沙箱、远程执行服务与本地进程四类，生产推荐容器沙箱。

## 2.1 四种执行方式

### 2.1.1 方式对比表

| 方式 | 隔离 | 适用 |
|------|:----:|------|
| 容器沙箱 | 强 | 生产推荐 |
| 进程沙箱 | 中 | 内部可信 |
| 远程执行服务 | 强 | 大规模 |
| 本地进程 | 弱 | 开发/测试 |

### 2.1.2 选择依据

| 维度 | 容器沙箱 | 远程服务 | 本地进程 |
|------|----------|----------|----------|
| 隔离强度 | 高 | 高 | 低 |
| 启动延迟 | 中 | 中 | 低 |
| 运维成本 | 中 | 高 | 低 |
| 弹性 | 中 | 高 | 低 |

## 2.2 执行生命周期

### 2.2.1 生命周期图

```
生成代码 → 静态检查 → 创建沙箱 → 注入代码 → 执行
   │                                            │
   │                                            ▼
   └────────── 返回结果 ← 收集输出 ← 超时/结束
```

### 2.2.2 边界与坑

- 跳过静态检查：危险代码直接进入沙箱，纵深防御变单点。
- 沙箱创建失败无降级：直接报错，用户体验差。
- 不回收沙箱：容器泄漏，资源被逐步耗尽。

### 2.2.3 结果模型

```java
public record ExecutionResult(
        boolean success,
        String output,
        String error,
        boolean timeout,
        long durationMillis
) {
    public static ExecutionResult ok(String output) {
        return new ExecutionResult(true, output, null, false, 0);
    }

    public static ExecutionResult timeout() {
        return new ExecutionResult(false, null, "执行超时", true, 0);
    }

    public static ExecutionResult error(String message) {
        return new ExecutionResult(false, null, message, false, 0);
    }
}
```

# 3. 容器执行实现

结论先行：用 Docker（容器引擎）以最小权限运行代码，禁用网络、限制资源、非 root、只读根，并用超时兜底。

## 3.1 Docker 执行器

### 3.1.1 完整 Java 示例

```java
@Component
public class SandboxCodeExecutor {

    public ExecutionResult execute(String code, String language,
                                   Duration timeout) {
        String image = switch (language) {
            case "python" -> "python:3.12-slim";
            case "java" -> "eclipse-temurin:21-jdk";
            case "node" -> "node:20-slim";
            default -> throw new IllegalArgumentException("不支持的语言: " + language);
        };

        String containerName = "sandbox-" + UUID.randomUUID();

        try {
            ProcessBuilder pb = new ProcessBuilder(
                    "docker", "run", "--rm",
                    "--name", containerName,
                    "--network", "none",                  // 无网络
                    "--memory", "256m",                   // 内存限制
                    "--cpus", "0.5",                      // CPU 限制
                    "--pids-limit", "64",                 // 进程限制
                    "--read-only",                        // 只读根
                    "--tmpfs", "/tmp:size=64m",           // 临时可写
                    "--user", "1000:1000",                // 非 root
                    "--cap-drop", "ALL",                  // 丢弃能力
                    "--security-opt", "no-new-privileges",
                    image,
                    language, "-c", code);

            pb.redirectErrorStream(true);
            Process process = pb.start();

            String output = readOutput(process, timeout);

            if (!process.waitFor(timeout.toMillis(), TimeUnit.MILLISECONDS)) {
                killContainer(containerName);
                return ExecutionResult.timeout();
            }
            return ExecutionResult.ok(output);

        } catch (Exception e) {
            return ExecutionResult.error(e.getMessage());
        }
    }
}
```

### 3.1.2 参数说明

| 参数 | 作用 |
|------|------|
| `--network none` | 禁止出网与访问内网 |
| `--memory 256m` | 限制内存，防耗尽 |
| `--cpus 0.5` | 限制 CPU 份额 |
| `--pids-limit 64` | 防 fork 炸弹 |
| `--read-only` | 根文件系统只读 |
| `--tmpfs /tmp` | 提供有限可写空间 |
| `--user 1000:1000` | 非 root 运行 |
| `--cap-drop ALL` | 丢弃全部 Linux 能力 |

### 3.1.3 进程等待与读取

```java
Process process = pb.start();
String output = readOutput(process, timeout);
boolean finished = process.waitFor(timeout.toMillis(), TimeUnit.MILLISECONDS);
if (!finished) {
    process.destroyForcibly();
}
```

## 3.2 超时与清理

### 3.2.1 超时处理

```java
private void killContainer(String name) {
    try {
        new ProcessBuilder("docker", "kill", name)
                .start().waitFor(5, TimeUnit.SECONDS);
    } catch (Exception ignored) {
        // 记录日志，避免影响主流程
    }
}
```

### 3.2.2 边界与坑

- 只 kill 进程不 kill 容器：`docker run` 被中断后容器可能仍在运行。
- 超时时间过长：恶意代码会长时间占用资源。
- 未使用 `--rm`：异常退出后残留容器。

# 4. 输出限制

结论先行：必须限制输出大小与读取时间，否则无限打印会撑爆内存并拖垮服务。

## 4.1 流式读取与截断

### 4.1.1 完整 Java 示例

```java
private String readOutput(Process process, Duration timeout) {
    StringBuilder sb = new StringBuilder();
    try (BufferedReader reader = new BufferedReader(
            new InputStreamReader(process.getInputStream()))) {

        char[] buffer = new char[4096];
        int read;
        long deadline = System.currentTimeMillis() + timeout.toMillis();

        while ((read = reader.read(buffer)) != -1) {
            if (System.currentTimeMillis() > deadline) break;
            if (sb.length() + read > MAX_OUTPUT_SIZE) {
                sb.append("\n...[输出被截断]");
                break;
            }
            sb.append(buffer, 0, read);
        }
    } catch (IOException e) {
        return "[读取输出失败: " + e.getMessage() + "]";
    }
    return sb.toString();
}
```

### 4.1.2 截断策略

| 策略 | 说明 |
|------|------|
| 字节上限 | 超过阈值即截断 |
| 行数上限 | 只保留前 N 行 |
| 首尾保留 | 保留开头与结尾，省略中间 |

## 4.2 边界与坑

### 4.2.1 死锁风险

若子进程写满标准错误（stderr）而父进程只读标准输出（stdout），可能死锁。应合并流或并发读取。

```java
pb.redirectErrorStream(true);   // 合并 stdout 与 stderr，避免死锁
```

### 4.2.2 边界与坑

- 不读输出直接 waitFor：缓冲区写满导致子进程阻塞。
- 只在超时后读取：应边读边限制，而非攒到最后。
- 截断不提示模型：模型不知道被截断，可能误判结果完整。

### 4.2.3 二进制输出

图片等二进制结果不应走标准输出，应写入文件后由宿主读取并转成 URL 或 Base64。

```java
// Python 侧：plt.savefig("/workspace/chart.png")
// Java 侧：读取 chart.png 并上传对象存储
```

# 5. 作为工具暴露

结论先行：把代码执行包装成工具，描述中写清适用场景与硬性限制，让模型知道边界。

## 5.1 工具定义

### 5.1.1 Spring AI 风格

```java
@Tool("""
        执行 Python 代码并返回结果。
        适用：数学计算、数据处理、生成图表。
        限制：无网络访问，内存 256MB，执行时间 30 秒。
        """)
ToolResult executePython(
        @ToolParam(description = "Python 代码，最后用 print 输出结果")
        String code) {

    ExecutionResult result = executor.execute(code, "python",
            Duration.ofSeconds(30));

    return result.success()
            ? ToolResult.ok(result.output())
            : ToolResult.fail(result.error());
}
```

### 5.1.2 LangChain4j 风格

```java
public class CodeExecutionTools {

    private final SandboxCodeExecutor executor;

    @Tool("执行 Python 代码并返回标准输出")
    public String runPython(@P("Python 源码") String code) {
        ExecutionResult r = executor.execute(code, "python",
                Duration.ofSeconds(30));
        return r.success() ? r.output() : "执行失败: " + r.error();
    }
}
```

## 5.2 提示与约束

### 5.2.1 描述要点

| 要点 | 说明 |
|------|------|
| 语言 | 明确支持哪些语言 |
| 输入 | 代码如何传入 |
| 输出 | 如何拿到结果（print/文件） |
| 限制 | 内存、时间、网络 |

### 5.2.2 边界与坑

- 描述不写限制：模型会写超长任务，频繁超时。
- 输出格式不约定：模型不知道结果在哪，需约定 print 到标准输出。
- 不返回错误详情：模型无法自我修正。

### 5.2.3 幂等与重试

```java
public ToolResult executePythonIdempotent(String code, String traceId) {
    String key = "exec:" + traceId + ":" + sha256(code);
    return cache.getOrCompute(key, () -> executePython(code));
}
```

> 同一段代码在同一会话内重复提交，可命中缓存，避免重复执行。

# 6. 文件输入输出

结论先行：需要文件时，把宿主目录挂载到容器工作目录，执行后读回结果文件。

## 6.1 输入文件传递

### 6.1.1 完整 Java 示例

```java
public ExecutionResult executeWithFiles(String code,
                                        Map<String, byte[]> inputs) {
    Path workDir = Files.createTempDirectory("sandbox-");

    // 写入输入文件
    inputs.forEach((name, content) -> {
        try {
            Files.write(workDir.resolve(name), content);
        } catch (IOException e) {
            throw new UncheckedIOException(e);
        }
    });

    try {
        // 挂载目录到容器
        ProcessBuilder pb = new ProcessBuilder(
                "docker", "run", "--rm",
                "-v", workDir + ":/workspace:rw",
                "-w", "/workspace",
                "--network", "none",
                "--memory", "512m",
                "python:3.12-slim",
                "sh", "-c", "pip install --no-index -q -r requirements.txt 2>/dev/null; python main.py");
        // ...
        return readResults(workDir);
    } finally {
        deleteRecursively(workDir);
    }
}
```

### 6.1.2 输出文件读取

执行结束后从工作目录读取约定的输出文件（如 `output.csv`、`chart.png`）。

```java
private ExecutionResult readResults(Path workDir) throws IOException {
    Path out = workDir.resolve("output.txt");
    if (Files.exists(out)) {
        return ExecutionResult.ok(Files.readString(out));
    }
    return ExecutionResult.ok("[无输出文件]");
}
```

## 6.2 依赖管理

### 6.2.1 预装镜像

生产建议**预装依赖镜像**，避免每次 `pip install`（且无网络时不可行）。

```dockerfile
FROM python:3.12-slim
RUN pip install --no-cache-dir pandas matplotlib numpy
```

### 6.2.2 边界与坑

- 运行时安装依赖：无网络时失败，有网络时又不安全。
- 挂载可写目录过宽：代码可篡改宿主文件，应只挂工作目录。
- 临时目录不清理：磁盘被逐步占满。

# 7. 语言选择

结论先行：Python 最通用，作为默认语言；其他语言按业务需要提供。

## 7.1 语言对比

### 7.1.1 对比表

| 语言 | 适用 | 镜像大小 |
|------|------|----------|
| Python | 数据处理、绘图、科学计算 | 中 |
| JavaScript | 前端相关、JSON 处理 | 小 |
| Java | Java 生态验证 | 大 |
| SQL | 数据查询 | 需数据库 |

### 7.1.2 默认选择

```java
private static final String DEFAULT_LANGUAGE = "python";
```

## 7.2 边界与坑

### 7.2.1 版本与镜像

| 语言 | 推荐基础镜像 |
|------|--------------|
| Python | `python:3.12-slim` |
| Java | `eclipse-temurin:21-jdk` |
| Node | `node:20-slim` |

### 7.2.2 边界与坑

- 镜像不固定版本：`latest` 会随时间漂移，结果不可复现。
- 镜像过大：拉取慢，影响启动延迟。
- 多语言全开：攻击面扩大，只开需要的语言。

# 8. 安全要点

结论先行：静态检查只能辅助，真正的防线是容器隔离。

## 8.1 静态检查

### 8.1.1 完整 Java 示例

```java
public class CodeSafetyChecker {

    private static final List<Pattern> DANGEROUS = List.of(
            Pattern.compile("import\\s+os"),
            Pattern.compile("import\\s+subprocess"),
            Pattern.compile("__import__"),
            Pattern.compile("eval\\s*\\("),
            Pattern.compile("exec\\s*\\("),
            Pattern.compile("open\\s*\\(.*['\"]/(etc|proc|sys)")
    );

    public void validate(String code) {
        for (Pattern p : DANGEROUS) {
            if (p.matcher(code).find()) {
                throw new SecurityException("代码包含危险操作: " + p.pattern());
            }
        }
    }
}
```

### 8.1.2 局限

| 局限 | 说明 |
|------|------|
| 易绕过 | 字符串拼接、编码可绕过正则 |
| 误报 | 合法用途（如读取配置）被拦 |
| 不完备 | 无法覆盖所有危险调用 |

> 静态检查只是辅助，**真正的防线是容器隔离**。

## 8.2 真正的防线

### 8.2.1 隔离优先

即使静态检查通过，也必须运行在禁网、限资源、非 root 的容器中，做到"假设代码不可信"。

### 8.2.2 边界与坑

- 依赖静态检查放行：一旦绕过即完全失守。
- 黑名单思路：永远列不全，应改为白名单或强隔离。
- 沙箱内有敏感凭据：代码可读取环境变量，需清理敏感变量。

# 9. 生产注意事项

结论先行：上线前核对沙箱、资源限制、网络、输出、权限、超时与静态检查七件事。

## 9.1 风险清单

| 风险 | 建议 |
|------|------|
| 无沙箱 | 必须容器隔离 |
| 无资源限制 | CPU/内存/时间/进程 |
| 有网络 | 默认禁网 |
| 输出无限 | 限制输出大小 |
| root 运行 | 非 root + 丢能力 |
| 无超时 | 强制超时并杀容器 |
| 危险代码 | 静态检查 + 沙箱 |

## 9.2 部署检查表

- 默认语言与镜像版本固定。
- 预装依赖镜像，禁止运行时联网安装。
- 每个请求都有超时与输出上限。
- 沙箱内不注入任何业务凭据。
- 执行记录进入审计与监控。

# 10. 小结

## 10.1 核心要点

- 代码执行让 LLM 完成精确计算与数据处理。
- 必须容器沙箱：禁网、限资源、非 root。
- 限制输出大小与执行时间。
- 预装依赖镜像，避免运行时安装。
- 静态检查辅助，容器隔离是根本防线。

## 10.2 落地建议

1. 先实现最小可用的 Docker 执行器，参数按第 3 节清单配置。
2. 用预装依赖镜像替代运行时安装。
3. 为输出与超时设置硬上限，防止资源耗尽。
4. 建立执行指标（成功率、超时率、耗时）并告警。
