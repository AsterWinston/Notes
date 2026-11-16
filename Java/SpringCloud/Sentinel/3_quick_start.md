
# 1. 环境准备

| 组件 | 版本要求 |
|------|---------|
| JDK | 17+（推荐 21） |
| Spring Boot | 3.2+ |
| Spring Cloud | 2023.x |
| Spring Cloud Alibaba | 2023.0.1+ |
| Sentinel | 1.8.6+ |

# 2. 依赖引入

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-alibaba-dependencies</artifactId>
            <version>2023.0.1.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <!-- Sentinel 核心依赖 -->
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
    </dependency>

    <!-- Spring Boot Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Sentinel Dashboard 通信（推荐） -->
    <dependency>
        <groupId>com.alibaba.csp</groupId>
        <artifactId>sentinel-transport-simple-http</artifactId>
    </dependency>
</dependencies>
```

# 3. 配置文件

```yaml
server:
  port: 8080

spring:
  application:
    name: sentinel-demo

  cloud:
    sentinel:
      transport:
        dashboard: localhost:8080     # Dashboard 地址
        port: 8719                    # 客户端与 Dashboard 通信端口

      # 取消 Dashboard 懒加载（启动即注册）
      eager: true

      # 数据源（Nacos 持久化）
      datasource:
        flow:
          nacos:
            server-addr: localhost:8848
            data-id: ${spring.application.name}-flow-rules
            group-id: SENTINEL_GROUP
            data-type: json
            rule-type: flow

      # 全局默认规则
      filter:
        enabled: true                  # 启用 Servlet Filter 自动拦截
```

# 4. 第一个限流示例

## 4.1 使用 @SentinelResource 注解

```java
@RestController
@RequestMapping("/api")
public class DemoController {

    @GetMapping("/hello")
    @SentinelResource(
        value = "sayHello",              // 资源名称
        blockHandler = "handleBlock",    // 限流/降级后的处理方法
        fallback = "handleFallback"      // 业务异常后的处理方法
    )
    public String sayHello(@RequestParam(defaultValue = "World") String name) {
        return "Hello, " + name;
    }

    // blockHandler: 参数与返回值与原方法一致，额外 BlockException
    public String handleBlock(String name, BlockException ex) {
        return "Too many requests! Please try later.";
    }

    // fallback: 参数与返回值与原方法一致，额外 Throwable
    public String handleFallback(String name, Throwable ex) {
        return "Service error, please try later.";
    }
}
```

## 4.2 使用 API 方式

```java
@RestController
@RequestMapping("/api")
public class DemoController {

    @GetMapping("/user/{id}")
    public String getUser(@PathVariable Long id) {
        // 定义资源
        try (Entry entry = SphU.entry("getUser")) {
            // 业务逻辑
            return "User: " + id;
        } catch (BlockException e) {
            // 被拦截
            return "Request blocked!";
        }
    }
}
```

# 5. 配置流控规则

## 5.1 通过 Dashboard 配置（推荐）

1. 启动 Dashboard：`java -jar sentinel-dashboard.jar`
2. 访问 `http://localhost:8080`，默认账号/密码：`sentinel/sentinel`
3. 找到应用 → 簇点链路 → 选择资源 → 新增流控规则

## 5.2 通过代码配置

```java
@PostConstruct
public void initFlowRules() {
    List<FlowRule> rules = new ArrayList<>();

    FlowRule rule = new FlowRule();
    rule.setResource("sayHello");                    // 资源名
    rule.setGrade(RuleConstant.FLOW_GRADE_QPS);     // QPS 限流
    rule.setCount(10);                               // 阈值 10
    rule.setLimitApp("default");                     // 默认对所有调用方生效
    rules.add(rule);

    FlowRuleManager.loadRules(rules);
}
```

## 5.3 通过 Nacos 持久化

发送 POST 请求到 Nacos 创建配置：

```bash
curl -X POST "http://localhost:8848/nacos/v1/cs/configs" \
  -d "dataId=sentinel-demo-flow-rules" \
  -d "group=SENTINEL_GROUP" \
  -d "content=[
    {
      \"resource\": \"sayHello\",
      \"grade\": 1,
      \"count\": 10,
      \"limitApp\": \"default\"
    }
  ]"
```

# 6. 验证效果

```bash
# 正常请求
curl http://localhost:8080/api/hello?name=Test
# Hello, Test

# 高频请求（超过 10 QPS 阈值）
for i in {1..20}; do
  curl http://localhost:8080/api/hello &
done
# Too many requests! Please try later.   ← blockHandler 生效

# 查看实时监控
curl http://localhost:8719/metrics
```

# 7. 关键注解速查

| 注解 | 作用 | 位置 |
|------|------|------|
| `@SentinelResource` | 声明资源、指定 blockHandler/fallback | 方法上 |
| `@SentinelRestTemplate` | 为 RestTemplate 添加 Sentinel 保护 | Bean 声明上 |
| `blockHandler` | 流控/降级时的处理方法 | `@SentinelResource` 属性 |
| `fallback` | 业务异常时的处理方法 | `@SentinelResource` 属性 |

# 8. 常见启动问题

| 问题 | 原因 | 解决 |
|------|------|------|
| Dashboard 看不到应用 | 未配置 `transport.port` | 添加 `spring.cloud.sentinel.transport.port=8719` |
| 规则不生效 | 资源未触发首次调用 | 先调用一次接口，Sentinel 默认懒加载 |
| 端口冲突 | 8719 已占用 | 更换 `transport.port` |
| 日志刷屏 | 默认打印拦截日志 | 配置 `csp.sentinel.log.dir` 指定目录 |
