# 1.NetworkPolicy 概述

## 1.1 为什么需要 NetworkPolicy

Kubernetes 的默认网络模型有一个重要特征：**所有 Pod 之间可以自由通信**（Pod-to-Pod），无论它们属于哪个 Namespace。这种"扁平网络"模型简化了网络设计，但也带来了严重的安全隐患：

- 前端 Pod 可以直接访问数据库 Pod（绕过 API 层的访问控制）
- 一个被攻破的应用 Pod 可以扫描和攻击集群中所有其他 Pod
- 不同 Namespace（如 dev 和 prod）之间默认可以互相访问
- 任何 Pod 可以访问控制平面组件（API Server、etcd 等）

NetworkPolicy 是 Kubernetes 内置的"防火墙"，允许你定义 Pod 之间、Pod 与外部之间的网络通信规则。它基于 Pod Selector、Namespace Selector 和 IP/CIDR 来精确定义哪些流量被允许。

### 默认开放 vs 默认拒绝

Kubernetes 的默认策略是"全部放行"（allow-all）。一旦你创建了第一条 NetworkPolicy，未被该策略明确允许的流量就可能被拒绝（取决于 CNI 插件的实现和策略的具体规则）。

```
无 NetworkPolicy:
┌─────────┐    ┌─────────┐
│  Pod A  │◄──►│  Pod B  │    所有流量自由通行
└─────────┘    └─────────┘

创建 NetworkPolicy 后:
┌─────────┐         ┌─────────┐
│  Pod A  │──allowed─►│  Pod B  │   明确允许的流量
└─────────┘         └─────────┘
     │
     │ blocked
     ▼
  ┌─────────┐
  │  Pod C  │                         被拒绝
  └─────────┘
```

## 1.2 NetworkPolicy 的前置条件

NetworkPolicy 只是一个"规则对象"，它**需要 CNI 网络插件来实现**。并非所有 CNI 插件都支持 NetworkPolicy：

| CNI 插件 | NetworkPolicy 支持 | 备注 |
|----------|-------------------|------|
| **Calico** | 完整支持 | Ingress + Egress，还支持额外的高级策略 |
| **Cilium** | 完整支持 | L3/L4 + L7 (HTTP/gRPC/Kafka) 策略 |
| **Weave Net** | 支持 | 支持 Ingress/Egress |
| **Kube-router** | 支持 | 使用 iptables 实现 |
| **Flannel** | 不支持 | 需要配合 Canal（Calico 策略 + Flannel 网络） |
| **Antrea** | 支持 | OVS 实现 |

如果使用 Flannel 作为 CNI 却创建了 NetworkPolicy，策略**不会生效**——所有流量仍然自由通行。这是初学者常见的困惑之一。

## 1.3 NetworkPolicy 与 Service/Ingress 的关系

常常被混淆的概念：

| 概念 | 作用 | 资源 |
|------|------|------|
| **Service** | 提供稳定的访问端点 + 四层负载均衡 | Service |
| **Ingress** | 七层 HTTP 路由、TLS 终结、域名/路径路由 | Ingress |
| **NetworkPolicy** | Pod 间网络防火墙规则 | NetworkPolicy |

三者互补：
- Service 定义"怎么找到后端"（服务发现）
- Ingress 定义"外部流量如何进入"（入口路由）
- NetworkPolicy 定义"谁可以访问谁"（安全隔离）

# 2.NetworkPolicy 资源详解

## 2.1 核心字段

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow
  namespace: production
spec:
  podSelector:                    # 策略应用于哪些 Pod（必填）
    matchLabels:
      app: api-server
  policyTypes:                    # 策略类型（Ingress/Egress）
  - Ingress
  - Egress
  ingress:                        # 入站规则
  - from:
    - podSelector:                # 允许来自哪些 Pod
        matchLabels:
          app: frontend
    - namespaceSelector:          # 允许来自哪些 Namespace
        matchLabels:
          team: backend
    - ipBlock:                    # 允许来自哪些 IP/CIDR
        cidr: 10.0.0.0/8
        except:                   # CIDR 范围内的排除
        - 10.0.1.0/24
    ports:                        # 允许访问哪些端口
    - protocol: TCP
      port: 8080
  egress:                         # 出站规则
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
```

## 2.2 podSelector：策略目标

`podSelector` 定义了策略影响的 Pod 集合。它使用与 Service 相同的 Label Selector 机制：

```yaml
spec:
  podSelector: {}                  # 空选择器 = 选择 Namespace 内所有 Pod
  # 或指定标签
  podSelector:
    matchLabels:
      app: api-server
  # 或使用表达式
  podSelector:
    matchExpressions:
    - key: tier
      operator: In
      values: [frontend, backend]
```

## 2.3 Ingress 入站规则

Ingress 规则定义了"哪些来源可以访问目标 Pod"（注意：这是 NetworkPolicy 的 Ingress，不是 Ingress 资源）：

```yaml
ingress:
# 规则一：允许来自特定 Pod 的流量
- from:
  - podSelector:
      matchLabels:
        app: frontend
  ports:
  - protocol: TCP
    port: 8080

# 规则二：允许来自特定 Namespace 中所有 Pod 的流量
- from:
  - namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: monitoring
  ports:
  - protocol: TCP
    port: 9090

# 规则三：允许来自外部 IP 段的流量
- from:
  - ipBlock:
      cidr: 172.17.0.0/16
      except:
      - 172.17.1.0/24
```

### from 的组合逻辑

`from` 数组内的多个条目是 **OR** 关系：
```yaml
- from:
  - podSelector: {app: frontend}      # 来自 frontend Pod
  - namespaceSelector: {env: test}    # 来自 test Namespace
  # 逻辑: frontend Pod OR test Namespace 中的任意 Pod
```

`from` 下的单个条目内的多个选择器是 **AND** 关系：
```yaml
- from:
  - podSelector:                       # 此条目
      matchLabels:                     # "来自 monitoring Namespace 中
        app: prometheus                #  标签为 app=prometheus 的 Pod"
    namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: monitoring
  # 逻辑: Namespace=monitoring AND Pod标签 app=prometheus
```

## 2.4 Egress 出站规则

Egress 规则定义了"目标 Pod 可以访问哪些外部目标"：

```yaml
egress:
# 允许访问 DNS
- to:
  - namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: kube-system
    podSelector:
      matchLabels:
        k8s-app: kube-dns
  ports:
  - protocol: UDP
    port: 53
  - protocol: TCP
    port: 53

# 允许访问数据库 Pod
- to:
  - podSelector:
      matchLabels:
        app: postgres-db
  ports:
  - protocol: TCP
    port: 5432

# 允许访问外网
- to:
  - ipBlock:
      cidr: 0.0.0.0/0
      except:
      - 10.0.0.0/8          # 排除内部网络
      - 172.16.0.0/12
  ports:
  - protocol: TCP
    port: 443
```

### 常见的 Egress 规则

默认情况下，如果不定义 Egress 规则，Pod 的出站流量不受限制。但一旦定义了任何 Egress 规则，所有出站流量都会被默认拒绝（需要显式允许）。

**最被忽视的 Egress 规则：DNS**。如果设置了 Egress 但忘记允许 DNS 流量（UDP 53），Pod 将无法解析任何域名。

## 2.5 policyTypes

`policyTypes` 定义了哪些方向的规则生效：

```yaml
spec:
  policyTypes:
  - Ingress       # 只应用 ingress 规则，egress 不受影响
  - Egress        # 只应用 egress 规则，ingress 不受影响
```

- 如果不指定，默认为 Ingress（已有的 Ingress 规则生效）
- 如果定义了 Egress 规则但 `policyTypes` 不包含 `Egress`，Egress 规则不生效

# 3.实战：常见的 NetworkPolicy 模式

## 3.1 模式一：全拒绝（Default Deny）

最安全的基础策略——先拒绝所有流量，再逐步放开。

```yaml
# 拒绝所有入站流量
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: production
spec:
  podSelector: {}                # 所有 Pod
  policyTypes:
  - Ingress
  # 不定义 ingress 规则 = 拒绝所有入站
```

```yaml
# 拒绝所有出站流量
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
```

注意：创建 deny-all-egress 后，Pod 连 DNS 都无法解析，会导致所有外部通信失败。建议配合 DNS 放行规则使用。

## 3.2 模式二：全放行（Default Allow）

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - {}                          # 空规则 = 允许所有入站
  egress:
  - {}                          # 空规则 = 允许所有出站
```

## 3.3 模式三：三层应用隔离

```yaml
# 前端层：只允许 Ingress Controller 访问
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
      podSelector:
        matchLabels:
          app.kubernetes.io/component: controller
    ports:
    - protocol: TCP
      port: 80

---
# API 层：只允许前端层访问
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080

---
# 数据库层：只允许 API 层访问
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 5432
```

## 3.4 模式四：命名空间级租户隔离

```yaml
# 允许同一个 Namespace 内的 Pod 互访，拒绝其他 Namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
  namespace: team-a
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}    # 同一 Namespace 内所有 Pod
---
# 同时需要允许监控 Namespace 的 Prometheus 抓取指标
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring
  namespace: team-a
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
      podSelector:
        matchLabels:
          app: prometheus
    ports:
    - protocol: TCP
      port: 8080          # metrics 端口
```

## 3.5 模式五：允许 DNS 的最小 Egress 规则

```yaml
# 配合 Default Deny Egress 使用
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
  # 允许访问特定外部 API
  - to:
    - ipBlock:
        cidr: 203.0.113.0/24
    ports:
    - protocol: TCP
      port: 443
```

# 4.高级特性与最佳实践

## 4.1 策略的叠加行为

同一组 Pod 可以被多个 NetworkPolicy 选中，策略的效果是**叠加（加法）**的——只要有一条策略允许，流量就会被放行。

```yaml
# 策略 A：允许 frontend Pod 访问
- podSelector: {tier: api}
  ingress:
  - from: [{podSelector: {tier: frontend}}]

# 策略 B：允许 monitoring Pod 访问
- podSelector: {tier: api}
  ingress:
  - from: [{podSelector: {app: prometheus}}]

# 结果：tier=api 的 Pod 可以被 frontend 和 prometheus 访问（两者都允许）
```

这种叠加行为使得你可以用模块化的方式构建安全策略——每个策略负责一个关注面。

## 4.2 日志与审计

NetworkPolicy 本身不提供流量日志。如果需要审计网络流量，需要 CNI 插件的支持：

- **Calico**：`calicoctl node diags`、Calico Enterprise 的流量日志
- **Cilium**：Hubble 可观测性平台，提供流日志和 L7 协议可见性
- **Antrea**：`antctl` 的流量日志功能

```bash
# Cilium Hubble 查看实时流日志
hubble observe --namespace production

# Calico 查看网络策略日志
calicoctl get networkpolicy -o wide
```

## 4.3 调试 NetworkPolicy

```bash
# 查看命名空间中所有 NetworkPolicy
kubectl get networkpolicy -n production

# 查看策略详情（重点关注哪些 Pod 被选中）
kubectl describe networkpolicy api-policy -n production

# 测试连通性（从一个 Pod 访问另一个 Pod）
kubectl exec -it frontend-pod -n production -- curl http://api-pod-ip:8080

# 使用 netshoot 镜像进行全面网络调试
kubectl run debug --rm -it --image=nicolaka/netshoot -- /bin/bash
# 容器内使用: curl, ping, dig, nslookup, nc, nmap, tcpdump
```

### 常见陷阱

1. **忘记了 DNS Egress 规则**：设置了 default-deny-egress 后忘记放行 DNS（UDP 53）
2. **混淆了 AND/OR 逻辑**：from 的多个条目是 OR，单个条目内的多个选择器是 AND
3. **CNI 不支持**：Flannel 默认不支持 NetworkPolicy，但没有任何错误提示
4. **选择器匹配了错误的 Pod**：空 `podSelector: {}` 会选中 Namespace 内所有 Pod
5. **端口协议错误**：DNS 查询使用 UDP，但策略只开了 TCP

## 4.4 生产环境推荐策略

```
推荐的 NetworkPolicy 配置层级：

层级 1: 全局基线
├── 每个 Namespace 设置 default-deny-ingress
└── 核心 Namespace (kube-system, monitoring) 设置适当豁免

层级 2: DNS 和基础服务
├── 允许所有 Pod 访问 CoreDNS (UDP 53 + TCP 53)
└── 允许 Prometheus 访问所有 Pod 的 metrics 端口

层级 3: 应用层策略
├── 前端层 → API 层
├── API 层 → 数据库层
├── API 层 → 缓存层
└── API 层 → 消息队列层

层级 4: 命名空间隔离
├── 默认拒绝跨 Namespace 流量
└── 通过显式 namespaceSelector 允许特定跨 Namespace 访问
```
