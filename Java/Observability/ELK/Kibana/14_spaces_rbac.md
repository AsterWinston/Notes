
# 1. Spaces 概述

Spaces 是 Kibana 的**多租户隔离机制**，允许不同团队在同一 Kibana 实例中拥有独立的工作空间。

```
Default Space
  ├── Space: Team-A
  │     ├── Data Views: order-*, product-*
  │     ├── Dashboards: Order Dashboard
  │     └── Saved Objects: 仅 Team-A 可见
  │
  ├── Space: Team-B
  │     ├── Data Views: user-*, log-*
  │     ├── Dashboards: User Dashboard
  │     └── Saved Objects: 仅 Team-B 可见
  │
  └── Space: Operations
        ├── Data Views: metricbeat-*, filebeat-*
        └── Dashboards: Infra Monitoring
```

# 2. 创建 Space

```
Stack Management → Spaces → Create space
  Name: Team-A
  Description: Team A Workspace
  Features: 选择可见的功能模块
    - Discover
    - Dashboard
    - Visualize Library
    - Maps
    - Machine Learning (disable)
    - APM (disable)
```

```json
# API 创建
POST /api/spaces/space
{
  "id": "team-a",
  "name": "Team A",
  "description": "Team A Workspace",
  "disabledFeatures": ["ml", "apm", "siem"]
}
```

# 3. RBAC 角色

## 3.1 Kibana Privilege

| 权限 | 说明 |
|------|------|
| `all` | 所有权限 |
| `read` | 只读 |
| `minimal_read` | 仅可看 Dashboard（不可 Discover） |
| `none` | 无权限 |

## 3.2 创建角色

```
Stack Management → Roles → Create role

Role: team_a_viewer
  Cluster privileges: monitor
  Index privileges:
    - Indices: order-*, product-*
    - Privileges: read, view_index_metadata
  Kibana privileges:
    - Space: Team-A
    - Privilege: read

Role: team_a_admin
  Kibana privileges:
    - Space: Team-A
    - Privilege: all
```

```json
POST /_security/role/team_a_viewer
{
  "cluster": ["monitor"],
  "indices": [
    {
      "names": ["order-*", "product-*"],
      "privileges": ["read", "view_index_metadata"]
    }
  ],
  "applications": [
    {
      "application": "kibana-.kibana",
      "privileges": ["feature_discover.read", "feature_dashboard.read"],
      "resources": ["space:team-a"]
    }
  ]
}
```

# 4. Feature Control

```
Role → Kibana → Feature controls

按功能精细控制:
  Discover: All / Read
  Dashboard: All / Read / None
  Visualize: All / Read / None
  Canvas: All / Read / None
  Maps: All / Read / None
  Machine Learning: All / Read / None
  APM: All / Read / None
  Infrastructure: All / Read / None
  Dev Tools: All / Read / None
  Saved Objects Management: All / Read / None
```

# 5. 多租户方案

```
场景 1: 按团队隔离
  Space: team-a / team-b / team-c
  每个团队只看到自己的数据和 Dashboard

场景 2: 按环境隔离
  Space: production / staging / development
  Data View 指向不同环境的索引

场景 3: 按角色 + 功能
  Space: admin / analyst / viewer
  - admin: 全部权限
  - analyst: Discover + Dashboard (read)
  - viewer: 仅 Dashboard (read)
```

# 6. Saved Objects 跨 Space 复制

```
Stack Management → Saved Objects
  → 选择对象 → Copy to Space
  → 选择目标 Space

场景: 在 Team-A 中创建的 Dashboard 模板
  复制到 Team-B，调整 Data View 指向 Team-B 的索引
```

# 7. 最佳实践

1. **最小权限原则**：默认给 `read` 权限，按需提升到 `all`
2. **Data View 隔离**：每个 Space 独立 Data View，避免跨团队数据泄露
3. **生产 Space 限制**：生产环境禁用 Dev Tools（防止误操作）
4. **Space 命名规范**：`{team}-{env}` 如 `order-prod`, `order-staging`
5. **审计日志**：开启 Kibana Audit Log 记录访问和操作
