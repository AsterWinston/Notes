# 1. 项目结构

## 1.1 目录布局

### 1.1.1 结论先行

**Quarkus LangChain4j 项目遵循 Quarkus 标准 Maven 布局，AI 相关代码集中在 AI Service 接口、工具类、RAG 配置三处，资源目录负责提示词与配置。**

### 1.1.2 目录树

```
src/main/java/com/example
├── Assistant.java          AI Service 接口
├── ChatResource.java       REST 端点
└── config/                 配置
src/main/resources
├── application.properties  配置
└── prompts/                Prompt 模板
```

### 1.1.3 扩展后的建议布局

```
com.example
├── ai/                     AI Service 接口
│   ├── Assistant.java
│   └── Extractor.java
├── tools/                  工具类
│   └── WeatherTools.java
├── rag/                    RAG 配置
│   ├── RetrieverConfig.java
│   └── IngestService.java
├── api/                    REST 资源
│   └── ChatResource.java
└── config/                 全局配置
```

## 1.2 分层约定

### 1.2.1 建议分层

| 层 | 职责 | 是否含 LLM |
|----|------|------------|
| Resource | 协议、鉴权、参数校验 | 否 |
| Service | 业务编排、事务 | 否 |
| AI Service | 提示词、模型调用 | 是 |
| Tools | 外部能力封装 | 否 |

### 1.2.2 边界与坑

- 不要把业务逻辑写进 AI Service 接口的默认方法，代理行为难以预测。
- 工具类应为无状态或线程安全，模型可能并发调用。

### 1.2.3 依赖与布局

| 依赖 | 放置位置 |
|------|----------|
| AI Service 接口 | `ai/` |
| 工具类 | `tools/` |
| RAG 配置 | `rag/` |
| REST 资源 | `api/` |
| 提示词模板 | `resources/prompts/` |

## 1.3 生产建议

- 按职责分包，避免 AI 代码与业务代码混杂。
- 提示词模板外置到 `resources/prompts/`，便于评审与版本化。

# 2. 定义 AI Service

## 2.1 最小定义

### 2.1.1 代码

```java
@RegisterAiService
public interface Assistant {

    @SystemMessage("你是一个专业、简洁的中文助手。")
    String chat(@UserMessage String message);

    @UserMessage("""
            从下面的文本中抽取信息，输出 JSON：
            {"name":"","age":0,"job":""}
            文本：{{it}}
            """)
    Person extract(String text);
}
```

### 2.1.2 机制

构建期扩展为接口生成实现类：

```
接口 Assistant
   │ 构建期扫描 @RegisterAiService
   ▼
生成的实现（CDI Bean）
   ├── 拼装 SystemMessage / UserMessage
   ├── 调用 ChatLanguageModel
   ├── 若配置工具则进入工具循环
   └── 解析返回值为方法返回类型
```

## 2.2 提示词模板

### 2.2.1 @SystemMessage

系统提示定义角色、边界与输出约束，可放在接口上（全局）或方法上（局部）。

```java
@SystemMessage("你是客服助手，只回答产品相关问题，其他问题礼貌拒绝。")
```

### 2.2.2 @UserMessage 与占位符

- `{{it}}`：引用唯一参数。
- 命名占位符：`{{name}}` 对应方法参数名。

```java
@UserMessage("把下面内容翻译成{{lang}}：{{text}}")
String translate(@V("lang") String lang, @V("text") String text);
```

### 2.2.3 边界与坑

- 模板中的花括号需与 JSON 示例区分，避免误解析。
- 文本块（text block）中的缩进会原样进入提示词，注意空白噪声。

## 2.3 结构化输出

### 2.3.1 POJO 映射

```java
public record Person(String name, int age, String job) {}
```

返回类型为 POJO 时，扩展会要求模型输出可解析的结构并映射。

### 2.3.2 边界与坑

- 模型可能返回多余解释文字，导致解析失败，需容错。
- 嵌套对象与枚举的映射最易出错，应优先扁平化。

## 2.4 边界与坑

- AI Service 接口方法数量不宜过多，职责应单一。
- 方法重载可能导致代理选择歧义，避免使用。

## 2.5 生产建议

- 为每个 AI Service 编写契约测试，固定输入验证输出结构。
- 提示词变更走代码评审流程。

# 3. REST 端点

## 3.1 同步端点

### 3.1.1 代码

```java
@Path("/api/chat")
public class ChatResource {

    @Inject
    Assistant assistant;

    @POST
    @Produces(MediaType.TEXT_PLAIN)
    public String chat(String message) {
        return assistant.chat(message);
    }

    @GET
    @Path("/stream")
    @Produces(MediaType.SERVER_SENT_EVENTS)
    public Multi<String> stream(@QueryParam("message") String message) {
        return assistant.stream(message);   // 流式
    }
}
```

### 3.1.2 机制

RESTEasy Reactive 在事件循环或工作线程上执行；模型调用是阻塞 IO，应确认运行线程模型，避免阻塞事件循环。

## 3.2 流式端点

### 3.2.1 SSE 与 Multi

`Multi<String>` 是 Mutiny 的响应式流，`SERVER_SENT_EVENTS` 将其逐条推送给客户端。

```
客户端 ──GET /stream──► Resource ──stream()──► LLM
   ▲                                            │
   └────────── SSE 逐 token ────────────────────┘
```

### 3.2.2 边界与坑

- 流式响应中途失败时，HTTP 状态码已发送，无法改为错误码。
- 客户端断连需主动取消订阅，否则浪费 token。

## 3.3 边界与坑

- 直接用 `String` 接收请求体时，编码与大小限制需显式配置。
- 长文本请求应设置请求体大小上限，防止滥用。

## 3.4 生产建议

- 为 AI 端点单独设置限流与超时，区别于普通业务端点。
- 同步端点用于短回答，长回答一律走流式。

# 4. 工具

## 4.1 定义工具

### 4.1.1 代码

```java
@ApplicationScoped
public class WeatherTools {

    @Tool("查询城市天气")
    public String getWeather(@P("城市名") String city) {
        return city + "：晴，25℃";
    }
}
```

### 4.1.2 @Tool 与 @P

- `@Tool`：把方法暴露给模型，注解文本是给模型看的描述。
- `@P`：描述参数含义，帮助模型正确填参。

## 4.2 注册工具

### 4.2.1 类级注册

```java
@RegisterAiService(tools = WeatherTools.class)
public interface Assistant {
    String chat(String message);
}
```

### 4.2.2 方法级注册

也可在单个方法上限定可用工具，缩小模型可选范围。

```java
@RegisterAiService
public interface Assistant {

    @ToolBox(WeatherTools.class)
    String chat(String message);
}
```

> 具体注解名以扩展版本为准，不确定时用类级 `tools` 属性。

### 4.2.3 工具调用循环

```
用户问题
  → 模型决定调用工具
  → 框架执行 Java 方法
  → 结果回填给模型
  → 模型生成最终回答
  （可能循环多轮）
```

## 4.3 边界与坑

- 工具描述不清会导致模型误调用或漏调用。
- 工具内部异常应捕获并返回可读信息，否则整轮调用失败。
- 工具可被模型间接触发，必须做权限与参数校验。

## 4.4 生产建议

- 工具方法保持幂等，便于安全重试。
- 对危险工具（写库、发消息）加人工确认或权限门。

# 5. 记忆

## 5.1 会话作用域

### 5.1.1 代码

```java
@RegisterAiService
@SessionScoped
public interface Assistant {
    String chat(@MemoryId String userId, @UserMessage String message);
}
```

### 5.1.2 @MemoryId 机制

```
@MemoryId 值
   │
   ▼
ChatMemoryStore（按 id 分区）
   ├── 消息窗口（最近 N 轮）
   └── 超出窗口的历史被裁剪或摘要
```

### 5.1.3 记忆实现对比

| 实现 | 存储 | 多实例 | 适用 |
|------|------|--------|------|
| 内存窗口 | JVM 堆 | 不一致 | 开发 / 单实例 |
| Redis | 外部 | 一致 | 生产 |
| 数据库 | 外部 | 一致 | 审计需求 |

## 5.2 持久化记忆

### 5.2.1 Redis 存储

```properties
quarkus.langchain4j.redis.host=localhost
quarkus.langchain4j.redis.port=6379
```

### 5.2.2 边界与坑

- 内存记忆在多实例部署下不一致，需外部存储。
- 记忆无限增长会推高成本与延迟，必须设置窗口上限。

## 5.3 生产建议

- 生产环境使用外部 `ChatMemoryStore`，并设置 TTL。
- 敏感信息在写入记忆前脱敏。

# 6. 配置

## 6.1 基础配置

```properties
quarkus.langchain4j.openai.api-key=${OPENAI_API_KEY}
quarkus.langchain4j.openai.chat-model.model-name=gpt-4o-mini
quarkus.langchain4j.openai.chat-model.temperature=0.3
quarkus.langchain4j.openai.chat-model.timeout=30s
quarkus.langchain4j.openai.chat-model.log-requests=false
quarkus.langchain4j.openai.chat-model.log-responses=false
```

## 6.2 参数详解

### 6.2.1 参数表

| 参数 | 含义 | 建议 |
|------|------|------|
| api-key | 密钥 | 环境变量 |
| model-name | 模型 | 显式指定 |
| temperature | 温度 | 抽取 0~0.2 |
| timeout | 超时 | 显式设置 |
| log-requests | 请求日志 | 生产关闭 |
| log-responses | 响应日志 | 生产关闭 |

### 6.2.2 边界与坑

- `timeout` 过短会导致长回答被中断。
- 开启请求/响应日志会把用户数据写入日志，存在合规风险。

## 6.3 生产建议

- 按环境用 profile 覆盖模型与日志配置。
- 密钥与日志开关纳入上线检查清单。

# 7. 构建与启动

## 7.1 开发模式

### 7.1.1 热重载

```bash
# 开发模式（热重载）
./mvnw quarkus:dev
```

修改 Java 代码或配置后自动重载，无需重启。

### 7.1.2 调试

开发模式默认开放调试端口，可远程附加调试器。

## 7.2 打包与运行

### 7.2.1 JVM 模式

```bash
# 打包
./mvnw package
java -jar target/quarkus-app/quarkus-run.jar
```

### 7.2.2 原生模式

```bash
./mvnw package -Pnative
./target/*-runner
```

### 7.2.3 边界与坑

- 热重载不覆盖所有场景（如部分构建期配置），必要时手动重启。
- 原生模式构建耗时数分钟，需在 CI 单独阶段执行。

## 7.3 容器化

### 7.3.1 Dockerfile 片段

```dockerfile
FROM registry.access.redhat.com/ubi9/openjdk-21-runtime
COPY target/quarkus-app/ /deployments/
EXPOSE 8080
```

### 7.3.2 边界与坑

- 容器内存限制需与 JVM 堆参数协调，否则触发 OOM。
- 原生镜像基础镜像应选择精简运行时。

## 7.4 生产建议

- 开发用 JVM 模式，发布前跑一次原生构建。
- 把构建命令固化进 CI 脚本，避免手工差异。
- 容器镜像与配置分离，配置由环境注入。

# 8. 生产注意事项

## 8.1 风险清单

| 风险 | 建议 |
|------|------|
| 密钥硬编码 | 环境变量 |
| 日志泄露 | 生产关闭请求日志 |
| 无超时 | 配置 timeout |
| 无监控 | 集成 Micrometer |
| 工具越权 | 参数与权限校验 |
| 记忆膨胀 | 窗口上限 + TTL |

## 8.2 落地清单

- [ ] 密钥外置
- [ ] 超时与重试
- [ ] 请求/响应日志关闭
- [ ] 指标接入
- [ ] 工具权限校验
- [ ] 记忆持久化与上限

## 8.3 边界与坑

- 开发模式的宽松配置不应带入生产。
- 工具与记忆是常见事故源，需重点测试。

## 8.4 生产建议

- 建立 AI 端点的 SLO（延迟、错误率）。
- 对模型调用成本做预算与告警。

# 9. 小结

## 9.1 要点回顾

- `@RegisterAiService` 声明 AI Service。
- CDI 注入使用，REST 端点暴露。
- 工具、记忆通过注解配置。
- 开发模式支持热重载。
- 配置用 `quarkus.langchain4j.*`。
- 工具需权限校验，记忆需持久化与上限，密钥与日志需治理。

## 9.2 落地行动

- 为 AI 端点设置超时、限流与指标。
- 工具与记忆纳入安全评审清单。
