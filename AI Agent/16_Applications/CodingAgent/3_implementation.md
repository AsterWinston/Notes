# 1. 代码索引

## 1.1 索引流程

### 1.1.1 扫描与过滤

结论先行：索引第一步是遍历仓库并过滤出源码文件，排除构建产物与二进制。

```java
private boolean isCodeFile(Path f) {
    String name = f.getFileName().toString();
    if (name.startsWith(".")) return false;
    if (name.contains("target") || name.contains("node_modules")) return false;
    return name.endsWith(".java") || name.endsWith(".kt") || name.endsWith(".xml");
}
```

边界与坑：过滤规则过宽会引入噪声，过窄会漏掉配置文件；应按仓库约定维护白名单。

### 1.1.2 结构切分

按函数/类为单位切分，保留符号名与起始行，便于回填与定位。

## 1.2 向量化与入库

```java
@Service
@RequiredArgsConstructor
public class CodeIndexingService {

    private final EmbeddingModel embeddingModel;
    private final VectorStore vectorStore;

    public int indexRepository(Path repoRoot) throws IOException {
        List<CodeChunk> chunks = new ArrayList<>();

        try (Stream<Path> files = Files.walk(repoRoot)) {
            files.filter(this::isCodeFile)
                 .forEach(f -> chunks.addAll(splitByStructure(f, repoRoot)));
        }
        // 向量化 + 入库
        List<TextSegment> segments = chunks.stream()
                .map(c -> TextSegment.from(c.content(),
                        Metadata.from(Map.of(
                                "filePath", c.filePath(),
                                "startLine", c.startLine(),
                                "language", c.language(),
                                "symbol", c.symbol()))))
                .toList();

        List<Embedding> embeddings = embeddingModel.embedAll(segments).content();
        vectorStore.add(embeddings, segments);

        return segments.size();
    }

    private List<CodeChunk> splitByStructure(Path file, Path root) {
        // 按函数/类切分（简化：按空行 + 缩进）
        String content = readFile(file);
        return CodeSplitter.split(content, root.relativize(file).toString());
    }
}
```

## 1.3 完整实现

结论先行：索引服务应支持全量与增量两种模式，并对外暴露进度与统计。

```java
public record CodeChunk(String filePath, int startLine, String language,
                        String symbol, String content) {}

public interface CodeSplitter {
    static List<CodeChunk> split(String content, String filePath) {
        // 简化实现：按顶层块切分，生产环境用 AST 解析
        throw new UnsupportedOperationException("见生产实现");
    }
}
```

## 1.4 增量与边界

| 场景 | 处理 |
|------|------|
| 新增文件 | 切分并入库 |
| 修改文件 | 先删后加 |
| 删除文件 | 按 filePath 删除 |
| 重命名 | 删除旧路径、新增新路径 |

生产建议：索引任务异步执行，带进度回调；大仓库分批提交避免内存峰值。

## 1.5 索引存储与元数据

| 元数据 | 用途 |
|--------|------|
| filePath | 定位与过滤 |
| startLine | 回填行号 |
| language | 语法提示 |
| symbol | 符号检索 |

```java
public int deleteByFile(String filePath) {
    return vectorStore.delete(
            FilterExpressionBuilder.eq("filePath", filePath));
}
```

结论先行：以 filePath 为分区键，可高效实现增删改的幂等更新。

# 2. 代码检索工具

## 2.1 工具清单

| 工具 | 说明 |
|------|------|
| `searchCode` | 检索代码片段 |
| `readFile` | 读取文件（限目录） |
| `writeFile` | 写入文件（限目录） |
| `applyPatch` | 应用补丁 |
| `runTests` | 执行测试（沙箱） |
| `gitDiff` | 查看变更 |

## 2.2 检索工具

### 2.2.1 语义检索

将查询向量化后与索引比对，返回最相近的片段。

### 2.2.2 混合检索

结论先行：代码场景中标识符精确匹配很重要，检索应在语义之外叠加关键词召回。

## 2.3 读写工具

### 2.3.1 路径安全

所有路径先 `resolve` 再 `normalize`，最后校验是否位于仓库根之下。

### 2.3.2 写入与补丁

写入前创建父目录，写入后返回提示；补丁以统一 diff 表达并逐文件应用。

## 2.4 完整实现

```java
@Component
@RequiredArgsConstructor
public class CodeTools {

    private final VectorStore vectorStore;
    private final Path repoRoot;

    @Tool("搜索相关代码片段")
    public List<CodeSnippet> searchCode(
            @ToolParam(description = "搜索描述，如'用户认证'") String query) {

        List<Document> docs = vectorStore.similaritySearch(
                SearchRequest.builder().query(query).topK(10).build());

        return docs.stream()
                .map(d -> new CodeSnippet(
                        d.getMetadata().get("filePath").toString(),
                        Integer.parseInt(d.getMetadata().get("startLine").toString()),
                        d.getText()))
                .toList();
    }

    @Tool("读取代码文件")
    public String readFile(@ToolParam(description = "相对路径") String path)
            throws IOException {
        Path target = resolveSafe(path);
        return Files.readString(target);
    }

    @Tool("写入代码文件")
    public String writeFile(@ToolParam(description = "相对路径") String path,
                            @ToolParam(description = "新内容") String content)
            throws IOException {
        Path target = resolveSafe(path);
        Files.createDirectories(target.getParent());
        Files.writeString(target, content);
        return "已写入：" + path;
    }

    private Path resolveSafe(String relative) {
        Path target = repoRoot.resolve(relative).normalize().toAbsolutePath();
        if (!target.startsWith(repoRoot)) {
            throw new SecurityException("路径越界: " + relative);
        }
        return target;
    }
}
```

边界与坑：`repoRoot` 本身必须已绝对化且规范化，否则 `startsWith` 可能误判。

生产建议：对写入增加文件大小与后缀校验，拒绝写入二进制或超大文件。

## 2.5 变更查看工具

```java
@Tool("查看当前变更")
public String gitDiff() throws IOException {
    ProcessBuilder pb = new ProcessBuilder("git", "diff", "--stat", "--patch");
    pb.directory(repoRoot.toFile());
    return readAll(pb.start());
}
```

结论先行：diff 工具让 Agent 与人都能审阅变更，是人工确认前的关键一步。

# 3. 测试执行

## 3.1 沙箱执行器

结论先行：测试执行是唯一的代码运行入口，全部在禁网容器内完成。

## 3.2 资源与超时

### 3.2.1 限制参数

| 参数 | 值 | 作用 |
|------|----|------|
| --network | none | 禁网 |
| --memory | 2g | 内存上限 |
| --cpus | 2 | CPU 上限 |
| --pids-limit | 128 | 进程上限 |

### 3.2.2 超时与输出

墙钟超时 10 分钟，输出上限 10 万字符，超限即截断。

## 3.3 完整实现

```java
@Component
public class SandboxTestRunner {

    public TestResult runTests(Path repoDir) {
        try {
            ProcessBuilder pb = new ProcessBuilder(
                    "docker", "run", "--rm",
                    "--network", "none",
                    "--memory", "2g",
                    "--cpus", "2",
                    "--pids-limit", "128",
                    "-v", repoDir.toAbsolutePath() + ":/workspace:rw",
                    "-w", "/workspace",
                    "maven:3.9-eclipse-temurin-21",
                    "mvn", "-q", "test");

            pb.redirectErrorStream(true);
            Process process = pb.start();

            String output = readLimited(process, 100_000);
            boolean finished = process.waitFor(10, TimeUnit.MINUTES);

            if (!finished) {
                process.destroyForcibly();
                return TestResult.timeout();
            }
            return output.contains("BUILD SUCCESS")
                    ? TestResult.passed(output)
                    : TestResult.failed(output);

        } catch (Exception e) {
            return TestResult.error(e.getMessage());
        }
    }
}
```

## 3.4 结果解析与边界

| 判定 | 依据 |
|------|------|
| 通过 | BUILD SUCCESS 且退出码 0 |
| 失败 | BUILD FAILURE |
| 超时 | waitFor 返回 false |
| 错误 | 启动或 IO 异常 |

边界与坑：仅凭文本关键字可能被日志中的 "BUILD SUCCESS" 干扰，应同时校验退出码。

生产建议：固定基础镜像 tag，预拉取；对输出做结构化解析提取失败测试列表。

## 3.5 测试框架适配

| 生态 | 命令 | 成功标识 |
|------|------|----------|
| Maven | mvn test | BUILD SUCCESS |
| Gradle | gradle test | BUILD SUCCESSFUL |
| npm | npm test | 退出码 0 |
| pytest | pytest | 退出码 0 |

结论先行：把命令与判定封装为策略，避免把构建工具细节散落在编排层。

# 4. 修复循环

## 4.1 循环结构

结论先行：循环以最多 N 次尝试为界，每次基于上一轮失败信息重新生成补丁。

## 4.2 提示词与上下文

### 4.2.1 系统提示

约束流程、风格与禁区，要求先检索后修改。

### 4.2.2 失败回灌

将截断后的失败输出拼入下一轮任务描述，聚焦错误信息。

## 4.3 完整实现

```java
@Service
@RequiredArgsConstructor
public class CodingAgent {

    private final CodeTools codeTools;
    private final SandboxTestRunner testRunner;
    private final ChatClient chatClient;

    public CodingResult solve(String task, Path repoDir) {
        for (int attempt = 0; attempt < 5; attempt++) {
            // ① 生成修改
            String result = chatClient.prompt()
                    .system("""
                            你是编码 Agent。
                            1. 先搜索相关代码；
                            2. 修改代码解决问题；
                            3. 保持代码风格一致。
                            """)
                    .user(task)
                    .tools(codeTools)
                    .call().content();

            // ② 执行测试
            TestResult test = testRunner.runTests(repoDir);
            if (test.passed()) {
                return CodingResult.success(result, test.output());
            }
            // ③ 基于失败修复
            task = """
                    原始任务：%s

                    上次测试失败：
                    %s

                    请修复。
                    """.formatted(task, truncate(test.output(), 3000));

            log.info("第 {} 次尝试失败，继续修复", attempt + 1);
        }
        return CodingResult.failed("多次尝试仍未通过测试");
    }
}
```

## 4.4 收敛与终止

| 条件 | 处理 |
|------|------|
| 通过 | 返回成功与测试输出 |
| 达上限 | 返回失败 |
| 错误重复 | 提前熔断 |

边界与坑：任务描述每轮拼接会不断膨胀，应保留原始任务并用独立字段承载失败信息。

生产建议：记录每轮补丁与结果，便于事后分析失败模式。

## 4.5 失败归因

| 类型 | 判据 | 修复重点 |
|------|------|----------|
| 编译错误 | 无测试执行 | 语法与符号 |
| 断言失败 | 有测试失败列表 | 逻辑与边界 |
| 环境错误 | 依赖下载失败 | 沙箱与依赖 |

结论先行：先归因再修复，比盲目重试更省 token。

# 5. 变更审计

## 5.1 审计模型

| 字段 | 说明 |
|------|------|
| taskId | 任务 ID |
| filePath | 文件路径 |
| beforeHash | 变更前哈希 |
| afterHash | 变更后哈希 |
| diff | 差异 |
| user | 操作者 |
| time | 时间 |

## 5.2 审计实现

```java
public void auditChange(String taskId, String filePath,
                        String oldContent, String newContent) {
    auditLog.record(new CodeChange(
            taskId,
            filePath,
            sha256(oldContent),
            sha256(newContent),
            diff(oldContent, newContent),
            currentUser(),
            Instant.now()));
}
```

## 5.3 回滚

结论先行：回滚基于 Git 分支与审计记录，可精确还原到变更前状态。

```
审计记录 → 定位变更文件 → 应用反向补丁 / 切换分支 → 校验
```

生产建议：审计日志只追加不修改，并设置留存周期满足合规要求。

## 5.4 审计查询

```java
public List<CodeChange> changesOf(String taskId) {
    return auditLog.query(
            FilterExpressionBuilder.eq("taskId", taskId),
            Sort.by("time").ascending());
}
```

结论先行：按任务维度可聚合出完整变更集，支撑评审与回滚。

# 6. 难点与解决

## 6.1 难点清单

| 难点 | 解决 |
|------|------|
| 上下文超限 | 代码检索而非全量 |
| 多文件修改 | 结构化 patch + 逐文件工具 |
| 测试环境 | 容器沙箱 |
| 沙箱安全 | 禁网 + 限资源 |
| 修复循环不收敛 | 最大尝试次数 |
| 破坏代码 | Git 隔离 + 人工确认 |

## 6.2 上下文超限

### 6.2.1 检索替代全量

只把与任务相关的片段喂给模型，而非整仓库。

### 6.2.2 截断与摘要

对长输出与长文件做截断，必要时先摘要再回灌。

## 6.3 多文件修改

结论先行：以统一 diff 表达多文件变更，逐文件应用并保证原子性。

## 6.4 沙箱与安全

容器隔离、禁网、限资源；所有路径经校验；生产凭证不进沙箱。

## 6.5 收敛与成本

| 手段 | 效果 |
|------|------|
| 最大尝试次数 | 防止无限循环 |
| 无进展检测 | 提前熔断 |
| 模型路由 | 降低成本 |

## 6.6 反例

- 反例一：把整仓库塞进上下文，导致超限与高成本。
- 反例二：不做路径校验直接写文件，存在越界风险。
- 反例三：沙箱开启网络，测试代码可外联泄密。
- 反例四：无最大尝试次数，失败任务无限重试。

结论先行：上述反例均应在代码评审与安全用例中显式拦截。

# 7. 部署与运行

## 7.1 部署形态

结论先行：Agent 服务与沙箱可同机部署，也可将沙箱独立到专用节点。

## 7.2 配置

```yaml
coding-agent:
  repo-root: /data/repos/backend
  max-attempts: 5
  sandbox:
    image: maven:3.9-eclipse-temurin-21
    network: none
    memory: 2g
    cpus: 2
    timeout: PT10M
```

## 7.3 监控

| 指标 | 说明 |
|------|------|
| 任务成功率 | 核心健康度 |
| 平均尝试次数 | 修复效率 |
| 沙箱超时率 | 资源是否充足 |
| 每任务成本 | 预算控制 |

生产建议：对连续失败与超时设置告警，避免资源被长任务占满。

## 7.4 容量与扩容

| 维度 | 策略 |
|------|------|
| 并发任务 | 队列削峰 |
| 沙箱节点 | 按 CPU/内存扩容 |
| 向量库 | 分片或托管服务 |
| 索引 | 定时增量 |

结论先行：沙箱是资源瓶颈，优先为其做独立扩容与配额隔离。

# 8. 小结

## 8.1 核心结论

- 代码索引按函数/类切分，保留路径与行号。
- 工具：搜索、读、写，均做路径校验。
- 沙箱执行测试：禁网 + 限资源 + 超时。
- Reflexion 循环基于测试错误修复。
- 所有变更需审计，人工确认后应用。

## 8.2 后续演进

- 索引增量更新与重排。
- 沙箱隔离升级与镜像预热。
- 失败归因驱动的自适应重试。
