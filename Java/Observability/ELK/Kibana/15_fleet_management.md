
# 1. Fleet 架构

Fleet 是 Kibana 的**统一采集管理平台**，集中管理 Elastic Agent 的部署、策略和升级。

```
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

# 2. Elastic Agent

Elastic Agent 是**统一采集器**，取代传统的 Filebeat/Metricbeat/Packetbeat 等单一 Beat：

```
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

# 3. Agent Policy

```
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

# 4. Integration Policy

```
Fleet → Integrations → Browse integrations

支持的 Integration:
  应用: Nginx, Apache, MySQL, Redis, Kafka, RabbitMQ
  云: AWS, GCP, Azure
  安全: Elastic Defend, Windows Event, Sysmon
  系统: System, Docker, Kubernetes

每个 Integration 包含:
  ├── 预置 Dashboard
  ├── 预置 Data View
  ├── Ingest Pipeline
  └── Agent Template
```

# 5. Agent 自动升级

```
Fleet → Agents → Actions → Upgrade agent

升级策略:
  - 手动升级: 逐个 Agent 升级
  - 自动升级: Agent 策略中设置目标版本
  - 滚动升级: 分批升级（最大并行数）

升级流程:
  1. Fleet 下发新版本 Agent 二进制
  2. Agent 下载并校验
  3. Agent 停止 → 替换二进制 → 重启
  4. Agent 重新注册
```

# 6. 对比 Filebeat

| 维度 | Elastic Agent (Fleet) | Filebeat |
|------|----------------------|----------|
| 管理方式 | Fleet UI 集中管理 | 手动配置 YAML |
| 多采集 | 一个 Agent 多 Integration | 每类数据一个 Beat |
| Dashboard | 开箱即用 | 需手动配置 |
| 升级 | Fleet UI 一键升级 | 手动替换二进制 |
| 适用场景 | 新部署 | 已有 Beat 基础设施 |
