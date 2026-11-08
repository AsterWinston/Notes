# 1. Fleet 架构

Fleet 是 Kibana 的**统一采集管理平台**，集中管理 Elastic Agent 的部署、策略和升级。8.x 时代它是官方唯一推荐的采集管理入口——新集成的开发、文档、示例都围绕 Agent + Fleet 展开，Beats 进入维护状态。理解架构的关键是分清三种进程的角色：**Elastic Agent**（在被监控主机上采集数据）、**Fleet Server**（特化的 Agent，承担策略下发、事件收集、数据中转的管理面）、**ES + Kibana**（存储与控制台）。

```text
┌──────────────────────────────────────────────────────┐
│                     Kibana                            │
│              ┌──────────────┐                        │
│              │ Fleet Server │ ← 策略下发、状态收集     │
│              └──────┬───────┘                        │
└─────────────────────┼────────────────────────────────┘
                      │ HTTPS
         ┌────────────┼────────────┐
         ▼            ▼            ▼
   ┌──────────┐ ┌──────────┐ ┌──────────┐
   │  Agent 1 │ │  Agent 2 │ │  Agent 3 │  ← Elastic Agent
   │ (采集/安全)│ │ (采集/安全)│ │ (采集/安全)│
   └──────────┘ └──────────┘ └──────────┘
```

通信全部走 **HTTPS 出网方向**（Agent/Fleet Server 主动连 Kibana API，不需要被管机器开放入站端口），这对混合云/办公网环境很关键。数据链路有两种形态：**直连模式**（Agent 直接把数据写 ES，适合能直达 ES 的小规模网络）和 **Fleet Server 中转模式**（Agent → Fleet Server → ES，Fleet Server 充当统一出口、鉴权与缓冲层，**生产推荐**——ES 凭据只存在于 Fleet Server 上，边缘 Agent 只持有低权限的 enroll 令牌）。Fleet Server 自身要按无状态服务水平部署 2–3 个实例并放在独立的"隔离主机"上，它的宕机不影响已下发策略的 Agent 继续采集，只影响策略更新与状态上报。

# 2. Elastic Agent

Elastic Agent 是**统一采集器**，一个二进制内嵌了各 Beat 的能力（filestream、system-metrics 等以"组件"形式运行），取代传统的 Filebeat/Metricbeat/Packetbeat 等单一 Beat：

```text
传统 Beat 模式:
  每台机器安装多个 Beat → Filebeat(日志) + Metricbeat(指标) + Packetbeat(网络)

Elastic Agent 模式:
  一个 Agent → 按 Integration Policy 采集所有数据
```

```bash
# 安装 Elastic Agent
curl -L -O https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-8.11.0-linux-x86_64.tar.gz
tar xzf elastic-agent-8.11.0-linux-x86_64.tar.gz

# 注册到 Fleet
cd elastic-agent-8.11.0
./elastic-agent install \
  --fleet-server-es=http://es:9200 \
  --fleet-server-service-token=my-token
```

实际部署中更常见的是从 Kibana **Fleet → Add agents** 向导里复制带 **Enrollment Token** 的安装命令，形如：

```bash
sudo ./elastic-agent install \
  --url=https://fleet.example.com:8220 \
  --enrollment-token=ZXVyb3BlLXdlc3QtMTpFSy4uLnQtg  # 一次性注册令牌
```

**Enrollment Token 有两种**：API 生成的普通 token（一次性、默认 30 分钟过期）与 **Policy 级可复用 token**（长期、限定注册数量与来源 IP 段）——K8s DaemonSet、Auto Scaling 组这类"机器会不断新增"的场景必须用后者，否则每次扩容都要人工建 token。**自注册**（auto-enroll）即靠这个机制实现：cloud-init/Terraform user_data 里写死"下载安装 + 带 token 注册"脚本，新机器开机自动出现在 Fleet 列表，无需人工介入。

> **注意**：安装命令里的 `--unprivileged` 与 `--staged`、以及 `elastic-agent inspect`/`status`/`logs` 子命令是排障三件套；Agent 修改配置**不需要重启**（Fleet Server 下发新 snapshot 后组件级滚动重启），但本地手改 `elastic-agent.yml` 在 managed 模式下会被策略覆盖——**managed agent 的真相永远在 Kibana 的 Policy 里，不在机器上**，这是从 Beats 时代运维习惯迁移过来最容易踩的坑。

# 3. Agent Policy

Agent Policy 是**一组 Agent 的采集配置模板**：绑定了哪些 Integration、日志走哪个路由、升级窗口如何安排，下发后成为不可变快照（snapshot）：

```text
Fleet → Agent policies → Create agent policy

Policy: "Production - Web Servers"
  Integrations:
    ├── System (CPU/Memory/Disk/Network)
    ├── Nginx (Access Logs + Error Logs)
    ├── Custom Logs (/var/log/app/*.log)
    └── Elastic Defend (安全)

Agent 状态:
  Fleet → Agents
    ├── Healthy (绿色)
    ├── Unhealthy (红色)
    ├── Offline (灰色)
    └── Updating (蓝色)
```

**设计考量**：

- **Policy 按"机器角色"划分而非按"应用"划分**：一台机器同时跑 nginx 和应用日志，应属于同一个 policy、装多个 integration；web/db/k8s/边缘各一个 policy，边界与运维单元一致。
- **一个 Policy 一个 Datastream 命名空间**：integration 的输出索引（data stream）遵循 `<type>-<dataset>-<namespace>`，**namespace 默认取 policy 名**，所以"生产日志"与"预发日志"要装同一 integration 两次、分别放进不同 policy，靠 namespace 区分——这是最干净的隔离方式。
- **Fleet 的 Output 与 Kibana 自身的 ES 分离**：Settings → Outputs 定义数据写到哪个 ES 集群（支持多个），采集面与平台面解耦，边缘采集可以汇聚到独立的数据集群。

# 4. Integration Policy

Integration 是**官方/自建的采集打包格式**（elastic-package 规范），Browse Integrations 里数千个包覆盖主流软件：

```text
Fleet → Integrations → Browse integrations

支持的 Integration:
  应用: Nginx, Apache, MySQL, Redis, Kafka, RabbitMQ
  云: AWS, GCP, Azure
  安全: Elastic Defend, Windows Event, Sysmon
  系统: System, Docker, Kubernetes

每个 Integration 包含:
  ├── 预置 Dashboard
  ├── 预置 Data View
  ├── Ingest Pipeline (字段解析/ECS 映射)
  └── Agent Template (var: 日志路径、采集端点)
```

把一个 integration 添加进 policy 时配置的是它的 **Agent Policy Star**（变量集）：nginx 的访问日志路径、是否启用 stub status 指标采集、日志文件描述符限制等。集成质量按官方分级（`canonical`/`community`/`experimental`），生产优先选 canonical。自建集成可用 `elastic-package` 脚手架开发，本质是"agent 模板 + ingest pipeline + 样本数据 + 测试"的目录包，团队内部中间件的日志格式就适合沉淀成私有 integration 统一分发。

> **行业实践**：不要为了"用 Fleet"把已有的成熟 Beats 采集一次性推翻。官方口径也是如此——**新场景一律 Agent，存量 Beat 继续服役、按机器换代节奏自然迁移**；两者可长期共存（写同一批索引时注意 dataset 命名对齐，Kibana 侧 Data View 兼容 `filebeat-*` 与 `logs-*` 双模式）。

# 5. Fleet Server 与高可用

Fleet Server 以同样方式安装（选 "Fleet Server" policy），关键差异在角色参数与扩展模式。单实例只适合测试；生产按**每 1000–2000 Agent 一个 Fleet Server** 的经验值横向扩，并配置多实例 + 前置 LB（8220 端口）：

```bash
# Fleet Server 高可用：多个实例指向同一 Kibana，token 认证
./elastic-agent install \
  --fleet-server \
  --es-cert /etc/fs/es.crt --es-key /etc/fs/es.key \
  --fleet-server-es=https://es1:9200,https://es2:9200 \
  --fleet-server-policy=fleet-server-policy-id \
  --enrollment-token=<fleet-server-policy-token>
```

Fleet Server 用 **service account token**（而非普通用户）连 ES，权限最小化且可吊销。监控 Fleet 本身：Fleet 页面顶部的状态计数 + `.fleet-actions-results`、Agent 的 `unhealthy`/`degraded` 原因（`elastic-agent status` 输出组件级 health）应做成告警——**采集面的故障往往先于数据面显现**（索引断流 15 分钟没数据，多半是 Agent offline 而不是 ES 挂了）。

# 6. 升级与隔离

Agent 升级是 Fleet 相对 Beats 最大的运维红利，支持**滚动升级**：Actions → Upgrade agents，指定目标版本 + 最大并行数 + 维护窗口，按 policy 分批推进：

```text
升级流程:
  1. Fleet 下发新版本 Agent 二进制
  2. Agent 下载并校验
  3. Agent 停止 → 替换二进制 → 重启
  4. Agent 重新注册 (snapshot 重放, 组件逐个 restart)
```

8.x 引入 **Upgrade Coordinator**（Kibana 内）统一管理升级批次，配合 Maintenance Window 避开业务高峰；升级失败会自动回滚到前一个 snapshot（`--staged` 安装模式下二进制保留多版本）。**隔离（Air-gapped）环境**的要点：Agent 下载升级包默认走 artifacts.elastic.co，离线环境需在 policy 中把 **Artifact Server 指向内网镜像**（Settings → Fleet → Artifact 地址改为自建 nginx 提供 `/downloads/` 目录）；enrollment 与策略下发本来就只需 HTTPS 到 Fleet Server，整体即可完全离网运行。K8s 场景 DaemonSet 升级遵循 maxUnavailable 滚动，**注意 Agent pod 与业务 pod 共用节点日志路径时必须挂 `readOnlyHostPath` + 容器逃逸防护参数**。

# 7. 对比 Beats 与排障速查

| 维度 | Elastic Agent (Fleet) | Filebeat |
|------|----------------------|----------|
| 管理方式 | Fleet UI 集中管理 | 手动配置 YAML |
| 多采集 | 一个 Agent 多 Integration | 每类数据一个 Beat |
| Dashboard | 开箱即用 | 需手动配置 |
| 升级 | Fleet UI 一键升级 | 手动替换二进制 |
| 配置一致性 | 策略 snapshot 强一致 | 靠配置管理工具保证 |
| 资源占用 | 内嵌多组件，常驻偏高 | 单职能、更省 |
| 适用场景 | 新部署 | 已有 Beat 基础设施 |

常见问题速查：

```text
Agent Offline      → 机器上 elastic-agent status; 检查到 Fleet Server 8220 连通性与证书
数据不进索引        → integration 的 namespace/output 指向; ingest pipeline 报错 (模拟 _simulate)
日志重复采集        → filestream 的 ID 冲突/同文件被两个 input 匹配; 清 registry 需谨慎
升级卡 Updating    → artifact 下载失败(离线环境查 mirror); 看 /var/log/elastic-agent/ 日志
CPU 超限          → 调 policy 里 integration 的采集频率/harvester 数; 或拆分 policy
```

# 8. 小结

- **架构**：Agent 采集、Fleet Server 中转与管理、ES 存储三层，全链路 HTTPS 出网，ES 凭据收敛在 Fleet Server，边缘只持 enrollment token。
- **配置哲学**：managed agent 的真相在 Policy 不在机器；Policy 按机器角色划分，靠 data stream namespace 做环境隔离；自注册用 policy 级可复用 token 配合云-init。
- **运维红利**：滚动升级 + 失败回滚 + 离网 artifact 镜像，是 Beats 时代手工运维做不到的；监控 Fleet 状态要前置到采集面，断流先查 Agent。
- **迁移策略**：新场景一律 Agent，存量 Beat 不强行推翻，长期共存但注意 dataset 命名对齐。
