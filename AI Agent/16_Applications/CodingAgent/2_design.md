# 1. 整体架构

## 1.1 架构总览

### 1.1.1 组件划分

结论先行：编码 Agent 由检索、理解、修改、执行、修复五个能力域组成，向上暴露统一任务接口，向下依赖代码库、Git、沙箱与测试框架。

```
┌────────────────────────────────────────┐
│  编码 Agent                             │
│  ├── 代码检索（混合检索）                │
│  ├── 代码理解（LLM）                     │
│  ├── 代码修改（工具）                    │
│  ├── 测试执行（沙箱）                    │
│  └── 修复循环（Reflexion）               │
└──────────┬─────────────────────────────┘
           │
   ┌───────┼───────┬──────────┐
   ▼       ▼       ▼          ▼
┌──────┐┌──────┐┌────────┐┌────────┐
│代码库 ││ Git  ││ 沙箱    ││ 测试   │
│索引   ││ 只读  ││ 执行    ││ 框架   │
└──────┘└──────┘└────────┘└────────┘
```

### 1.1.2 数据流

```
任务文本 → 检索器 → 上下文片段 → LLM → 补丁
   → 写工具 → 仓库副本 → 沙箱测试 → 结果
   → 失败回灌 LLM / 成功生成说明
```

边界与坑：上下文片段与补丁在循环中反复进出，必须做体积控制，否则很快超出模型窗口。

## 1.2 分层设计

| 层 | 职责 | 关键类 |
|----|------|--------|
| 接口层 | 任务提交、查询、应用 | CodingController |
| 编排层 | 循环、状态、工具调度 | CodingAgent |
| 能力层 | 检索、修改、执行 | Retriever / Tools / Runner |
| 基础设施 | 向量库、Git、沙箱 | VectorStore / Git / Docker |

结论先行：编排层是唯一持有循环状态的地方，其余层保持无状态以便水平扩展。

## 1.3 部署形态

### 1.3.1 单体部署

Agent 服务与沙箱运行在同一主机，通过 Docker 启动隔离容器。适合初期与小规模。

### 1.3.2 微服务部署

检索、执行、编排拆为独立服务，沙箱执行单独部署在专用节点，便于资源隔离与弹性伸缩。

| 形态 | 优点 | 缺点 |
|------|------|------|
| 单体 | 简单、时延低 | 资源耦合 |
| 微服务 | 隔离好、可扩展 | 运维复杂 |

生产建议：从单体起步，当沙箱资源竞争明显时再拆分。

## 1.4 组件交互时序

```
用户   接口层   编排层   检索器   工具    沙箱
 │       │        │        │       │       │
 │─提交─▶│        │        │       │       │
 │       │─创建──▶│        │       │       │
 │       │        │─检索──▶│       │       │
 │       │        │◀─片段──│       │       │
 │       │        │─写补丁─────────▶│       │
 │       │        │─执行────────────────────▶│
 │       │        │◀─测试结果────────────────│
 │       │◀─状态──│        │       │       │
 │◀─结果─│        │        │       │       │
```

结论先行：编排层是唯一的协调者，检索器、工具、沙箱均不感知循环状态。

# 2. 代码检索

## 2.1 切分策略

### 2.1.1 按结构切分

结论先行：按函数/类为单位切分，比按固定长度切分更契合代码语义。

```
代码库 → 按函数/类切分 → 向量化 + 关键词索引

查询："处理认证失败"
  → 混合检索：语义 + 关键词（getUserById 等标识符）
  → 返回相关代码片段（含文件路径、行号）
```

| 特点 | 说明 |
|------|------|
| 按结构切分 | 函数/类为单位 |
| 保留元数据 | 文件路径、行号、语言 |
| 混合检索 | 代码标识符适合关键词 |

### 2.1.2 元数据保留

每个片段携带文件路径、起始行、语言、符号名，便于回填上下文与定位修改点。

## 2.2 混合检索

### 2.2.1 语义检索

将片段向量化，按余弦相似度召回，擅长"意图相近但用词不同"的查询。

### 2.2.2 关键词检索

基于倒排索引（如 BM25），擅长精确匹配标识符、类名、方法名。

### 2.2.3 融合排序

结论先行：混合检索的关键在融合策略，常用加权求和或倒数排名融合（RRF）。

| 融合方式 | 说明 | 适用 |
|----------|------|------|
| 加权求和 | 归一化后按权重相加 | 分数可比 |
| RRF | 按排名倒数求和 | 分数不可比 |
| 重排 | 交叉编码器精排 | 追求精度 |

## 2.3 检索实现

```java
public List<CodeSnippet> hybridSearch(String query, int topK) {
    List<Document> semantic = vectorStore.similaritySearch(
            SearchRequest.builder().query(query).topK(topK).build());
    List<Document> lexical = keywordIndex.search(query, topK);
    return fuse(semantic, lexical, topK);
}

private List<CodeSnippet> fuse(List<Document> a, List<Document> b, int k) {
    Map<String, Double> score = new HashMap<>();
    for (int i = 0; i < a.size(); i++) {
        score.merge(id(a.get(i)), 1.0 / (RRF_K + i + 1), Double::sum);
    }
    for (int i = 0; i < b.size(); i++) {
        score.merge(id(b.get(i)), 1.0 / (RRF_K + i + 1), Double::sum);
    }
    return score.entrySet().stream()
            .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
            .limit(k)
            .map(this::toSnippet)
            .toList();
}
```

## 2.4 边界与调优

- 片段过大：超出上下文，需限制单片段行数。
- 片段过小：丢失上下文，需带回相邻签名与注释。
- 权重失衡：语义与关键词权重需按仓库调参。
- 增量更新：文件变更后需增量重建索引，避免陈旧。

生产建议：为检索质量建立离线指标（召回率、MRR），随版本回归。

# 3. Agent 循环

## 3.1 循环概览

```
① 理解任务
② 检索相关代码
③ 生成修改方案
④ 应用修改（工具）
⑤ 执行测试（沙箱）
⑥ 测试失败 → 读取错误 → 修复 → 回到 ⑤
⑦ 测试通过 → 生成变更说明
```

结论先行：循环以"测试通过"为唯一成功条件，其余均为中间态。

## 3.2 状态机

### 3.2.1 状态定义

| 状态 | 含义 |
|------|------|
| PLANNING | 理解与检索 |
| PATCHING | 生成并应用补丁 |
| TESTING | 沙箱执行 |
| FIXING | 基于失败修复 |
| DONE | 通过并生成说明 |
| FAILED | 达上限仍失败 |

### 3.2.2 转移条件

```
PLANNING → PATCHING : 上下文就绪
PATCHING → TESTING  : 补丁已应用
TESTING  → DONE     : 测试通过
TESTING  → FIXING   : 测试失败且未达上限
TESTING  → FAILED   : 达最大尝试次数
FIXING   → PATCHING : 生成新补丁
```

## 3.3 循环实现

```java
public String codeTask(String task) {
    for (int attempt = 0; attempt < MAX_ATTEMPTS; attempt++) {
        List<CodeChunk> context = codeRetriever.retrieve(task);
        String patch = coder.generatePatch(task, context);

        applyPatch(patch);

        TestResult result = testRunner.run();
        if (result.passed()) {
            return generateChangeSummary(patch);
        }
        task = task + "\n\n上次测试失败：\n" + result.failureOutput();
    }
    return "多次尝试仍未通过测试";
}
```

## 3.4 收敛与终止

| 终止条件 | 处理 |
|----------|------|
| 测试通过 | 成功返回 |
| 达最大尝试 | 返回失败并保留最后补丁 |
| 沙箱超时 | 中止并回收 |
| 无进展 | 连续相同错误即熔断 |

边界与坑：若不检测"无进展"，Agent 可能反复生成相同补丁而空耗配额。

## 3.5 提示词设计

结论先行：系统提示词约束流程与风格，用户提示词承载任务与失败反馈。

```java
String system = """
        你是编码 Agent，必须：
        1. 先搜索相关代码，再修改；
        2. 只改必要文件，保持代码风格一致；
        3. 不得修改部署与文档目录；
        4. 输出统一 diff。
        """;
```

边界与坑：提示词无法替代硬约束，越界拦截必须落在工具与沙箱层。

# 4. 工具设计

## 4.1 工具清单

| 工具 | 说明 |
|------|------|
| `searchCode` | 检索代码 |
| `readFile` | 读取文件（限目录） |
| `writeFile` | 写入文件（限目录） |
| `applyPatch` | 应用补丁 |
| `runTests` | 执行测试（沙箱） |
| `gitDiff` | 查看变更 |

## 4.2 工具契约

### 4.2.1 入参校验

结论先行：所有涉及路径的工具必须先做规范化与越界校验，再做任何 IO。

### 4.2.2 输出约定

| 工具 | 输出 | 失败语义 |
|------|------|----------|
| searchCode | 片段列表 | 空列表 |
| readFile | 文件内容 | 抛异常并回灌 |
| writeFile | 成功提示 | 抛异常并回灌 |
| applyPatch | 变更文件列表 | 冲突提示 |
| runTests | 测试输出与状态 | 超时/错误状态 |

## 4.3 工具实现

```java
@Tool("读取代码文件")
public String readFile(@ToolParam(description = "相对路径") String path) {
    Path target = repoRoot.resolve(path).normalize();
    if (!target.startsWith(repoRoot)) {
        throw new SecurityException("路径越界");
    }
    return Files.readString(target);
}
```

```java
@Tool("应用补丁")
public String applyPatch(@ToolParam(description = "统一 diff") String patch) {
    List<String> changed = patchService.apply(repoRoot, patch);
    return "已应用：" + String.join(", ", changed);
}
```

## 4.4 工具安全

| 风险 | 措施 |
|------|------|
| 路径穿越 | normalize + startsWith |
| 越权写 | 白名单目录 |
| 注入 | 参数化、禁止 shell 拼接 |
| 资源滥用 | 限调用次数与大小 |

生产建议：工具集合按用户角色下发，只读用户不下发写类工具。

# 5. 沙箱设计

## 5.1 隔离模型

结论先行：沙箱以容器为单位，禁网、限资源、只挂载仓库副本，是执行类工具的唯一出口。

## 5.2 资源限制

| 资源 | 限制 |
|------|------|
| 网络 | --network none |
| 内存 | --memory 2g |
| CPU | --cpus 2 |
| 进程 | --pids-limit 128 |
| 时长 | 10 分钟超时 |

## 5.3 执行实现

```java
public TestResult runTests() {
    ProcessBuilder pb = new ProcessBuilder(
            "docker", "run", "--rm",
            "--network", "none",              // 禁网
            "--memory", "2g",
            "--cpus", "2",
            "--pids-limit", "128",
            "-v", repoDir + ":/workspace:rw",
            "-w", "/workspace",
            "maven:3.9-eclipse-temurin-21",
            "mvn", "test");

    // 超时 + 输出限制
    return executeWithTimeout(pb, Duration.ofMinutes(10));
}
```

## 5.4 结果解析

```
mvn 输出 → 判定 BUILD SUCCESS / FAILURE
   → 提取失败测试与堆栈 → 截断至上限 → 回灌
```

边界与坑：仅靠关键字判定可能误判，应结合退出码；输出需截断避免撑爆上下文。

生产建议：镜像预拉取并固定版本，避免运行时拉取带来的时延与不确定性。

## 5.5 沙箱编排

结论先行：每个任务分配独立工作目录与容器，任务结束即销毁容器并清理副本。

```
准备仓库副本 → 启动容器 → 执行命令 → 采集输出
   → 销毁容器 → 清理副本 → 归档日志
```

| 阶段 | 失败处理 |
|------|----------|
| 启动 | 重试一次 |
| 执行 | 超时强杀 |
| 清理 | 定时兜底回收 |

# 6. 安全设计

## 6.1 威胁模型

| 威胁 | 来源 | 后果 |
|------|------|------|
| 路径穿越 | 恶意路径参数 | 读写越界文件 |
| 网络外联 | 测试代码 | 数据外泄 |
| 资源耗尽 | 死循环/炸弹 | 宿主机不可用 |
| 破坏代码 | 误改 | 仓库受损 |
| 访问生产 | 凭证泄露 | 数据泄露 |

## 6.2 防护矩阵

| 风险 | 防护 |
|------|------|
| 路径穿越 | normalize + startsWith |
| 网络访问 | 沙箱禁网 |
| 资源耗尽 | CPU/内存/时间限制 |
| 破坏代码 | Git 隔离 + 人工确认 |
| 访问生产数据 | 沙箱无生产凭证 |

### 6.2.1 路径穿越

结论先行：`resolve` 后 `normalize`，再校验 `startsWith(repoRoot)`，三步缺一不可。

### 6.2.2 网络访问

禁用容器网络，并限制基础镜像不自带外联工具。

### 6.2.3 资源耗尽

同时限制 CPU、内存、进程数与墙钟时长，任一超限即强制回收。

## 6.3 审计与回滚

```java
public void audit(String taskId, String file, String before, String after) {
    auditLog.record(new CodeChange(taskId, file,
            sha256(before), sha256(after), diff(before, after),
            currentUser(), Instant.now()));
}
```

| 能力 | 实现 |
|------|------|
| 留痕 | 变更日志 |
| 回滚 | Git 分支还原 |
| 追溯 | 任务与用户关联 |

## 6.4 安全测试用例

| 用例 | 期望 |
|------|------|
| 路径 `../../etc/passwd` | 拒绝 |
| 测试代码发起外联 | 失败 |
| 死循环占用 CPU | 超时回收 |
| 内存炸弹 | OOM 终止 |
| 访问生产库凭证 | 无凭证可用 |

结论先行：安全用例应纳入 CI，随每次变更回归。

# 7. 接口设计

## 7.1 REST 接口

### 7.1.1 提交任务

```java
POST /api/coding/task
{
  "task": "修复用户登录失败的问题",
  "repoId": "backend"
}
→ {
  "status": "COMPLETED",
  "patch": "...",
  "summary": "...",
  "testResult": "PASSED"
}
```

### 7.1.2 查询与 diff

```
GET /api/coding/task/{id}/diff   // 查看变更
```

### 7.1.3 应用与回滚

```
POST /api/coding/task/{id}/apply // 人工确认后应用
POST /api/coding/task/{id}/revert // 回滚
```

## 7.2 数据模型

| 字段 | 类型 | 说明 |
|------|------|------|
| id | string | 任务 ID |
| status | enum | 状态 |
| patch | string | 补丁 |
| summary | string | 变更说明 |
| testResult | enum | 测试结果 |
| createdAt | timestamp | 创建时间 |

## 7.3 异步与回调

结论先行：长任务应异步化，提交返回任务 ID，通过轮询或回调获取结果。

边界与坑：同步等待长任务会耗尽连接，必须异步并设置超时。

生产建议：提供幂等键，避免重复提交造成重复执行。

# 8. 关键技术点

## 8.1 上下文控制

| 问题 | 方案 |
|------|------|
| 上下文超限 | 代码检索而非全量 |

## 8.2 多文件修改

### 8.2.1 结构化补丁

用统一 diff 表达多文件变更，便于审阅与应用。

### 8.2.2 逐文件应用

结论先行：逐文件应用并校验，单文件失败即回滚整个补丁，保证原子性。

## 8.3 修复循环

| 问题 | 方案 |
|------|------|
| 测试失败修复 | Reflexion 循环 |
| 沙箱安全 | 容器隔离 + 禁网 |
| 可回滚 | Git 分支 + patch |

## 8.4 关键点对照

| 问题 | 方案 |
|------|------|
| 上下文超限 | 代码检索而非全量 |
| 多文件修改 | 结构化 patch |
| 测试失败修复 | Reflexion 循环 |
| 沙箱安全 | 容器隔离 + 禁网 |
| 可回滚 | Git 分支 + patch |

# 9. 小结

## 9.1 核心结论

- 架构：检索 + 理解 + 修改 + 执行 + 修复循环。
- 代码按函数/类切分，混合检索。
- 沙箱执行测试，禁网 + 限资源。
- 安全：路径校验、Git 隔离、人工确认。
- 用 Reflexion 循环基于测试错误修复。

## 9.2 后续演进

- 检索引入重排与增量索引。
- 沙箱隔离升级到用户态内核。
- 循环加入失败归因与自适应重试。
