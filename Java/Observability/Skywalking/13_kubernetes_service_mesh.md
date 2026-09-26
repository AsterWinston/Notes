
# 1. K8s 部署

## 1.1 OAP + UI Helm 部署

```bash
helm repo add skywalking https://apache.jfrog.io/artifactory/skywalking-helm
helm install skywalking skywalking/skywalking \
  --set oap.image.tag=9.7.0 \
  --set ui.image.tag=9.7.0 \
  --set oap.storageType=elasticsearch \
  --set elasticsearch.host=es-cluster:9200
```

## 1.2 Sidecar Agent

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
        - name: app
          image: my-app:latest
          # ...
        - name: skywalking-agent
          image: apache/skywalking-java-agent:9.7.0
          volumeMounts:
            - name: agent
              mountPath: /skywalking/agent
      initContainers:
        - name: init-agent
          image: apache/skywalking-java-agent:9.7.0
          command: ['cp', '-r', '/skywalking/agent', '/target/']
          volumeMounts:
            - name: agent
              mountPath: /target
      volumes:
        - name: agent
          emptyDir: {}
```

# 2. k8s-event-exporter

```yaml
# 部署 k8s-event-exporter 收集 K8s 事件
apiVersion: apps/v1
kind: Deployment
metadata:
  name: skywalking-k8s-event-exporter
spec:
  template:
    spec:
      serviceAccountName: skywalking
      containers:
        - name: exporter
          image: apache/skywalking-kubernetes-event-exporter:latest
          env:
            - name: OAP_ADDRESS
              value: skywalking-oap:11800
            - name: CLUSTER_NAME
              value: prod-cluster
```

```
K8s Events → Event Exporter → OAP Server → UI 展示
  Pod OOMKilled
  Node NotReady
  Deployment Scaled
```

# 3. Service Mesh 集成（Istio）

```
┌────────────────────────────────────────────┐
│               Service Mesh                  │
│                                             │
│  ┌──────────┐         ┌──────────┐        │
│  │  Envoy   │ ←────→ │  Envoy   │        │
│  │ (Sidecar)│  mTLS   │ (Sidecar)│        │
│  └────┬─────┘         └────┬─────┘        │
│       │ ALS                 │ ALS          │
│       ▼                     ▼              │
│  ┌──────────────────────────────────┐     │
│  │      SkyWalking OAP (ALS 接收)    │     │
│  └──────────────────────────────────┘     │
└────────────────────────────────────────────┘
```

## 3.1 Envoy ALS 配置

```yaml
# Istio MeshConfig
meshConfig:
  defaultConfig:
    envoyAccessLogService:
      address: skywalking-oap:11800
  enableEnvoyAccessLogService: true
```

# 4. SkyWalking Satellite

Satellite 是轻量级 Sidecar 代理，用于减少 OAP 压力：

```
Agent → Satellite (聚合/采样/过滤) → OAP Server
```

```yaml
# satellite-config.yaml
pipeline:
  - name: trace
    type: trace
    filter:
      - type: sampling
        rate: 1000    # 每秒最多 1000 条
    sender:
      - type: grpc
        server: oap:11800
```

# 5. 容器环境 JVM 配置

```bash
# K8s 环境中的 Java Agent 参数
java -javaagent:/skywalking/agent/skywalking-agent.jar \
     -DSW_AGENT_NAME=${POD_NAME} \
     -DSW_AGENT_COLLECTOR_BACKEND_SERVICES=skywalking-oap:11800 \
     -DSW_AGENT_INSTANCE_NAME=${POD_IP} \
     -jar app.jar
```

# 6. 最佳实践

1. **InitContainer 拷贝 Agent**：避免在应用镜像中打包 Agent
2. **POD_NAME 做实例名**：便于区分同服务不同 Pod
3. **Satellite 汇聚**：大规模集群使用 Satellite 降低 OAP 压力
4. **Service Mesh ALS**：Mesh 场景优先使用 ALS，无需注入 Java Agent
