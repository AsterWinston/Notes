# 1. 概述

Sentinel Dashboard（控制台）是一个轻量级的 Web 管理平台，提供**规则管理、实时监控、机器发现**等功能。

```text
┌──────────────┐    HTTP/API    ┌─────────────────────┐
│   Sentinel    │ ←─────────── │  Sentinel Dashboard  │
│   Client 1    │   心跳+规则   │                      │
│   Client 2    │               │  ┌─────────────────┐ │
└──────────────┘               │  │  规则管理        │ │
                               │  │  实时监控        │ │
┌──────────────┐               │  │  机器发现        │ │
│   Sentinel    │ ←─────────── │  │  集群流控        │ │
│   Client 2    │               │  └─────────────────┘ │
└──────────────┘               └──────────────────────┘
```

## 1.1 先立好对它的期望：一个"无状态遥控器"

官方 Dashboard 的定位要一句话钉死：**它自己什么都不持久**。规则存在它的内存里（重启即空，除非接了可写回的数据源改造版）、监控曲线存在它的内存环形缓存里（只看得到最近一段）、机器列表靠心跳维持（Dashboard 重启，应用列表要等下一轮心跳重新出现）。所以"控制台数据怎么没了"从来不是 bug，是形态；生产上把它当**观测窗 + 临时操作台**，规则的权威存储放 Nacos/数据库（§5 与 `14_dynamic_rules.md`），要长期监控就对接 Prometheus 拉取或 liveness 探针改造——这个期望管理比任何功能介绍都重要。

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

## 2.4 部署命令的三处注脚

§2.1 里 `-Dcsp.sentinel.dashboard.server=localhost:8080` 对 Dashboard **自身**其实可有可无（那是客户端上报用的参数，官方 README 带着它只为让控制台把自己也当应用注册看一眼监控页），删掉不影响任何功能；`-Dproject.name` 才决定它在应用列表里的显示名。§2.2/2.3 的 `bladex/sentinel-dashboard` 是社区镜像（官方不发 Docker 镜像），标签要跟版本核对——社区镜像版本漂移出过监控协议不兼容的案例。容器化部署最容易漏的一条：**Dashboard 要能反连每个应用的 8719±**（拉指标走的是 Dashboard→Client 方向的 HTTP），跨宿主机/K8s 网络时应用列表正常、曲线全空几乎必然是这条单向通道断了——排障先 `curl http://<pod-ip>:8719/metrics` 验证可达性，比查应用配置快。

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

## 3.1 心跳默认值纠偏与端口语义补全

`transport.heartbeat-interval-ms` 的默认值是 **5000（5 秒）**，不是表中写的 10000——§4.3 的"每 10s 心跳"同源纠正。这个数字的实践意义：心跳失败后 Dashboard 侧机器"离线"判定与重新出现的延迟都在秒级到十几秒（心跳连续失败才摘除），改大它省的是微乎其微的流量，亏的是故障时机器列表的时效，**保持默认即可**。`port` 一行的语义只说了一半：Command Center 同时服务**规则命令（Dashboard→Client）与指标拉取（Dashboard→Client `/metrics`）**两个方向，所以它不是"推规则专用口"；"同一机器多实例需不同端口"有自动救济——8719 被占会 +1 递增尝试并随心跳上报真实端口，Dashboard 跟随，手工指定反而容易在滚动发布时留死配置。`client-ip`（底层属性 `csp.sentinel.heartbeat.client.ip`）的场景是容器/NAT：默认取本机 route 地址，K8s 里 Pod IP 对集群外 Dashboard 不可路由时必须显式给可寻址 IP（NodePort/宿主网络），否则"注册成功但永远拉不到数"。

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

## 4.4 功能页背后的三个实现真相

**"立即推送"的落点**（§4.1 第 4 步）：保存动作是 Dashboard 逐机器调 Command Center 写**内存** RuleManager——多版本一致性靠"逐个推"串行完成，机器多时列表尾部有秒级参差；重启即失效（§1.1），所以这套操作在生产只应该出现在两种时刻：接了持久化数据源之前的临时演练、或紧急止血且事后回写 Nacos（`2_architecture.md` §6.1 纪律）。**曲线数据流**（§4.2）：面板数字来自 Dashboard 周期拉各机器 `/metrics`（1s 粒度原始点，控制台展示的是它内存里的拼接），**按机器分列**——"总览曲线"是多机聚合视图，单机毛刺会被平均掉，排查"QPS 掉了"务必点开具体机器看分实例曲线。**P99 要打个折**：控制台 RT 曲线的主值是**平均 RT**，分位数据依赖客户端 `csp.sentinel.metric.file.*` 落盘与上报配置，未开时 P99 显示缺失或恒零——拿它做 SLA 结论前先确认口径，精确分位看业务侧 Micrometer/APM（`21_logging_monitoring.md`）。**懒加载机器页**：簇点链路只有"被调用过的资源"，上线后没人打的接口不出现在列表里，新增规则时对不上资源名先想这条（`3_quick_start.md` §6.2 第 1 步同源）。

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

## 5.2 改造谱系：三档方案按成本排

§5.1 的推荐（客户端接 Nacos 读、控制台**只读监控**、规则变更走 Nacos 控制台/OpenAPI/流水线）是成本最低的正解——它没有解决"在 Sentinel Dashboard 里改规则"的体验，但把**权威存储+变更审计**交给了配置中心，多数团队到这一档就停了。要保留控制台编辑体验的再往上：**改 Dashboard 源码**（规则 Repository 换成 NacosDataSource 写实现，官方 README 给过样例，改动面是 FlowRuleProvider/Controller 一族，升级要跟改）；或直接换**带持久化改造的发行版**（bladex 镜像部分版本内置、各家开源控制台如 loongcollector-dashboard/liveness 类社区项目）；云上则 AHAS/各商业托管自带存储。选型判据一条：规则变更频率与操作人——低频、由发布流水线改（GitOps 管 JSON）走只读监控档即可；高频、由运营/值班在控制台手改才值得维护改造版 Dashboard。**无论哪档都要防"双写漂移"**：控制台内存态与 Nacos 态并存时，以数据源为唯一入口（控制台裸推用完必须回写，`2_architecture.md` §6.1）。

# 6. 集群流控

## 6.1 原理

```text
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

## 6.2 Token Server 的两种形态与选举

"独立部署的令牌服务"只画了形态之一。1.8.x 集群流控有**嵌入模式与专属模式**：嵌入模式下 Token Server **自动选举落在某个业务实例上**（默认行为——规则里不指定 namespace 归属时，第一台/按协商结果承担），优点零额外部署，缺点"发版重启那个实例=全局计数断档"（瞬时 `fallbackToLocalWhenFail` 退化单机阈值，全局量短时失真）；专属模式单独部署一个只跑 token server 角色的进程（`ClusterServerConfigManager` 配独立地址 + 应用以 client 身份连它），生产关键链路建议专属，代价是多一个运维对象与一跳网络。拓扑信息在 Dashboard"集群限流"页可见（当前 token server 在哪台、连接状态）；`flowId` 全局唯一且 client/server 两侧规则必须同步（改了 server 忘推 client 会出现"有 flowId 无阈值→拒绝或放行的参差"，`14_dynamic_rules.md` §10 治理清单）。压测结论提前给：全局阈值的单机语义换算问题（`4_flow_control.md` §6.1）它确实解决，但换来的每请求一跳 RPC 与 server 单点容量上限也要计入决策——QPS 万级以下的资源，多数时候"单机阈值×实例数换算 + 弹性联动"更划算。

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

## 7.1 认证强度：形同虚设的那层要补上

Dashboard 自带登录只有**单组用户名密码、无角色、无审计、无锁定策略**，且默认 `sentinel/sentinel` 全网皆知的密码是它的安全刻板印象——它从来不是面向公网的产品。三条硬纪律：**绝不暴露公网**（内网部署 + 入口加反代认证/OIDC 网关，§8 那批 API 与 UI 同权限面）；改默认口令只是及格线，账号写进部署模板避免"运维交接后没人知道密码"的私设；网络隔离兜底——8719± 的 Command Center 端口是**无认证 HTTP**（客户端间/Dashboard 对它的信任基于内网假设），跨信任域部署时这批端口才是真正要先堵的洞，规则 API 被任意调用的后果比 UI 登录被绕过大一个量级。

# 8. 常用 API

| API | 用途 |
|-----|------|
| `GET /registry/machine` | 查询机器列表 |
| `GET /metric/queryTopResourceMetricSorted` | 查询监控指标 |
| `POST /v1/flow/rule` | 新增流控规则 |
| `GET /v1/flow/rules` | 查询流控规则 |
| `POST /v2/flow/rule` | v2 格式新增流控 |
| `GET /cluster/server/{app}/page` | 查询集群 Token Server |

## 8.1 把这些 API 用起来的两条实操

这套 REST 与 UI 完全同源（UI 就是它的皮肤），所以**批量运维走脚本**：发布流水线改规则、多环境同步、规则审计导出都直接打这些端点（先 `/webapi/login` 拿 cookie，认证复用 §7 那组账）；v1/v2 的区别是 v2 支持 clusterConfig 等新字段，**新代码统一用 v2**。另一条：对**客户端**还有一族独立 API（各机器 Command Center：`/rules`、`/setting`、`/cluster/token` 等，`curl localhost:8719/rules` 这类）——它查的是"这台机器内存里现在真正生效的规则"，与控制台/Dashboard 显示的期望态不一致时，**以 Command Center 返回为准**定位推送丢失/覆盖竞态，这是集群规则排障的第一手证据链（§7.1 的无认证提醒同样适用于它们）。

# 9. 相关阅读

规则流转与持久化改造全图在 `14_dynamic_rules.md`；指标数字的生成端（`/metrics` 返回什么、窗口口径）在 `18_statistics_engine.md`；集群流控源码在 `16_flow_control_source.md`；替代控制台/商业方案与监控告警接线在 `21_logging_monitoring.md`；"看不到应用/曲线空白"完整决策树在 `24_troubleshooting.md`。
