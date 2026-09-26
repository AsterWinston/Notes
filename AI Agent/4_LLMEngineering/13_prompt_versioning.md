# 1. 为什么 Prompt 要版本化

## 1.1 问题背景

Prompt（提示词）是 LLM（大语言模型）应用的"行为定义"，它决定了模型如何理解任务、如何组织输出、遵守哪些约束。虽然它看起来只是一段字符串，但它和传统代码一样会持续演化：

```
Prompt 是"代码"，改动会影响效果：
- 改好了 A 可能改坏 B
- 需要灰度验证
- 出问题要能快速回滚
```

Prompt 变更引发的故障往往比代码变更更隐蔽：编译不会报错，单元测试不会失败，类型系统也无法约束，只有上线后通过效果指标（准确率、满意度、拒答率）才能暴露。

更麻烦的是，Prompt 的效果高度依赖模型版本、温度（temperature）等采样参数以及上下文长度。同一个 Prompt 换一个模型可能从"表现优秀"变成"答非所问"，因此版本化不仅要管理 Prompt 文本，还要记录它当时所搭配的模型与参数。

## 1.2 核心原则

### 1.2.1 与代码同源

> **Prompt 必须与代码一起版本化，并支持独立灰度与回滚。**

"与代码一起版本化"意味着：每次变更都可追溯（谁改的、何时改的、为什么改），并且与一次代码提交或一次审批一一对应。若 Prompt 以字符串常量散落在业务代码中，就无法审计、无法回滚，也无法复现历史行为。

### 1.2.2 独立灰度与回滚

Prompt 的发布节奏通常快于代码。若每次改 Prompt 都必须走完整发版流程，就失去了快速试错的能力。因此 Prompt 需要独立于代码部署进行灰度（按比例放量）与回滚（切回历史版本），且这两件事不应触发应用重启。

### 1.2.3 Git 的边界

Git（分布式版本控制工具）能解决"历史可追溯"，但无法解决"运行时按用户切换版本"。生产环境需要在不重启、不发版的前提下调整 Prompt，这需要配置中心或数据库参与。Git 负责"存"，配置中心/数据库负责"取"。

| 能力 | 仅用 Git | Git + 配置中心/数据库 |
|------|----------|----------------------|
| 历史追溯 | 支持 | 支持 |
| 运行时切换 | 不支持 | 支持 |
| 灰度放量 | 不支持 | 支持 |
| 快速回滚 | 需发版 | 秒级切换 |
| 非研发人员操作 | 困难 | 可视化后台 |

# 2. 版本化方式

## 2.1 四种方案概览

| 方式 | 说明 | 适用 |
|------|------|------|
| 资源文件目录 | `prompts/v1/`、`prompts/v2/` | 简单 |
| 配置中心 | Nacos/Apollo 管理 | 热更新 |
| 数据库 | Prompt 表 + 版本字段 | 可视化 |
| Prompt 平台 | Langfuse 等 | 团队协作 |

四种方案并非互斥，很多团队采用"文件做基线 + 配置中心做灰度 + 平台做运营"的组合。

## 2.2 方案对比与选型

### 2.2.1 选型维度

| 维度 | 资源文件 | 配置中心 | 数据库 | Prompt 平台 |
|------|----------|----------|--------|-------------|
| 上手成本 | 低 | 中 | 中 | 中高 |
| 热更新 | 否（需重启或刷新） | 是 | 是 | 是 |
| 版本回滚 | 靠 Git | 靠历史配置 | 靠版本字段 | 内置 |
| 可视化 | 无 | 弱 | 需自建 | 强 |
| 审批流 | 靠代码评审 | 弱 | 可自建 | 内置 |
| 适合阶段 | 起步/单团队 | 中小团队 | 中大型 | 多团队协作 |

### 2.2.2 组合使用

一种常见落地组合：

```
基线（Git 文件） → 配置中心（灰度开关） → 数据库/平台（运营态）
        ▲                                          │
        └──────────── 回滚时统一回落到基线 ─────────┘
```

- 文件方案保证"永远有一个可运行、可复现的默认版本"。
- 配置中心负责放量比例与开关。
- 数据库/平台承载高频变更与可视化操作。

### 2.2.3 反例：不推荐的做法

```java
// 反例：Prompt 硬编码在业务代码中，且包含分支
public String ask(String q) {
    if (q.contains("总结")) {
        return client.prompt().system("你是总结助手……").user(q).call().content();
    }
    return client.prompt().system("你是问答助手……").user(q).call().content();
}
```

上述写法的问题：无法灰度、无法审计、无法回滚、逻辑与提示耦合、修改需要重新编译发版。

# 3. 资源文件方案

## 3.1 目录结构

```
src/main/resources/prompts/
├── v1/
│   ├── qa-system.txt
│   └── summary.txt
├── v2/
│   ├── qa-system.txt
│   └── summary.txt
└── manifest.json
```

```json
{
  "active": "v2",
  "rollout": {"v2": 30, "v1": 70}
}
```

按版本分目录的好处：版本边界清晰，`git diff` 一目了然，打包后仍可作为 classpath（类路径）资源读取。

## 3.2 清单与加载实现

### 3.2.1 manifest 的作用

`manifest.json`（清单文件）声明当前激活版本与放量比例。它让"版本切换"变成改一个数字，而不是改代码。生产环境可用配置中心覆盖该文件内容，实现文件与配置的融合。

### 3.2.2 PromptRegistry 实现

```java
@Component
public class PromptRegistry {

    private final Map<String, PromptTemplate> templates = new ConcurrentHashMap<>();

    public PromptRegistry(@Value("classpath:/prompts/${app.prompt.version}/*.txt")
                          Resource[] resources) throws IOException {
        for (Resource r : resources) {
            String name = Objects.requireNonNull(r.getFilename()).replace(".txt", "");
            templates.put(name, PromptTemplate.from(
                    new String(r.getInputStream().readAllBytes(), UTF_8)));
        }
    }

    public PromptTemplate get(String name) {
        PromptTemplate t = templates.get(name);
        if (t == null) throw new IllegalArgumentException("未知模板: " + name);
        return t;
    }
}
```

JDK 17 起可用 `readAllBytes()` 直接读取资源；如需支持多版本共存，可把 `Map<String, PromptTemplate>` 改为 `Map<String, Map<String, PromptTemplate>>`，外层键为版本。

### 3.2.3 边界与反例

| 场景 | 是否适合文件方案 | 说明 |
|------|------------------|------|
| 版本变更低频 | 适合 | 发版时切换 |
| 需要分钟级灰度 | 不适合 | 需配置中心 |
| 运营人员自助修改 | 不适合 | 需可视化后台 |
| 需要完整审计 | 部分适合 | 靠 Git 提交记录 |
| 超大 Prompt（>100KB） | 不适合 | 建议外部存储 |

# 4. 配置中心方案

## 4.1 Nacos 配置

Nacos（阿里巴巴开源的服务发现与配置管理平台）支持配置热更新。把版本与放量比例放进配置：

```yaml
# Nacos 配置
app:
  prompt:
    version: v2
    rollout-percentage: 30
```

配置中心的优势是"改配置即生效"，无需重新打包；劣势是版本内容本身通常不适合直接存放（可只存版本号，内容仍从文件/数据库读取）。

## 4.2 动态解析与灰度

### 4.2.1 哈希灰度实现

```java
@Component
@RefreshScope
public class DynamicPromptService {

    @Value("${app.prompt.version:v1}")
    private String version;

    @Value("${app.prompt.rollout-percentage:100}")
    private int rolloutPercentage;

    public String resolveVersion(UserContext user) {
        if (rolloutPercentage >= 100) return version;

        // 灰度：按用户哈希
        int hash = Math.abs((user.userId() + version).hashCode());
        return (hash % 100) < rolloutPercentage ? version : "v1";
    }
}
```

关键点：哈希输入必须包含 `userId`（保证同一用户始终命中同一版本，避免体验抖动），同时包含 `version`（不同版本独立分桶，便于对比）。

### 4.2.2 动态刷新注意事项

- `@RefreshScope` 会让 Bean 在配置变更后重建，注意重建期间的开销与副作用。
- 灰度比例从 30 改到 100 时，应保证"已命中新版的用户不会回退"，即放量只增不减。
- 配置中心不可用时必须有本地兜底，否则应用无法启动。

### 4.2.3 灰度分桶的常见坑

```
错误：hash(userId) % 100 < 30   // 换版本后同一批人还是同一批人
正确：hash(userId + version)     // 不同版本独立分桶
```

若只用 `userId` 哈希，切换版本时命中的仍是同一批用户，无法观察"新老版本在同一人群上的差异"。

# 5. 数据库方案

## 5.1 表结构设计

```sql
CREATE TABLE prompt_version (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(64)  NOT NULL,      -- 模板名
    version     VARCHAR(16)  NOT NULL,      -- 版本
    content     TEXT         NOT NULL,      -- 模板内容
    status      VARCHAR(16)  NOT NULL,      -- DRAFT/ACTIVE/DEPRECATED
    rollout     INT          NOT NULL DEFAULT 100,
    created_by  VARCHAR(64),
    created_at  TIMESTAMP    NOT NULL DEFAULT now(),
    UNIQUE (name, version)
);
```

状态机设计：

| 状态 | 含义 | 可否被解析命中 |
|------|------|----------------|
| DRAFT | 草稿，仅测试环境 | 否 |
| ACTIVE | 正式版本 | 是 |
| DEPRECATED | 已下线 | 否（仅回滚用） |

同一 `name` 下最多一个 `ACTIVE`，可用唯一索引或应用层保证。

## 5.2 版本解析与激活

### 5.2.1 解析逻辑

```java
@Service
public class PromptVersionService {

    public String resolve(String name, UserContext user) {
        PromptVersion active = repository.findActive(name);
        PromptVersion candidate = repository.findCandidate(name);

        if (candidate != null && inRollout(user, candidate.rollout())) {
            return candidate.content();
        }
        return active.content();
    }

    @Transactional
    public void activate(String name, String version, String operator) {
        repository.deprecateCurrent(name);
        repository.activate(name, version);
        auditLog.record("PROMPT_ACTIVATE", name, version, operator);
    }
}
```

`inRollout` 与配置中心的哈希逻辑保持一致，确保"候选版本"只在灰度人群生效。

### 5.2.2 缓存与一致性

数据库方案必须加缓存，否则每次请求都查库。推荐：

```java
// 伪代码：本地缓存 + 主动失效
@Cacheable(value = "prompt", key = "#name + ':' + #version")
public String content(String name, String version) {
    return repository.find(name, version).content();
}

// 激活后主动清缓存，而非依赖 TTL
public void activate(...) {
    // ...更新数据库...
    cacheManager.getCache("prompt").clear();
}
```

> 不要用固定 TTL 等缓存自然过期，否则激活后最长要等一个 TTL 才生效。

### 5.2.3 审计不可省略

每次激活、回滚、灰度调整都应写入审计日志，字段至少包含：操作人、时间、模板名、旧版本、新版本、原因。没有审计，出问题时无法定位"是谁、什么时候改坏的"。

# 6. 版本化配置化

## 6.1 应用代码不写死 Prompt

```java
// 应用代码不写死 Prompt
@Service
public class QaService {

    private final PromptVersionService promptService;

    public String ask(String question, UserContext user) {
        String systemPrompt = promptService.resolve("qa-system", user);

        return chatClient.prompt()
                .system(systemPrompt)
                .user(question)
                .call()
                .content();
    }
}
```

> **应用代码零改动**，Prompt 变更只改配置/数据库。

## 6.2 依赖注入与可测试性

把 Prompt 解析抽象为接口，便于单测替换：

```java
public interface PromptResolver {
    String resolve(String name, UserContext user);
}

@Service
public class QaService {

    private final PromptResolver resolver;
    private final ChatClient chatClient;

    public QaService(PromptResolver resolver, ChatClient chatClient) {
        this.resolver = resolver;
        this.chatClient = chatClient;
    }

    public String ask(String question, UserContext user) {
        String systemPrompt = resolver.resolve("qa-system", user);
        return chatClient.prompt().system(systemPrompt).user(question)
                .call().content();
    }
}
```

```java
@Test
void should_use_resolved_prompt() {
    PromptResolver resolver = (name, user) -> "你是测试助手";
    QaService service = new QaService(resolver, mockChatClient);
    assertNotNull(service.ask("你好", user));
}
```

# 7. 环境隔离

## 7.1 环境与版本映射

| 环境 | Prompt 版本 |
|------|-------------|
| dev | 最新草稿 |
| test | 候选版本 |
| prod | 灰度/正式版本 |

隔离原则：dev 可以指向任何草稿；test 只能指向通过评测的候选；prod 只能指向已审批的正式版本。任何"从 dev 直接推到 prod"的行为都应被流程阻断。

## 7.2 配置示例与隔离边界

```yaml
# application-dev.yaml
app:
  prompt:
    version: v3-draft

# application-prod.yaml
app:
  prompt:
    version: v2
    rollout-percentage: 30
```

| 隔离手段 | 说明 |
|----------|------|
| 配置分文件 | dev/test/prod 各自 `application-*.yaml` |
| 命名空间隔离 | Nacos 不同 namespace |
| 数据库隔离 | 不同 schema 或表前缀 |
| 密钥隔离 | 不同环境的模型 Key 不共用 |
| 权限隔离 | prod 变更需审批 |

> 反例：三个环境共用一个 Prompt 表，仅靠 `version` 区分。一旦误操作，测试流量会污染生产。

# 8. 灰度与回滚

## 8.1 灰度放量

```java
// 灰度放量
public void rampUp(String name, int from, int to) {
    // 5% → 20% → 50% → 100%
    repository.updateRollout(name, to);
    metrics.watch(name, to);           // 观察指标
}
```

放量节奏建议：

| 阶段 | 比例 | 观察时长 | 通过条件 |
|------|------|----------|----------|
| 内部验证 | 1% | 数小时 | 无异常、无报错 |
| 小流量 | 5% | 1 天 | 核心指标不降 |
| 放量 | 20%→50% | 1–2 天 | 指标持平或上升 |
| 全量 | 100% | 持续 | 稳定后进入正式 |

每一步的 `metrics.watch` 应绑定具体的告警阈值，而不是"人肉盯盘"。

## 8.2 快速回滚

### 8.2.1 回滚实现

```java
// 快速回滚
public void rollback(String name) {
    PromptVersion previous = repository.findPreviousActive(name);
    repository.activate(name, previous.version());
    log.warn("Prompt 已回滚: {} -> {}", name, previous.version());
}
```

### 8.2.2 回滚边界

- 回滚必须秒级生效：依赖缓存主动失效，而非 TTL。
- 回滚要幂等：重复调用不应产生副作用。
- 回滚后要复盘：记录触发回滚的指标与阈值，避免重复踩坑。
- 若新版本已改变输出格式且下游已适配，回滚可能引发下游解析失败——因此格式类变更应与 Prompt 变更解耦发布。

# 9. 变更影响评估

## 9.1 变更类型与评估方式

| 变更 | 评估方式 |
|------|----------|
| 改 Prompt | 跑评测对比 |
| 换模型 | 评测 + 灰度 |
| 调参数 | 评测 |
| 加约束 | 评测 + 人工抽查 |

| 变更类型 | 风险等级 | 建议门禁 |
|----------|----------|----------|
| 措辞微调 | 低 | 评测通过即可 |
| 输出格式变更 | 高 | 评测 + 契约测试 |
| 安全约束变更 | 高 | 评测 + 安全测试 |
| 换模型 | 高 | 评测 + 灰度 + 回滚预案 |

## 9.2 评测门禁实现

```java
@Test
void prompt_change_should_not_degrade() {
    EvalReport v1 = evaluateWith("v1");
    EvalReport v2 = evaluateWith("v2");

    assertTrue(v2.accuracy() >= v1.accuracy() - 0.02,
            "v2 准确率下降");
    assertTrue(v2.avgCost().compareTo(v1.avgCost()) <= 0,
            "v2 成本上升");
}
```

评测门禁应作为 CI（持续集成）的一环：不通过则不允许激活新版本。容差（如 `-0.02`）用于抵消模型输出的随机性，建议多次运行取均值。

# 10. 生产注意事项

## 10.1 风险清单

| 风险 | 建议 |
|------|------|
| Prompt 写死代码 | 外置 + 版本化 |
| 无灰度 | 按比例放量 |
| 无法回滚 | 保留历史版本 |
| 环境混用 | 严格隔离 |
| 变更无评测 | 评测门禁 |
| 变更无审计 | 记录操作人 |

## 10.2 落地检查清单

- 是否所有 Prompt 都已外置，代码中不再出现长提示词常量？
- 是否存在"一键回滚"入口，且验证过秒级生效？
- 灰度哈希是否包含版本因子，保证分桶独立？
- 缓存失效是否主动触发，而非依赖 TTL？
- 每次变更是否有评测报告与审计记录？
- 三个环境的配置、命名空间、密钥是否物理隔离？

## 10.3 与其他模块的衔接

| 关联模块 | 关系 |
|----------|------|
| `3_PromptEngineering` | 提供 Prompt 编写方法论 |
| `12_testing_llm_apps.md` | 提供评测与回归实现 |
| `14_ProductionEngineering` | 提供发布、灰度、监控基础设施 |

# 11. 小结

## 11.1 核心结论

- Prompt 是代码，必须版本化。
- 四种方案：资源文件、配置中心、数据库、Prompt 平台。
- 应用代码不写死 Prompt，通过服务解析。
- 支持灰度放量与快速回滚。
- 变更前必须评测，变更后必须审计。

## 11.2 一句话记忆

> Prompt 外置、版本可追溯、灰度可放量、回滚秒级、变更过评测、操作留审计。
