
# 1. SIEM 概述

Kibana Security 提供**安全信息事件管理（SIEM）**能力，包括威胁检测、调查和响应。

```
Security 功能:
  ├── Detection Rules  → 自动检测威胁
  ├── Timeline         → 事件调查时间线
  ├── Cases            → 安全案例管理
  ├── Hosts            → 主机安全分析
  └── Network          → 网络安全分析
```

# 2. Detection Rules

## 2.1 规则类型

| 类型 | 说明 | 示例 |
|------|------|------|
| **Query** | 基于 ES 查询检测 | 10 分钟内 SSH 失败 > 20 次 |
| **Machine Learning** | ML Job 异常 | 网络流量异常 |
| **Threshold** | 指标阈值 | 请求数 > 10000/s |
| **Event Correlation** | 多事件关联 | A 事件 → 5min 内 B 事件 |
| **Indicator Match** | 威胁情报匹配 | IP 在已知恶意 IP 列表 |
| **New Terms** | 新出现值 | 从未见过的进程名 |

## 2.2 创建规则

```
Security → Rules → Create new rule

Type: Query
Index: winlogbeat-*
Query: event.code:4625 AND winlog.event_data.Status:0xC000006A
Threshold: count > 20 (for last 10 minutes)

Name: "Brute Force Login Attempt"
Severity: Critical
```

# 3. Timeline 调查

```
Security → Timelines → Create new

Timeline 允许手动探索和分析事件:

1. 添加查询
   Index: winlogbeat-*
   Query: host.name:"web-server-01"

2. 查看事件时间线
   ┌──────────────────────────────────────┐
   │ 10:00:01  用户 admin 登录成功         │
   │ 10:01:15  用户 root SSH 登录失败      │
   │ 10:01:16  用户 root SSH 登录失败       │
   │ 10:01:17  用户 root SSH 登录失败       │
   │ 10:02:00  用户 root SSH 登录成功       │
   └──────────────────────────────────────┘

3. 保存 Timeline → 附加到 Case
```

# 4. Cases 案例管理

```
Security → Cases → Create case

Case: "Web Server Compromised"
  ├── Description: 描述安全事件
  ├── Assignee: 分配给安全分析师
  ├── Tags: bruteforce, ssh, production
  ├── Timeline: 关联调查时间线
  ├── Comments: 团队协作讨论
  └── Connectors: 自动创建 Jira Ticket / ServiceNow Incident
```

# 5. Hosts 面板

```
Security → Hosts

Host 列表:
  Host Name         IP           Alerts  Risk Score
  web-server-01    10.0.1.10    3       75 (High)
  db-server-01     10.0.1.20    0       10 (Low)

点击 Host → 查看:
  ├── Authentications (登录活动)
  ├── Uncommon Processes (异常进程)
  ├── Anomalies (ML 检测的异常)
  └── Events (原始事件列表)
```

# 6. Network 面板

```
Security → Network

网络流量分析:
  ├── DNS 查询统计
  ├── HTTP 请求分析
  ├── TLS 证书信息
  ├── Flow 记录（来源/目标 IP + 端口）
  └── 外部 IP 流量（识别出站到可疑 IP）
```

# 7. 告警集成

```
Security → Alerts

Detection Rules 触发 → 生成 Alert:
  - Alert Table: 所有告警列表
  - Alert Details: 关联事件详情
  - Alert Actions: 发送到 Connector
  - Alert Status: Open → Acknowledged → Closed
```
