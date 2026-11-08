# 1. Spaces 概述

Spaces 是 Kibana 的**多租户隔离机制**，允许不同团队在同一 Kibana 实例中拥有独立的工作空间。它隔离的是**Kibana 层的展示对象与功能入口**——每个 Space 有自己的一套 Dashboards、Data Views、告警规则、Lenses，彼此不可见；但要注意**Space 本身不隔离 ES 数据**：数据访问边界由角色（Role）的 index privileges 决定，Space 只决定"看得到哪些看板和功能"。两层机制配合才构成完整的租户隔离，单独依赖任何一层都有漏洞。

```text
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

技术上每个 Space 是 `.kibana` 索引里的一个 `space` 类型 Saved Object，其余对象（dashboard、search、data-view…）通过 `namespace_agnostic`/`namespaces` 字段归属到具体 Space——8.x 用 `_indices` 里的 `namespaces: ["team-a"]` 标记实现过滤，**同一个 Dashboard ID 在不同 Space 可以独立存在**。Default Space 无法删除，早期未分租户的资产全部沉淀在 Default 里，推行 Spaces 的第一步通常是把 Default 的存量对象按归属迁移（Saved Objects 里 Cut/Copy to Space）到各团队空间。

# 2. 创建 Space

Space 的 UI 创建入口在 Stack Management，核心决策只有两个：**图标/配色**（帮用户认路）和 **disabledFeatures**（裁剪功能入口）：

```text
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

> **注意**：`disabledFeatures` 是**导航级裁剪**——关掉 ML 入口后，即便用户角色里有 ML 权限也看不到菜单，但它不是安全边界，掌握 API 的用户仍可绕过 UI 调用；真正的权限收口永远在 Role 层。反过来这也意味着可以用它做"防呆"：不给业务团队展示 Dev Tools 入口，能挡掉九成误操作。

# 3. 内置角色与权限模型

Kibana RBAC 沿袭 ES 的三层结构：**Cluster privileges（集群级）+ Index privileges（数据级）+ Application privileges（Kibana 功能级）**。内置角色给出常见档位的现成组合：

| 内置角色 | 定位 | 权限概要 |
|----------|------|---------|
| `superuser` | 平台管理员 | 全部权限（含 .kibana 写） |
| `kibana_admin` | Kibana 管理员 | 管 Kibana 配置/对象/Space，不碰数据权限 |
| `kibana_system` | 服务账号 | Kibana 进程连接 ES 所用，**不能登录 UI** |
| `monitoring_user` | 只读监控 | 看 stack 监控页与相关索引 |
| `viewer` | 只读查看 | Dashboard/Discover 只读，跨全部 Space |
| `editor` | 编辑 | 可创建修改对象，但不能改管理配置 |
| `apm_*` / `ml_*` / `transform_*` | 功能专项 | 各子系统最小权限 |

实践中 `viewer`/`editor` 这类内置角色是**全 Space 生效**的，多租户场景通常需要自定义角色把权限按 Space 收窄。

## 3.1 Kibana Privilege

Application privilege 作用于 Kibana 功能层，档位从粗到细：

| 权限 | 说明 |
|------|------|
| `all` | 所有权限 |
| `read` | 只读 |
| `minimal_read` | 仅可看 Dashboard（不可 Discover） |
| `none` | 无权限 |

`minimal_read` 常被低估：它给业务方"看板投屏/自服务浏览"刚好够——能打开嵌入的 Dashboard，但**不能进 Discover 翻原始文档**，避免把数据探索面暴露给不需要它的人群。

## 3.2 创建角色

一个典型的多租户组合是"某团队的查看者"：**集群只给 monitor、索引只给本团队数据、Kibana 只限本 Space**：

```text
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

三层缺一层都会出事故：**只配 application 不配 indices**，用户在 Discover 里对着 Data View 报 403；**只配 indices 不配 application**，能查数据但没有 UI 功能；`resources` 写 `*` 等于全 Space 放开。更细的粒度还有两种：**base privilege**（`kibana_all_read` 配 `space:team-a`，控制"这个 Space 能不能进"）与 **feature privilege**（`feature_discover.read`，控制"进来后能用什么"），官方推荐用 Space 的 UI 矩阵生成、用 API 审计。

**字段级与文档级安全**（DLS/FLS）是 index privilege 的进阶能力，多租户数据隔离的杀手锏：

```json
{
  "indices": [
    {
      "names": ["orders-*"],
      "privileges": ["read"],
      "query": { "match": { "tenant_id": "team-a" } },
      "field_security": {
        "grant": ["order_no", "amount", "status", "@timestamp"],
        "except": ["customer_phone", "customer_id_card"]
      }
    }
  ]
}
```

`query` 即 **Document Level Security**——该角色所有搜索自动附加此过滤，跨租户文档在 ES 层就不可见（连 count 都不算入）；`field_security` 即 **Field Level Security**，`customer_phone` 对分析师彻底消失而非打码。**两者都受 License 限制**（DLS/FLS 属 Platinum+），且注意：**runtime field、aggregations 结果同样受 DLS 约束**，这是"用同一个索引按租户隔离"方案可行的前提。

> **设计考量**：按 Space 分租户（看板隔离、数据共用 index pattern）实现快、管理省，适合"团队边界=看板边界"；按 DLS 分租户（同一 Space 同一 Data View、行级隔离）适合 SaaS 多租户或对数据边界要求严格者。大多数企业内部可观测平台用 Space 方案就够，涉及客户数据才上 DLS/FLS。

# 4. Feature Control

UI 建角色时的 Feature controls 矩阵，本质就是生成 feature privilege 的表单，按功能精细授权：

```text
Role → Kibana → Feature controls

按功能精细控制:
  Discover: All / Read / None
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

两个易混点：**Dev Tools 的"Read"只是允许打开 Console，执行任意 API 仍需背后 ES 权限**——Console 里发出的请求以当前用户身份鉴权，所以 Dev Tools 权限的真正闸门在 cluster/index privileges；**Management 类功能（Index Management、Advanced Settings）给不给 All，决定该角色能否改平台配置**，业务角色一律 None。

# 5. 外部认证集成

Kibana 自身不维护密码库（native realm 只是兜底），企业方案是把认证**委托给外部 IdP**（Realm 概念：ES 侧定义身份源，Kibana 透传）：

| Realm | 适用 | 要点 |
|-------|------|------|
| native | 内部工具/测试 | 本地用户，`_security/user` 管理 |
| **saml** | 企业 SSO（AD FS、Okta） | metadata 交换，Kibana SP 发起，支持 role/session 属性 |
| **oidc** | 现代 IdP（Keycloak、Auth0） | Authorization Code + PKCE，最常用 |
| ldap | 传统目录 | 配合 role mapping 从 AD 组映射角色 |
| kerberos | 内网 Windows 域 | 桌面免密，常与 ldap 组合 |
| http（proxy header） | 前置网关鉴权后透传头 | 信任边界前移到网关 |

SAML 的最小配置（`kibana.yml` + ES `elasticsearch.yml` 成对出现）：

```yaml
# elasticsearch.yml
xpack.security.authc.realms.saml.saml1:
  order: 1
  idp.metadata.path: /etc/es/saml-idp-metadata.xml
  idp.entity_id: "https://idp.example.com/"
  sp.entity_id: "https://kibana.example.com/app/saml"
  sp.acs: "https://kibana.example.com/api/security/saml/acs"
  attributes.principal: "nameid:persistent"
  attributes.groups: "Role"        # IdP 断言里的组属性

# kibana.yml
xpack.security.authn.providers:
  native: { order: 0, enabled: true }   # 保留 break-glass 账号
  saml1: { order: 1, enabled: true }
```

角色不在断言里写死，而是用 **Role Mapping** 把外部组映射到 Kibana 角色，IdP 改组即权限生效：

```json
PUT /_security/role_mapping/ad_team_a
{
  "roles": ["team_a_viewer"],
  "enabled": true,
  "authentication_type": "realm",
  "rule": { "all": [
    { "field": { "realm.name": "saml1" } },
    { "field": { "groups": "cn=kibana-team-a,ou=groups,dc=example,dc=com" } }
  ] }
}
```

> **行业实践**：永远保留一个不依赖 SSO 的 native **break-glass 管理员账号**（受监控、密码托管、SSO 故障时救急）；IdP 会话超时与 `xpack.security.session.idleTimeout` 要对齐，否则出现"IdP 已登出、Kibana 还能用"的窗口期不一致。LDAP/AD 环境优先考虑"ldap realm + role mapping"，比 SAML 排障链路短得多。

# 6. 多租户方案

```text
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

三种场景可叠加（Space 按团队、角色按功能档位），但**不建议用 Space 承载合规级数据边界**——审计上要证明"隔离"时，index privileges/DLS 才是可取证的层，Space 只是展示层。

# 7. Saved Objects 跨 Space 复制

```text
Stack Management → Saved Objects
  → 选择对象 → Copy to Space
  → 选择目标 Space

场景: 在 Team-A 中创建的 Dashboard 模板
  复制到 Team-B，调整 Data View 指向 Team-B 的索引
```

Copy to Space 会连带复制依赖对象（引用的 searches、lenses），但 **Data View 复制过去仍指向原 pattern**，必须手动改指目标团队索引，否则"看起来在 Team-B、实际查 Team-A 数据"——只要后者角色碰巧有索引权限就是静默的跨租户泄露。规模化做法是把看板模板导出 NDJSON、以 CI 方式按 Space 变量化导入，而不是手工复制。

# 8. 运维常见误区

- **权限过大**：图省事给用户挂 `superuser` 或全 Space 的 `kibana_all_all`，是最常见的审计发现。收敛路径：先 `viewer` 起步，按需逐个 feature 提权；管理动作交给少数 `kibana_admin`。
- **忘了 API Key 与 Service Account**：脚本和采集器用个人账号 + 明文密码，人员离职即断链。应统一用 API Key（带角色与过期）或 service account，并定期 `GET _security/api_keys` 盘点。
- **Index pattern 用 `*-*`**：一个 `read` 角色配 `*` 索引，连 `.kibana`、安全内部索引都能读。授权列表必须显式列前缀，且用 `except` 排除敏感字段。
- **只测能登录、不测能越权**：上线前用低权限角色登录验证"看不到别人的 Space/看板/数据"才算隔离生效，越权测试要进 checklist。
- **Default Space 长尾**：旧对象留在 Default，全员可见，成为隔离的漏点；迁移完成后把 Default 的角色访问面收干。

# 9. 小结

- **两层隔离各管一段**：Space 隔离展示对象与功能入口（UI 层），Role 的 cluster/index/application 三层权限隔离能力与数据（ES 层），数据级边界的正解是 DLS/FLS 而非 Space。
- **角色 JSON 要三层齐配**：cluster 给最小（如 monitor）、indices 显式列 pattern 并加字段/文档过滤、application 用 feature privilege 按 Space 收口——resources 写 `*` 等于没隔离。
- **认证交给 IdP、授权交给 role mapping**：SAML/OIDC 接入后以"AD/IdP 组 → Kibana 角色"映射维持策略单一来源，并保留 native break-glass 账号兜底。
- **最常见的事故不是进不去而是越权可见**：默认最小权限、显式索引列表、上线前做越权测试、及时清空 Default Space，四条都做到才算治理完成。
