# 1. 为什么需要管理

## 1.1 Prompt 即代码

Prompt（提示词）不是随手写在字符串里的文本，而是决定模型行为的核心资产。它具备代码的全部特征：

```
Prompt 是代码：
- 会频繁变更
- 变更影响效果
- 需要灰度与回滚
- 需要版本追溯
```

把 Prompt 当作代码，意味着它必须进入完整的研发流程：评审、测试、发布、监控、回滚。任何"改完直接上线"的做法，都会在规模化后变成事故来源。

> 详见 `4_LLMEngineering/13_prompt_versioning.md`，本文聚焦管理实践。

## 1.2 变更的三类风险

### 1.2.1 效果回归

一处措辞调整往往同时影响多个场景。整体准确率上升，但某个垂类或某类边界输入下降，是 Prompt 变更中最常见、也最难察觉的回归。

### 1.2.2 无法追溯

线上出问题时，无法回答三个基本问题：当前生效的是哪个版本、是谁改的、改了什么。缺少审计时，只能靠猜测定位。

### 1.2.3 无法止损

没有旧版本留存，出问题就只能回滚整个应用，而不是把单个 Prompt 切回上一个版本。止损成本从"秒级"变成"小时级"。

## 1.3 与软件工程的类比

| 软件工程 | Prompt 管理 |
|----------|-------------|
| 源码 | Prompt 模板 |
| Git 版本号 | 版本目录 + manifest 清单 |
| 灰度发布 | 按比例分流 |
| 回滚 | 切回旧版本 |
| 变更审计 | 操作日志 |
| CI / 单元测试 | 评测集回归 |
| 配置中心 | 模板注册中心 |

**结论**：凡是软件工程对代码做过的事，Prompt 都需要做一遍。区别只在于 Prompt 的"编译器"是模型，回归是概率性的而非确定性的。

# 2. 管理的五个维度

## 2.1 五维度总览

```
① 版本：每次变更一个新版本
② 注册：统一注册与命名
③ 灰度：按比例放量
④ 回滚：快速切回旧版本
⑤ 审计：谁在何时改了什么
```

| 维度 | 目标 | 主要手段 | 缺失后果 |
|------|------|----------|----------|
| 版本 | 可追溯 | 版本目录 / 版本号 | 无法定位问题版本 |
| 注册 | 可发现 | 注册中心 / 命名规范 | 模板散落、重复 |
| 灰度 | 可控放量 | 分流算法 / 开关 | 全量即事故 |
| 回滚 | 可止损 | 保留历史版本 | 止损慢 |
| 审计 | 可问责 | 操作日志 / diff | 无法追责 |

## 2.2 维度之间的关系

五个维度不是并列的，而是一条闭环流水线：

```
变更 → 新版本 → 注册 → 评测 → 灰度 → 全量
                          │
                          └── 指标异常 → 回滚
        全程 ─────────────────────────→ 审计留痕
```

- 版本是基础：没有版本，灰度与回滚都无从谈起。
- 注册是入口：模板必须先被注册，才能被引用与检索。
- 灰度是保护：用小流量暴露问题，避免全量损失。
- 回滚是兜底：灰度发现异常时的最后一道防线。
- 审计是横切：贯穿所有环节，记录每一次动作。

## 2.3 成熟度演进

团队能力通常经历三级演进，级别越高，自动化程度越高：

| 等级 | 形态 | 版本 | 灰度 | 回滚 | 审计 |
|------|------|------|------|------|------|
| L1 | 文件 | 文件 | 无 | 重新部署 | Git 记录 |
| L2 | 版本目录 | 目录 | 手动 | 切版本 | 应用日志 |
| L3 | 管理平台 | 平台 | 自动 | 一键 | 平台审计 |

- L1：Prompt 以文件随代码提交，简单但发布依赖重新部署。
- L2：引入版本目录与 manifest，支持运行时加载、灰度与回滚。
- L3：独立平台提供可视化编辑、评测、审批、灰度与审计。

# 3. 模板注册中心

## 3.1 注册中心的职责

注册中心是 Prompt 管理的运行时入口，通常承担以下职责：

- 启动时批量加载模板并缓存到内存。
- 提供按名称获取模板与原始文本的接口。
- 校验模板命名与变量声明，启动即失败（fail-fast）。
- 暴露当前版本信息，供健康检查与监控使用。

## 3.2 完整 Java 实现

### 3.2.1 模板模型

```java
public record PromptTemplate(
        String name,
        String content,
        List<String> variables
) {

    public static PromptTemplate from(String name, String content) {
        return new PromptTemplate(name, content, extractVariables(content));
    }

    private static List<String> extractVariables(String content) {
        Matcher m = Pattern.compile("\\{\\{(\\w+)\\}\\}").matcher(content);
        List<String> vars = new ArrayList<>();
        while (m.find()) {
            vars.add(m.group(1));
        }
        return List.copyOf(vars);
    }
}
```

### 3.2.2 注册中心

```java
@Component
public class PromptRegistry {

    private final Map<String, PromptTemplate> templates = new ConcurrentHashMap<>();
    private final Map<String, String> rawContents = new ConcurrentHashMap<>();
    private final String version;

    public PromptRegistry(@Value("${app.prompt.version:v1}") String version,
                          @Value("classpath:/prompts/${app.prompt.version:v1}/*.txt")
                          Resource[] resources) throws IOException {
        this.version = version;
        for (Resource r : resources) {
            String name = Objects.requireNonNull(r.getFilename())
                    .replace(".txt", "");
            String content = new String(r.getInputStream().readAllBytes(), UTF_8);

            templates.put(name, PromptTemplate.from(name, content));
            rawContents.put(name, content);
        }
        log.info("已加载 {} 个 Prompt 模板，版本 {}", templates.size(), version);
    }

    public PromptTemplate get(String name) {
        PromptTemplate t = templates.get(name);
        if (t == null) {
            throw new IllegalArgumentException("未知模板: " + name);
        }
        return t;
    }

    public String raw(String name) {
        String content = rawContents.get(name);
        if (content == null) {
            throw new IllegalArgumentException("未知模板: " + name);
        }
        return content;
    }

    public String version() {
        return version;
    }
}
```

### 3.2.3 调用示例

业务代码只依赖注册中心的 `get` 与 `raw` 两个方法：`get` 返回带变量声明的模板对象，`raw` 返回原始文本。两者都不感知版本，版本切换对业务透明。

## 3.3 加载策略对比

| 策略 | 生效时机 | 优点 | 缺点 |
|------|----------|------|------|
| 启动时加载 | 重启后 | 简单、零延迟 | 变更需重启 |
| 定时轮询 | 轮询周期 | 无需重启 | 有延迟窗口 |
| 配置中心推送 | 实时 | 秒级生效 | 依赖基础设施 |
| 每次请求读取 | 实时 | 最简单 | 性能差、不可缓存 |

# 4. 命名规范

## 4.1 命名结构

```
<模块>_<用途>_<类型>

qa_system          QA 模块的系统提示
qa_user_template   QA 模块的用户模板
summary_system     摘要模块的系统提示
classify_fewshot   分类模块的少样本示例
```

| 命名要素 | 说明 |
|----------|------|
| 模块 | 归属功能 |
| 用途 | 具体作用 |
| 类型 | system/user/fewshot |

## 4.2 正例与反例

| 名称 | 评价 | 原因 |
|------|------|------|
| `qa_system` | 好 | 模块、用途、类型清晰 |
| `prompt1` | 差 | 无信息量 |
| `qaSystemFinalNew` | 差 | 用 final/new 表达版本 |
| `QA_SYSTEM` | 差 | 大小写不统一 |
| `qa_system_v2` | 差 | 版本应进目录，不进名称 |

**原则**：名称表达"是什么"，版本由目录或 manifest 表达"第几版"。

## 4.3 命名校验

用正则 `^[a-z][a-z0-9]*_[a-z][a-z0-9]*_[a-z][a-z0-9]*$` 校验模板名，启动时对每个模板名执行一次，不合规直接抛异常，避免脏名称进入生产。

# 5. 变量规范

## 5.1 变量语义化

```java
// 用明确的变量名，避免歧义
@UserMessage("""
        资料：{{context}}
        问题：{{question}}
        回答风格：{{style}}
        """)
```

| 规范 | 说明 |
|------|------|
| 变量名语义化 | `context` 而非 `c` |
| 用 record 承载 | 避免字段漂移 |
| 必填校验 | 缺失时报错 |

反例：`{{c}}`、`{{q}}`、`{{s}}` 这类单字母变量，模板一多就无法维护，且极易与正文冲突。

## 5.2 用 record 承载变量

```java
public record QaPromptVars(
        String context,
        String question,
        String style
) {
    public QaPromptVars {
        Objects.requireNonNull(context, "context 不能为空");
        Objects.requireNonNull(question, "question 不能为空");
        style = (style == null || style.isBlank()) ? "简洁" : style;
    }
}
```

用 record 承载变量的好处：

- 字段固定，编译器保证调用方不会漏传或错传。
- 可在紧凑构造器中集中做校验与默认值。
- 重构时可全局搜索字段名，避免"变量漂移"。

## 5.3 必填校验与默认值

| 变量类型 | 缺失处理 | 示例 |
|----------|----------|------|
| 必填 | 抛异常 | `question` |
| 可选有默认 | 用默认值 | `style` 默认"简洁" |
| 可选为空 | 保留占位符或删除整行 | `examples` |

渲染时逐个替换 `{{name}}`，遇到必填变量缺失立即抛出异常，宁可请求失败，也不要带着空占位符送给模型。

# 6. 版本目录结构

## 6.1 目录布局

```
resources/prompts/
├── v1/
│   ├── qa_system.txt
│   └── qa_user_template.txt
├── v2/
│   ├── qa_system.txt
│   └── qa_user_template.txt
└── manifest.json
```

目录布局的三个约定：

- 每个版本一个独立目录，版本之间互不覆盖。
- 文件名即模板名，不含版本号。
- manifest 只记录元数据，不记录正文。

## 6.2 manifest 清单

```json
{
  "active": "v2",
  "rollout": {"v2": 30, "v1": 70},
  "updatedAt": "2026-09-21T10:00:00Z",
  "updatedBy": "zhangsan"
}
```

| 字段 | 含义 |
|------|------|
| active | 默认生效版本 |
| rollout | 各版本流量比例 |
| updatedAt | 最近更新时间 |
| updatedBy | 最近修改人 |

## 6.3 版本元数据与生命周期

除 manifest 外，每个版本还应记录元数据：创建人、创建时间、审批人、评测分数与变更说明。版本状态机如下：

```
draft → evaluating → approved → gray → active → archived
```

- `draft`：草稿，未评测。
- `evaluating`：评测中。
- `approved`：审批通过，待发布。
- `gray`：灰度中。
- `active`：全量生效。
- `archived`：已归档，仅保留追溯。

状态只能单向流转，`archived` 版本不可再被激活，保证追溯链条完整。

# 7. 灰度解析

## 7.1 分流算法

```java
@Service
public class PromptResolver {

    @Value("${app.prompt.rollout-percentage:100}")
    private int rolloutPercentage;

    public String resolve(String name, UserContext user) {
        if (rolloutPercentage >= 100) {
            return registryV2.raw(name);
        }
        if (rolloutPercentage <= 0) {
            return registryV1.raw(name);
        }
        int bucket = stableBucket(user.userId(), name);
        return bucket < rolloutPercentage
                ? registryV2.raw(name)
                : registryV1.raw(name);
    }

    private int stableBucket(String userId, String name) {
        int hash = Math.abs((userId + name).hashCode());
        return hash % 100;
    }
}
```

要点：当放量为 0 或 100 时直接短路，避免哈希带来的边界误差。

## 7.2 稳定哈希与一致性

同一个用户在同一模板上必须始终落到同一版本，否则会出现"刷新一次换一个版本"的体验问题。稳定哈希的做法是对 `userId + ":" + name` 计算确定性哈希，再对 100 取模，得到 0 到 99 的稳定桶号，同一用户每次结果一致。

| 做法 | 一致性 | 说明 |
|------|--------|------|
| `Math.random()` | 差 | 每次请求都变 |
| 用户 ID 哈希 | 好 | 同用户稳定 |
| 用户 ID + 模板名 | 最好 | 不同模板独立分流 |
| 会话 ID 哈希 | 中 | 会话内稳定 |

## 7.3 放量策略

### 7.3.1 比例放量

按固定比例逐步提升，如 1% → 5% → 30% → 100%，每一档观察至少一个业务周期。

### 7.3.2 白名单放量

先把内部员工、测试账号加入白名单，验证无误后再比例放量。白名单优先级应高于比例分流。

### 7.3.3 自动放量

结合监控指标自动放量或自动回滚：指标正常则按计划提升比例，异常则立即回退到 0。

| 策略 | 风险 | 适用 |
|------|------|------|
| 比例放量 | 中 | 常规发布 |
| 白名单 | 低 | 高风险变更 |
| 自动放量 | 低 | 指标完善时 |

# 8. 变更审计

## 8.1 审计要素

一条合格的审计记录至少包含六个要素：

| 要素 | 说明 |
|------|------|
| 模板名 | 改的是哪个模板 |
| 旧内容哈希 | 改前版本指纹 |
| 新内容哈希 | 改后版本指纹 |
| 操作人 | 谁改的 |
| 时间 | 何时改的 |
| 变更原因 | 为什么改 |

## 8.2 AOP 实现

```java
@Aspect
@Component
public class PromptAuditAspect {

    private final AuditLog auditLog;
    private final PromptRepository repository;

    @Around("execution(* PromptService.update(..)) && args(name, content)")
    public Object auditUpdate(ProceedingJoinPoint pjp, String name, String content)
            throws Throwable {
        String oldContent = repository.findActive(name).content();

        Object result = pjp.proceed();

        auditLog.record(new PromptChange(
                name,
                hash(oldContent),
                hash(content),
                currentUser(),
                Instant.now()));
        return result;
    }

    private String hash(String content) {
        try {
            byte[] digest = MessageDigest.getInstance("SHA-256")
                    .digest(content.getBytes(UTF_8));
            return HexFormat.of().formatHex(digest);
        } catch (NoSuchAlgorithmException e) {
            throw new IllegalStateException(e);
        }
    }
}
```

## 8.3 差异对比与查询

审计日志应支持按模板名、操作人、时间段查询，并能对任意两个版本做统一 diff（unified diff），把"改了什么"直接呈现给审批人。常见查询包括某模板的全部变更历史、某人的全部变更、某时间段的变更列表。审计日志应只增不改，并设置与业务日志一致的保留周期。

# 9. 团队协作

## 9.1 角色与权限

| 角色 | 权限 |
|------|------|
| 开发 | 创建草稿版本 |
| 评测 | 运行评测 |
| 负责人 | 审批发布 |
| 运维 | 回滚 |

权限设计原则：创建与发布分离，评测与审批分离，避免"自己改、自己批、自己发"。

## 9.2 发布流程

```
草稿 → 评测通过 → 审批 → 灰度 → 全量
```

每个环节都有明确的准入条件：

| 环节 | 准入条件 |
|------|----------|
| 草稿 → 评测 | 通过命名与变量校验 |
| 评测 → 审批 | 评测分数不低于基线 |
| 审批 → 灰度 | 负责人签字 |
| 灰度 → 全量 | 灰度指标无异常 |

## 9.3 审批门禁与自动化

发布函数在激活版本前应做两道校验：评测分数不低于基线、审批人非空；任一不满足即抛异常拒绝发布。门禁自动化能把"流程靠自觉"变成"流程靠系统"，是规模化后唯一可行的方式。

# 10. 生产注意事项

## 10.1 风险与对策清单

| 风险 | 建议 |
|------|------|
| 无版本 | 每次变更新版本 |
| 命名混乱 | 统一命名规范 |
| 变量漂移 | 用 record 承载 |
| 无灰度 | 按比例放量 |
| 无法回滚 | 保留历史版本 |
| 变更无审计 | 记录操作人 |
| 无评测 | 发布前评测 |

## 10.2 监控指标

需要重点监控：模板加载数量（启动校验，骤变告警）、灰度版本实际占比（偏离配置告警）、回滚次数（频繁回滚告警）、变量缺失率（大于 0 告警）、版本切换耗时（超阈值告警）。这些指标是灰度阶段判断"是否继续放量"的直接依据。

## 10.3 常见反模式

| 反模式 | 问题 | 正确做法 |
|--------|------|----------|
| 版本写进名称 | 名称无限膨胀 | 版本进目录 |
| 直接改线上模板 | 无历史、无回滚 | 新版本 + 灰度 |
| 模板里硬编码业务数据 | 无法复用 | 用变量注入 |
| 只测 happy path | 边界回归 | 覆盖边界用例 |
| 审计只记"已更新" | 无法追溯 | 记录 diff 与哈希 |

# 11. 小结

## 11.1 核心要点

- Prompt 管理五维度：版本、注册、灰度、回滚、审计。
- 用注册中心统一加载与命名。
- 命名规范：模块_用途_类型。
- 变量用 record 承载，避免漂移。
- 变更必须评测 + 审计 + 可回滚。

## 11.2 落地清单

- 建立版本目录与 manifest，纳入代码评审。
- 实现注册中心，启动即校验命名与变量。
- 灰度按用户 ID 稳定哈希，支持一键回滚。
- 审计记录哈希与 diff，只增不改。
- 把评测分数与审批人设为发布门禁。
