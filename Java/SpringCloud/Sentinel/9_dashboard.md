
# 1. 概述

Sentinel Dashboard（控制台）是一个轻量级的 Web 管理平台，提供**规则管理、实时监控、机器发现**等功能。

```
┌──────────────┐    HTTP/API    ┌─────────────────────┐
│   Sentinel    │ ←─────────── │  Sentinel Dashboard  │
│   Client 1    │   心跳+规则   │                      │
└──────────────┘               │  ┌─────────────────┐ │
                               │  │  规则管理        │ │
┌──────────────┐               │  │  实时监控        │ │
│   Sentinel    │ ←─────────── │  │  机器发现        │ │
│   Client 2    │               │  │  集群流控        │ │
└──────────────┘               │  └─────────────────┘ │
                               └─────────────────────┘
```

# 2. 部署方式

## 2.1 直接运行

```bash
# 1. 下载 Dashboard JAR
wget https://github.com/alibaba/Sentinel/releases/download/1.8.6/sentinel-dashboard-1.8.6.jar

# 2. 启动（默认端口 8080）
java -Dserver.port=8080 \
     -Dcsp.sentinel.dashboard.server=localhost:8080 \
     -Dproject.name=sentinel-dashboard \
     -jar sentinel-dashboard-1.8.6.jar
```

## 2.2 Docker 部署

```bash
docker run -d \
  --name sentinel-dashboard \
  -p 8080:8080 \
  -e JAVA_OPTS="-Dserver.port=8080" \
  bladex/sentinel-dashboard:1.8.6
```

## 2.3 Docker Compose

```yaml
version: '3'
services:
  sentinel-dashboard:
    image: bladex/sentinel-dashboard:1.8.6
    container_name: sentinel-dashboard
    ports:
      - "8080:8080"
    environment:
      - JAVA_OPTS=-Dserver.port=8080
      - SENTINEL_AUTH_USERNAME=sentinel
      - SENTINEL_AUTH_PASSWORD=sentinel
```

# 3. 客户端连接配置

```yaml
spring:
  cloud:
    sentinel:
      transport:
        dashboard: localhost:8080      # Dashboard 地址
        port: 8719                     # 客户端通信端口（每个实例唯一）
        client-ip: 192.168.1.100       # 可选：指定上报 IP
      eager: true                      # 启动时立即注册
```

**关键配置说明**：

| 配置项 | 说明 | 默认值 |
|--------|------|--------|
| `dashboard` | Dashboard 地址 | — |
| `port` | 客户端启动的 HTTP Server 端口，用于接收 Dashboard 推送的规则 | 8719（同一机器多个实例需不同端口） |
| `eager` | 是否启动即注册（不等待首次资源调用） | false |
| `heartbeat-interval-ms` | 心跳间隔 | 10000 |

# 4. 核心功能

## 4.1 流控规则管理

在 Dashboard 中：
1. 左侧菜单 → 流控规则
2. 选择应用 → 新增流控规则
3. 配置：资源名、阈值类型、阈值、流控模式、流控效果
4. 保存 → 规则立即推送到目标机器

## 4.2 实时监控

Dashboard 提供秒级监控面板：
- **通过 QPS**：每秒通过的请求数
- **拒绝 QPS**：每秒被流控/降级拒绝的请求数
- **响应时间**：平均 RT、P99
- **线程数**：当前并发数

## 4.3 机器发现

Dashboard 通过心跳自动发现 Sentinel Client：
- Client 启动后每 10s 发送心跳
- Dashboard 维护机器列表及其健康状态
- 失联超过阈值自动标记为离线

# 5. Dashboard 规则推送模式

| 模式 | 原理 | 规则持久化 | 推荐场景 |
|------|------|-----------|---------|
| **原始模式** | Dashboard → Client 直接推送 | ✗ 内存，重启丢失 | 开发测试 |
| **Pull 模式** | Client 定时从数据源拉取 | ✓ 持久化 | 简单生产 |
| **Push 模式** | 数据源变更 → 推送 Client | ✓ 持久化 | **生产推荐** |

## 5.1 Push 模式改造（Nacos 持久化）

```java
@Configuration
public class SentinelPushConfig {

    @PostConstruct
    public void init() {
        // 改造为 Push 模式，需要修改 Dashboard 源码或使用
        // 社区方案 sentinel-dashboard-nacos
    }
}
```

**推荐方案**：使用 `sentinel-datasource-nacos` + 手动向 Nacos 发布规则，Dashboard 仅用于监控可视化。

# 6. 集群流控

## 6.1 原理

```
                    ┌──────────────┐
                    │  Token Server │  ← 独立部署的令牌服务
                    │  (全局计数)    │
                    └──┬────────┬──┘
              申请令牌 │        │ 申请令牌
                    ┌──▼──┐  ┌──▼──┐
                    │Client│  │Client│
                    │  A   │  │  B   │
                    └─────┘  └─────┘
```

Dashboard 中配置集群流控需要：
1. 部署 Token Server
2. Dashboard → 集群流控 → 新增规则
3. 配置 ClusterClient 连接 Token Server

# 7. 安全配置

```yaml
# Dashboard 登录认证
sentinel:
  dashboard:
    auth:
      username: sentinel
      password: sentinel
```

或通过环境变量：

```bash
java -Dsentinel.dashboard.auth.username=admin \
     -Dsentinel.dashboard.auth.password=admin123 \
     -jar sentinel-dashboard.jar
```

# 8. 常用 API

| API | 用途 |
|-----|------|
| `GET /registry/machine` | 查询机器列表 |
| `GET /metric/queryTopResourceMetricSorted` | 查询监控指标 |
| `POST /v1/flow/rule` | 新增流控规则 |
| `GET /v1/flow/rules` | 查询流控规则 |
| `POST /v2/flow/rule` | v2 格式新增流控 |
| `GET /cluster/server/{app}/page` | 查询集群 Token Server |
