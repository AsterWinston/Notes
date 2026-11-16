
# 1. 环境要求

SkyWalking 快速上手的目标是**把 Agent → OAP → Storage → UI 的最小可观测闭环搭起来**：Java Agent 挂在业务进程里负责采集，OAP Server 负责接收、分析与查询服务，存储负责持久化，UI 只是依赖 OAP 的瘦客户端。能跑通"挂 agent → 打流量 → UI 上看到拓扑和 Trace"，起步环境就算验收通过。

**版本要求是最容易踩错的第一关**。OAP Server 与 UI 需要 **JDK 11+**（9.x 建议直接上 17），应用侧 Agent 仍支持 **JDK 8+**；Spring Boot 3 / Jakarta 命名空间的应用必须使用 **9.0+** 的 skywalking-java。9.x 起发行包默认存储为 **BanyanDB**，Elasticsearch/MySQL 可切换；基于 H2 的 `standalone` 内存存储仍是体验最快、依赖最少的方式。

| 组件 | 版本要求 |
|------|---------|
| JDK | 11+（OAP，建议 17），8+（挂 Agent 的应用），Spring Boot 3 需 Agent 9.x |
| 存储 | BanyanDB（9.x 默认）/ Elasticsearch 7.x+ / MySQL 5.7+ / H2 standalone（开发） |
| 内存 | OAP 至少 2GB（生产 4GB+），Agent 额外约 50–100MB |
| 网络 | OAP 的 11800（gRPC）与 12800（HTTP）需对 Agent/UI 可达 |

> **行业实践**：生产环境不建议单机一体化部署 demo 链路。典型形态是 **1 套存储集群（ES 或 BanyanDB）+ 2 台以上 OAP（无状态水平扩展）+ UI**，Agent 通过域名/VIP 寻址 OAP，避免写死 IP 后扩容无从下手。

# 2. 下载

Apache 官方发行包 `apache-skywalking-apm-x.y.z.tar.gz` 是**全家桶**：OAP、UI、工具与一份随包发布的 Java Agent 打包在一起。需要理清一个关系：**Java Agent 自 8.8 起以 skywalking-java 项目独立发布**，发行包里的 agent 版本可能落后于 agent 最新发布版；OAP 对上报协议保持向后兼容，"新 agent + 略旧 OAP"通常可用。

```bash
# 下载最新稳定版（以 9.7.0 为例）
wget https://archive.apache.org/dist/skywalking/9.7.0/apache-skywalking-apm-9.7.0.tar.gz
tar -xzf apache-skywalking-apm-9.7.0.tar.gz
cd apache-skywalking-apm-bin

# 目录结构
tree -L 1
# ├── agent/     # Java Agent（探针）
# ├── bin/       # OAP + UI 启动脚本
# ├── config/    # OAP + UI 配置
# ├── oap-libs/  # OAP 依赖
# ├── webapp/    # UI Dashboard
# └── tools/     # 工具
```

# 3. OAP Server 部署

OAP 本质是一条**流处理管道**：Receiver 收数据 → Analyzer 解析实体 → Aggregator 聚合指标 → Exporter 落存储（详见《2. 整体架构》）。起步阶段只需要关心两件事：**存储选什么、gRPC 端口通不通**，其余 receiver 配置均可用默认值。

## 3.1 H2 存储（快速体验）

9.x 发行包默认 `selector: banyan`；想零外部依赖，可把 `config/application.yml` 的存储选择器改为 `standalone`（基于 H2，数据在内存），直接启动：

```bash
bin/oapService.sh
# Windows: bin\oapService.bat
tail -f logs/skywalking-oap-server.log
# SkyWalking OAP started successfully!
```

启动成功的标志是日志出现 `SkyWalking OAP started successfully`，且监听 11800/12800。`standalone` 模式下 **重启即丢数据**，只适合验证 agent 是否上报成功。

## 3.2 Elasticsearch 存储（生产推荐）

ES 是目前落地面最广的后端：trace 明细、指标、日志、拓扑关系全部落 ES 索引，查询能力强，但索引量随端点数膨胀，需要规划好 TTL 与分片。

```yaml
# config/application.yml
storage:
  selector: elasticsearch
  elasticsearch:
    namespace: skywalking        # 索引前缀，多套环境共用集群时靠它隔离
    clusterNodes: localhost:9200
    protocol: http
    connectTimeout: 5000
    socketTimeout: 30000
    indexReplicasNumber: 1       # 副本数，生产至少 1
    indexShardsNumber: 2         # 分片数，按写入量评估
    recordDataTTL: 7             # 记录数据保留天数（trace/log）
    metricsDataTTL: 7            # 指标数据保留天数
```

**关键取舍在 TTL 与分片**：`recordDataTTL` 决定排障时"最多能往前查几天的 Trace"，一般 7–14 天；`metricsDataTTL` 决定历史趋势图的跨度，通常 15–30 天。分片数建索引后不能对存量修改，初期宁可保守，靠按天滚动索引自然分散写入。`namespace` 修改等同于换一套索引，OAP 会重新初始化 schema，老索引不会自动迁移。

## 3.3 Docker 部署

官方镜像把配置项全部转成了环境变量，规则是**大写 + 下划线替换点号**，例如 `storage.selector` 对应 `SW_STORAGE`、UI 侧对应 `SW_OAP_ADDRESS`。单容器示例：

```bash
docker run -d --name skywalking-oap -p 11800:11800 -p 12800:12800 \
  -e SW_STORAGE=elasticsearch -e SW_STORAGE_ES_CLUSTER_NODES=es:9200 \
  apache/skywalking-oap-server:9.7.0

docker run -d --name skywalking-ui -p 8080:8080 \
  -e SW_OAP_ADDRESS=http://oap:12800 apache/skywalking-ui:9.7.0
```

## 3.4 docker-compose 一键最小拓扑

把 ES、OAP、UI 与两个带 agent 的 demo 应用编排在一起，一条 `docker compose up -d` 即可看到跨服务拓扑：

```yaml
# docker-compose.yaml
version: "3.8"
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.16.3
    environment:
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xms1g -Xmx1g

  oap:
    image: apache/skywalking-oap-server:9.7.0
    depends_on: [elasticsearch]
    environment:
      - SW_STORAGE=elasticsearch
      - SW_STORAGE_ES_CLUSTER_NODES=elasticsearch:9200
    ports: ["11800:11800", "12800:12800"]   # gRPC 上报 / GraphQL 查询

  ui:
    image: apache/skywalking-ui:9.7.0
    depends_on: [oap]
    environment:
      - SW_OAP_ADDRESS=http://oap:12800
    ports: ["8080:8080"]

  # 两个 demo 应用：挂载 agent 目录，YAML anchor 复用公共配置
  demo-base: &demo
    image: eclipse-temurin:17-jre
    volumes: ["./agent:/skywalking/agent", "./demo:/app"]   # 发行包里的 agent 目录挂进来
    depends_on: [oap]
    entrypoint: ["java", "-javaagent:/skywalking/agent/skywalking-agent.jar", "-jar", "/app/app.jar"]

  order-demo:
    <<: *demo
    environment:
      - SW_AGENT_NAME=order-service
      - SW_AGENT_COLLECTOR_BACKEND_SERVICES=oap:11800

  product-demo:
    <<: *demo
    environment:
      - SW_AGENT_NAME=product-service
      - SW_AGENT_COLLECTOR_BACKEND_SERVICES=oap:11800
```

```bash
docker compose up -d
# 循环打流量后即可在 UI 看到 order→product 的跨服务拓扑
for i in $(seq 1 100); do curl -s http://localhost:8081/order/1 > /dev/null; sleep 0.2; done
```

> **注意**：compose 网络里 Agent 的 `collector.backend_service` 必须写**服务名**（`oap:11800`）而不是 `localhost:11800`——容器里的 localhost 指向容器自身，这是新手"数据不翼而飞"的头号原因。

# 4. Java Agent 配置

Agent 的加载方式是 JVM 标准的 `-javaagent` premain 挂载，**应用代码和构建产物完全不用改**。所有配置项都是 `agent/config/agent.config` 里的 key，且每个 key 都写成 `${环境变量名:默认值}` 的形式，同一份 agent 目录可以通过环境变量差异化启动——这正是容器化部署推荐环境变量方式的原因。配置生效优先级：**系统属性（-D）> 环境变量 > agent.config 文件默认值**。

## 4.1 启动参数方式

```bash
java -javaagent:/path/to/skywalking-agent.jar \
     -DSW_AGENT_NAME=my-spring-app \
     -DSW_AGENT_COLLECTOR_BACKEND_SERVICES=127.0.0.1:11800 \
     -jar my-app.jar
```

配置没生效时先用 `-D` 强制覆盖验证，再检查环境变量是否透传进 JVM 进程（容器里 `docker exec` 进去 `env | grep SW_` 看一眼最直观）。

## 4.2 agent.config 文件方式

```properties
# agent/config/agent.config
agent.service_name=${SW_AGENT_NAME:my-spring-app}
collector.backend_service=${SW_AGENT_COLLECTOR_BACKEND_SERVICES:127.0.0.1:11800}

# 采样配置：每 3 秒窗口内采样的 Trace 数；-1 表示全量（9.x 默认）
agent.sample_n_per_3_secs=${SW_AGENT_SAMPLE:-1}

# 忽略端点（按后缀）
agent.ignore_suffix=${SW_AGENT_IGNORE_SUFFIX:.jpg,.png,.css,.js,.ico}

# Agent 自身日志
logging.level=INFO
logging.dir=logs
```

`agent.sample_n_per_3_secs` 的语义要记准：**每个 3 秒窗口内最多完整上报多少条 Trace**，未命中的 Trace 只传播不存储（指标也不生成），并非"每秒上限"。老版本默认值是 3，高流量服务会看到大量端点"有指标无链路"的现象；9.x 默认改为 `-1` 全采，生产再按容量往下调。

## 4.3 IDE 开发环境

```text
VM Options:
-javaagent:D:/skywalking/agent/skywalking-agent.jar
-DSW_AGENT_NAME=my-dev-app
-DSW_AGENT_COLLECTOR_BACKEND_SERVICES=127.0.0.1:11800
```

IDEA 里在 Run Configuration → VM options 填入即可。Windows 路径注意用正斜杠或双反斜杠；`-javaagent` 路径解析失败时 JVM 会**直接启动失败**，这类错误反而最容易发现。

# 5. Spring Boot 应用集成

SkyWalking 对 Spring Boot 是**零依赖侵入**：不引任何 Maven 坐标、不改一行代码，字节码增强全部由 agent 在类加载时完成。这与 Sleuth/Micrometer 那种"引 starter 写配置"的思路截然不同——代价是排查问题时需要理解 agent 插件机制，收益是探针升级和业务代码彻底解耦。

```java
@SpringBootApplication
@RestController
public class DemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }

    @GetMapping("/hello")
    public String hello() {
        return "Hello SkyWalking";
    }
}
```

挂上 agent 启动后循环访问几次 `/hello`，UI 上就会出现 `my-spring-app`。**验证顺序建议**：服务是否注册 → 端点列表是否有 `/hello` → 点开一条 Trace 看 segment/span 结构，三步都通说明采集、传输、存储、查询四层全部正常。
# 6. UI Dashboard 验证

1. 访问 `http://localhost:8080`
2. **General Service** → 选择 `my-spring-app`
3. 查看：**Service Dashboard**（QPS/RT/成功率）、**Topology**（demo 双应用应显示 order→product 调用边）、**Trace**（可按端点/耗时/异常过滤）、**Endpoint**（端点列表与各自指标）

> **注意**：UI 默认时区跟随浏览器，时间选错"看不到数据"其实是查错了窗口；刚启动的前 1–2 分钟因分钟级聚合尚未滚动完成，图表可能空白，属正常现象。

# 7. 关键端口

| 端口 | 用途 | 配置项 |
|------|------|--------|
| 11800 | Agent → OAP gRPC（原生 sw8 与 OTLP gRPC 共用） | `core.default.grpcPort` |
| 12800 | UI → OAP GraphQL/HTTP（与 OTLP HTTP 共用） | `core.default.restPort` |
| 8080 | UI Dashboard HTTP | Webapp 端口 |

9.x 的 OTLP receiver（OpenTelemetry 协议）默认**复用**核心端口：gRPC 走 11800、HTTP 走 12800；Zipkin receiver 则默认单独监听 9411，可按需改端口。因此防火墙策略通常只需开 11800/12800 就能同时接纳原生 sw8 与 OTLP 上报，但压测评估时要考虑共享端口下不同来源数据的相互挤占。

# 8. 常用配置速查

```properties
# agent.config 常用配置
agent.service_name=my-app                        # 服务名（拓扑的分组键，务必唯一且稳定）
collector.backend_service=127.0.0.1:11800        # OAP 地址（多实例逗号分隔）
agent.sample_n_per_3_secs=1000                   # 采样率（-1 全采）
agent.ignore_suffix=.jpg,.css                    # 忽略静态资源
plugin.springmvc.collect_http_params=true        # 采集 HTTP 参数（注意脱敏）
plugin.jdbc.trace_sql_parameters=true            # 追踪 SQL 参数（默认关闭）
agent.span_limit_per_segment=300                 # 单 segment span 上限（9.x 默认 300，老版本 50）
```

服务命名规范建议一开始就定死：`业务域-应用名`（如 `trade-order-service`），灰度/多环境用 instance 属性或 namespace 区分，而不是靠给 service_name 加随机后缀——后者会把拓扑搅乱，K8s 下 Pod 重启还会不断制造"幽灵服务"。

# 9. 版本策略与存储索引（sw 8.x → 9.x）

存储层的索引/表以 `storage.elasticsearch.namespace` 配置值为前缀（第 3.2 节示例配置为 `skywalking`，则索引形如 `skywalking_segment-20240115`、`skywalking_log-*`），schema 结构与**OAP 大版本**绑定，OAP 首次启动会在存储里写入版本元数据索引。**9.x OAP 可以直接读取 8.x 的历史索引平滑升级，但反向不支持**——用 8.x OAP 连 9.x 写过的存储，启动检查会直接报版本不兼容并退出，回滚唯一出路是清空索引。

> **注意**：生产升级 OAP 前先做存储快照（ES snapshot / MySQL dump）；Agent 侧（skywalking-java）独立版本号，通过 segment 上报协议与 OAP 协商，协议向后兼容。

# 10. 常见起步坑：Agent 不生效排查三查

"挂了 agent 但 UI 看不到服务"是起步期最高频问题。与其乱翻配置，不如按**一查挂载（agent.log 有无启动完成日志）、二查连接（gRPC 能否到 11800）、三查上报（采样/忽略/时区）**的顺序收敛：

**一查挂载**：看 `agent/logs/agent.log`。出现 `SkyWalking agent start completed` 说明 premain 成功、插件已装载；连这个文件都没有，就是 `-javaagent` 路径错或 jar 用错。Boot 3 应用挂旧 agent 的典型症状是日志里大量插件解析失败。

**二查连接**：agent.log 里若刷 `UNAVAILABLE: io exception` 或 `deadline exceeded`，是到 OAP 11800 的网络问题——容器里写 `localhost`、安全组没开、OAP 尚未就绪都会如此，`nc -vz <oap> 11800` 一秒定性。

**三查上报**：连接正常但 UI 无数据，检查采样是否为 0、端点是否被忽略规则命中、UI 时间窗与时区是否选对；再无则翻 OAP 日志看 receiver 与 storage 是否报错。

| 坑 | 症状 | 解法 |
|----|------|------|
| `-javaagent` 路径错误 | JVM 启动即失败 | 核对绝对路径与 jar 名 |
| 容器内写 localhost | agent.log 连接超时 | 改用 compose 服务名/宿主机 IP |
| 采样命中为 0 | 有指标无 Trace / 完全无数据 | `sample_n_per_3_secs` 调大或 -1 |
| 服务名重复混乱 | 拓扑出现幽灵服务 | 固定 service_name 命名规范 |

# 11. 小结

| 主题 | 要点 |
|------|------|
| 最小闭环 | Agent → OAP(11800/12800) → Storage → UI(8080)，四件套跑通即起步成功 |
| 存储选型 | demo 用 standalone（H2，重启丢数据），生产 ES/BanyanDB，TTL 与分片先规划 |
| 配置覆盖 | `${环境变量:默认值}` 三级优先：-D > 环境变量 > agent.config |
| 采样语义 | `sample_n_per_3_secs` 是 3 秒窗口的 Trace 条数，未采样不落库也不出指标 |
| 排查心法 | 一查挂载（agent.log）、二查连接（11800）、三查上报（采样/忽略/时区） |

下一篇《4. Java Agent 原理》将拆开 agent.log 背后的字节码增强与插件装载机制。
