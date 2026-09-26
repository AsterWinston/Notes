
# 1. 环境要求

| 组件 | 版本要求 |
|------|---------|
| JDK | 11+（OAP Server），8+（应用使用 Agent） |
| 存储 | Elasticsearch 7.x+ / MySQL 5.7+ / H2（开发） |
| 内存 | OAP 至少 2GB，Agent 额外约 100MB |

# 2. 下载

```bash
# 下载最新稳定版
wget https://archive.apache.org/dist/skywalking/9.7.0/apache-skywalking-apm-9.7.0.tar.gz
tar -xzf apache-skywalking-apm-9.7.0.tar.gz
cd apache-skywalking-apm-bin

# 目录结构
tree -L 1
# ├── agent/              # Java Agent（探针）
# ├── bin/                # OAP + UI 启动脚本
# ├── config/             # OAP + UI 配置
# ├── oap-libs/           # OAP 依赖
# ├── webapp/             # UI Dashboard
# └── tools/              # 工具
```

# 3. OAP Server 部署

## 3.1 H2 存储（快速体验）

```bash
# 默认使用 H2 内存数据库，无需额外配置
bin/oapService.sh
# Windows: bin\oapService.bat

# 查看日志
tail -f logs/skywalking-oap-server.log
# SkyWalking OAP started successfully!
```

## 3.2 Elasticsearch 存储（生产推荐）

```yaml
# config/application.yml
storage:
  selector: elasticsearch
  elasticsearch:
    namespace: skywalking
    clusterNodes: localhost:9200
    protocol: http
    connectTimeout: 5000
    socketTimeout: 30000
    indexReplicasNumber: 1
    indexShardsNumber: 2
    recordDataTTL: 7         # 记录数据保留天数
    metricsDataTTL: 7        # 指标数据保留天数
```

## 3.3 Docker 部署

```bash
# OAP Server
docker run -d \
  --name skywalking-oap \
  -p 11800:11800 -p 12800:12800 \
  -e SW_STORAGE=elasticsearch \
  -e SW_STORAGE_ES_CLUSTER_NODES=es:9200 \
  apache/skywalking-oap-server:9.7.0

# UI Dashboard
docker run -d \
  --name skywalking-ui \
  -p 8080:8080 \
  -e SW_OAP_ADDRESS=http://oap:12800 \
  apache/skywalking-ui:9.7.0
```

# 4. Java Agent 配置

## 4.1 启动参数方式

```bash
java -javaagent:/path/to/skywalking-agent.jar \
     -DSW_AGENT_NAME=my-spring-app \
     -DSW_AGENT_COLLECTOR_BACKEND_SERVICES=127.0.0.1:11800 \
     -jar my-app.jar
```

## 4.2 agent.config 文件方式

```properties
# agent/config/agent.config
agent.service_name=${SW_AGENT_NAME:my-spring-app}
collector.backend_service=${SW_AGENT_COLLECTOR_BACKEND_SERVICES:127.0.0.1:11800}

# 采样配置
agent.sample_n_per_3_secs=1000          # 每秒最多采集 1000 条

# 忽略端点
agent.ignore_suffix=.jpg,.png,.css,.js,.ico

# 日志
logging.level=INFO
logging.dir=logs
```

## 4.3 IDE 开发环境

```
VM Options:
-javaagent:/path/to/skywalking-agent.jar
-DSW_AGENT_NAME=my-dev-app
-DSW_AGENT_COLLECTOR_BACKEND_SERVICES=127.0.0.1:11800
```

# 5. Spring Boot 应用集成

```xml
<!-- 无需任何 Maven 依赖！ -->
<!-- Java Agent 在 JVM 层透明注入 -->
```

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

启动后，访问几次 `/hello`，然后在 SkyWalking UI 中查看。

# 6. UI Dashboard 验证

1. 访问 `http://localhost:8080`
2. **General Service** → 选择 `my-spring-app`
3. 查看：
   - **Service Dashboard**：QPS、RT、成功率
   - **Topology**：服务拓扑图
   - **Trace**：完整的调用链路
   - **Endpoint**：端点列表和各自指标

# 7. 关键端口

| 端口 | 用途 | 配置项 |
|------|------|--------|
| 11800 | Agent → OAP gRPC | `core.default.grpcPort` |
| 12800 | UI → OAP GraphQL | `core.default.restPort` |
| 8080 | UI Dashboard HTTP | Webapp 端口 |

# 8. 常用配置速查

```properties
# agent.config 常用配置
agent.service_name=my-app                        # 服务名
collector.backend_service=127.0.0.1:11800        # OAP 地址
agent.sample_n_per_3_secs=1000                   # 采样率
agent.ignore_suffix=.jpg,.css                     # 忽略静态资源
plugin.jdbc.trace_sql_parameters=true             # 追踪 SQL 参数
plugin.mongodb.trace_param=true                   # 追踪 MongoDB 参数
```
