# 1. 定位

结论先行：**Google ADK（Agent Development Kit，智能体开发套件）** 是 Google 推出的 Agent 开发框架，同时支持 Python 与 Java。它把"Agent 定义、工具、多 Agent 编排、部署运行时"打包为一套统一抽象，并与 Gemini 模型深度集成。

## 1.1 什么是 ADK

### 1.1.1 框架定位

```
ADK = Agent 定义 + 工具 + 多 Agent 编排 + 部署运行时
```

ADK 不是单纯的模型 SDK，而是覆盖"开发—编排—部署"全流程的框架。

| 层次 | ADK 提供 |
|------|----------|
| 开发 | Agent 抽象、指令、工具 |
| 编排 | 多 Agent 协作、工作流 |
| 运行 | Runner、Session、部署 |

```
ADK 组成
  ├── Agent 抽象（LlmAgent / WorkflowAgent）
  ├── Tools（函数、检索、第三方）
  ├── Orchestration（Sequential / Parallel / Loop）
  └── Runtime（Runner / Session / Artifact）
```

| 语言 | 状态 | 说明 |
|------|------|------|
| Python | 成熟 | 功能最全 |
| Java | 较新 | 本笔记重点 |

### 1.1.2 与 Gemini 的关系

ADK 由 Google 维护，对 Gemini 模型的支持最完整（原生多模态、函数调用、流式）。但框架在模型层做了抽象，理论上可接入其他模型。

## 1.2 设计哲学

### 1.2.1 代码优先

ADK 以代码（Python / Java）为中心，Agent 与工具都是代码对象，便于版本管理、测试与复用。

### 1.2.2 运行时与开发分离

开发阶段定义 Agent，运行阶段由 `Runner` 驱动。同一份定义可在本地调试，也可部署到托管运行时。

## 1.3 边界与反例

反例：把 ADK 当作"提示词模板库"使用，只用它拼提示词。这样会错失 Runner、Session 与多 Agent 编排能力。应把它当作完整的 Agent 运行时来设计。

| 反模式 | 后果 | 建议 |
|--------|------|------|
| 当模板库用 | 浪费运行时能力 | 用 Runner 驱动 |
| 状态放 Agent | 并发串数据 | 状态放 Session |
| 硬编码 Key | 泄露风险 | 用环境变量 |

# 2. 核心概念

结论先行：ADK 的核心概念可归纳为五类——Agent、Tool、Runner、Session、SubAgent。理解它们的关系是使用框架的前提。

## 2.1 概念总览

| 概念 | 说明 |
|------|------|
| `Agent` | 智能体（指令 + 模型 + 工具） |
| `Tool` | 可调用的工具 |
| `Runner` | 执行器 |
| `Session` | 会话状态 |
| `SubAgent` | 子 Agent（多 Agent 协作） |

### 2.1.1 Agent

Agent 是最小智能单元，包含名称、模型、指令与工具集合。常见类型有 `LlmAgent`（由大模型驱动）与工作流 Agent。

| 类型 | 驱动方式 | 用途 |
|------|----------|------|
| `LlmAgent` | 大模型 | 通用对话与推理 |
| `SequentialAgent` | 顺序 | 固定流水线 |
| `ParallelAgent` | 并行 | 并发子任务 |
| `LoopAgent` | 循环 | 迭代直到满足条件 |

```java
LlmAgent agent = LlmAgent.builder()
        .name("assistant")
        .model("gemini-2.0-flash")
        .instruction("你是一个专业的中文助手。")
        .build();
```

### 2.1.2 Tool

Tool 是 Agent 可调用的外部能力，可包装 Java 方法、HTTP API 或检索器。

```java
public class WeatherTool {
    public String getWeather(String city) {
        return weatherService.query(city);
    }
}

LlmAgent agent = LlmAgent.builder()
        .name("travel")
        .model("gemini-2.0-flash")
        .instruction("需要天气时调用 getWeather。")
        .tools(new WeatherTool())
        .build();
```

### 2.1.3 Runner 与 Session

`Runner` 负责驱动 Agent 执行；`Session` 保存一次对话的状态与历史。

```java
SessionService sessionService = InMemorySessionService.create();
Session session = sessionService.createSession("app", "user-1");

Runner runner = Runner.builder()
        .agent(agent)
        .sessionService(sessionService)
        .build();
```

## 2.2 概念关系

```
         ┌────────────┐
         │   Runner   │  驱动
         └─────┬──────┘
               ▼
         ┌────────────┐
         │   Agent    │
         └──┬──────┬──┘
            ▼      ▼
      ┌────────┐ ┌────────┐
      │  Tool  │ │SubAgent│
      └────────┘ └────────┘
               │
         ┌─────▼──────┐
         │  Session   │  状态
         └────────────┘
```

## 2.3 边界与反例

反例：把状态直接放在 Agent 对象的字段里。Agent 通常是可复用的无状态定义，状态应放在 `Session`，否则并发请求会串数据。

```java
// 错误：状态挂在 Agent 上，并发不安全
agent.setState("history", messages);

// 正确：状态放在 Session 中
session.state().put("history", messages);
```

# 3. 依赖引入

结论先行：通过 Maven 或 Gradle 引入 `google-adk` 即可。版本以官方最新为准，生产环境务必锁定版本。

## 3.1 Maven 依赖

### 3.1.1 核心依赖

```xml
<dependency>
    <groupId>com.google.adk</groupId>
    <artifactId>google-adk</artifactId>
    <version>0.1.0</version>
</dependency>
```

> 版本以官方最新为准。

### 3.1.2 可选依赖

| 能力 | 依赖 |
|------|------|
| Gemini 模型 | `google-adk` 内置 |
| 日志 | SLF4J + Logback |
| 测试 | JUnit 5 |

多模块项目建议用 BOM（Bill of Materials，物料清单）统一版本：

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.google.adk</groupId>
            <artifactId>google-adk-bom</artifactId>
            <version>0.1.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

## 3.2 Gradle 依赖

```groovy
dependencies {
    implementation 'com.google.adk:google-adk:0.1.0'
}
```

## 3.3 配置与凭证

### 3.3.1 环境变量

```bash
export GOOGLE_API_KEY=your-api-key
```

### 3.3.2 配置文件

```yaml
adk:
  model: gemini-2.0-flash
  api-key: ${GOOGLE_API_KEY}
```

反例：把 API Key 硬编码进代码或提交到仓库。必须使用环境变量或密钥管理服务。

# 4. 最小 Agent

结论先行：一个可运行的 Agent 只需三步——构建 `LlmAgent`、创建 `Runner`、用 `Session` 发起一次对话。

## 4.1 构建 Agent

### 4.1.1 LlmAgent.builder

```java
LlmAgent agent = LlmAgent.builder()
        .name("assistant")
        .model("gemini-2.0-flash")
        .instruction("你是一个专业的中文助手。")
        .build();
```

### 4.1.2 指令与模型

| 参数 | 含义 |
|------|------|
| `name` | Agent 名称，用于日志与路由 |
| `model` | 模型标识 |
| `instruction` | 系统指令 |

## 4.2 运行 Agent

### 4.2.1 Runner

```java
Runner runner = Runner.builder().agent(agent).build();
```

### 4.2.2 Session

```java
Session session = Session.create("session-1");
```

会话保存历史消息，多次调用共享上下文。

| 会话服务 | 适用 |
|----------|------|
| `InMemorySessionService` | 本地调试 |
| 持久化实现 | 生产多实例 |

会话历史会随轮次增长，需设置截断或摘要策略，避免超出模型上下文窗口。

## 4.3 完整示例

```java
public class Main {
    public static void main(String[] args) {
        LlmAgent agent = LlmAgent.builder()
                .name("assistant")
                .model("gemini-2.0-flash")
                .instruction("你是一个专业的中文助手。")
                .build();

        Runner runner = Runner.builder().agent(agent).build();
        Session session = Session.create("session-1");

        String reply = runner.run(session, "你好，介绍一下 ADK");
        System.out.println(reply);
    }
}
```

## 4.4 边界与反例

反例：每次请求都新建 `Runner`，导致连接与缓存无法复用。Runner 应作为长生命周期对象复用，Session 按会话创建。

流式场景可改用异步接口，逐事件消费：

```java
runner.runAsync(session, "你好")
        .subscribe(event -> System.out.println(event.content()));
```

# 5. 适用场景

结论先行：ADK 适合 Google 生态、需要多 Agent 编排或需要托管运行时的项目；不适合纯离线、对模型无要求或团队不愿引入新依赖的场景。

## 5.1 适用

| 场景 | 说明 |
|------|------|
| Google 生态 | Gemini 模型 |
| 多 Agent 编排 | 原生支持 |
| 需要部署运行时 | ADK 提供 |
| Java 团队 | Java SDK |

## 5.2 不适用

| 场景 | 原因 |
|------|------|
| 纯离线部署 | 依赖云模型 |
| 只用单一模型调用 | 直接 SDK 更轻 |
| 生态锁定敏感 | 框架较新 |

## 5.3 决策表

| 维度 | 选 ADK | 选其他 |
|------|--------|--------|
| 多 Agent | 是 | 否 |
| Gemini | 是 | 否 |
| Java 成熟度优先 | 否 | 是 |

也可以混合：用 ADK 做多 Agent 编排，用 Spring AI 做模型与 RAG，二者通过工具边界衔接。

# 6. 与其他框架对比

结论先行：ADK 的差异化在于"原生多 Agent + Google 运行时"，但在 Java 成熟度上不如 Spring AI 与 LangChain4j。

## 6.1 对比表

| 维度 | ADK | Spring AI | LangChain4j |
|------|-----|-----------|-------------|
| 出身 | Google | Spring | 社区 |
| 模型 | Gemini 优先 | 通用 | 通用 |
| 多 Agent | 原生 | 需扩展 | Agentic |
| Java 成熟度 | 较新 | 成熟 | 成熟 |
| 部署运行时 | 提供 | 需自建 | 需自建 |
| 工具抽象 | 原生 | 注解/函数 | 注解 |

## 6.2 选型建议

| 需求 | 建议 |
|------|------|
| 深度使用 Gemini | ADK |
| Spring 项目集成 | Spring AI |
| 通用模型 + 声明式 | LangChain4j |
| 多 Agent 编排 | ADK 或 LangChain4j |

# 7. 学习路径

结论先行：按"定义 Agent → 添加工具 → 多 Agent 编排 → 部署运行时"四步推进，每步都有可运行产出。

## 7.1 四步路径

```
① 定义 Agent（指令 + 模型）
② 添加工具
③ 多 Agent 编排
④ 部署运行时
```

## 7.2 每步产出

| 步骤 | 产出 | 验证方式 |
|------|------|----------|
| ① | 可对话 Agent | 命令行问答 |
| ② | 能调用工具 | 工具返回正确 |
| ③ | 多 Agent 协作 | 子任务委派 |
| ④ | 可部署服务 | 接口可用 |

```java
// 步骤②：添加工具
LlmAgent agent = LlmAgent.builder()
        .name("assistant")
        .model("gemini-2.0-flash")
        .instruction("可以调用工具完成任务。")
        .tools(new WeatherTool())
        .build();
```

```java
// 步骤③：多 Agent 编排（伪代码）
SequentialAgent pipeline = SequentialAgent.builder()
        .name("pipeline")
        .subAgents(researchAgent, writerAgent)
        .build();
```

# 8. 生产注意事项

结论先行：ADK Java SDK 较新，生产引入需评估成熟度、保持模型抽象、补齐可观测并锁定版本。

## 8.1 成熟度

Java SDK 较新，API 可能变动。建议先在非核心链路试点，跟踪发布说明。

| 版本阶段 | 建议 |
|----------|------|
| 0.x | 试点、勿上核心链路 |
| 1.x+ | 评估后再推广 |

## 8.2 模型绑定

避免在业务代码中散落模型名与厂商类型，集中配置以便替换。

```java
@Bean
LlmAgent assistant(ModelConfig config) {
    return LlmAgent.builder()
            .name("assistant")
            .model(config.modelName())
            .instruction(config.instruction())
            .build();
}
```

## 8.3 可观测

ADK 自带能力有限，需自行接入 Trace 与指标，记录每次工具调用与模型调用。

```java
runner.runAsync(session, input)
        .doOnNext(event -> log.info("event={}", event))
        .doOnError(e -> log.error("run failed", e))
        .blockLast();
```

## 8.4 版本管理

锁定依赖版本，使用 BOM 或版本目录统一管理，避免传递依赖冲突。

| 风险 | 建议 |
|------|------|
| Java SDK 较新 | 评估成熟度 |
| 绑定 Gemini | 保持模型抽象 |
| 无观测 | 加 Trace |
| 版本迭代 | 锁定版本 |

# 9. 小结

## 9.1 核心要点

- ADK 是 Google 的 Agent 开发框架，支持 Java。
- 核心概念：Agent、Tool、Runner、Session、SubAgent。
- 原生支持多 Agent 编排，与 Gemini 深度集成。
- Java SDK 较新，需评估成熟度。

## 9.2 落地清单

- 适合 Google 生态与 Gemini 模型。
- 按四步学习路径推进，每步可运行。
- 保持模型抽象、补齐可观测、锁定版本。
