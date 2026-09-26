
# 1. 概述

Jenkins 与 Kubernetes 的集成是现代 DevOps 实践的核心组合。Kubernetes Plugin 允许 Jenkins 在 Kubernetes 集群上动态创建 Pod 作为 Agent，实现 CI/CD 环境的弹性伸缩和高可用。

核心价值：
- **弹性伸缩**：按需创建 Agent Pod，用完即销毁
- **资源高效**：Pod 可设置资源限制，避免资源浪费
- **环境一致性**：每次构建使用相同的镜像环境
- **隔离性**：每个构建运行在独立的 Pod 中

# 2. Kubernetes 插件安装与配置

## 2.1 安装插件

```
必需插件：
├── Kubernetes Plugin（核心）
├── Kubernetes CLI Plugin（kubectl 操作）
└── Kubernetes Credentials Plugin（凭据）

可选插件：
├── Kubernetes Pipeline Plugin（Pipeline 步骤）
└── Blue Ocean（可视化）
```

## 2.2 Kubernetes 云配置

**配置路径**：Manage Jenkins > Nodes > Clouds > Add a new cloud > Kubernetes

```yaml
# Kubernetes 云配置
Name: kubernetes-cluster
Kubernetes URL: https://kubernetes.default.svc.cluster.local
Kubernetes Namespace: jenkins
Credentials: jenkins-service-account (ServiceAccount Token)
# 或 kubeconfig
# Credentials: kubeconfig-file (Secret file)

# 连接测试
WebSocket: true  # 使用 WebSocket 连接 Pod
Connection Timeout: 30
Read Timeout: 30

# Pod 默认配置
Container Cap: 50  # 最大并发 Pod 数
Pod Retention:
  - Never: 构建完成立即删除
  - On Failure: 失败时保留（调试用）
  - Always: 始终保留（调试用）

# Jenkins URL（Pod 访问 Jenkins Master 的地址）
Jenkins URL: http://jenkins:8080  # 集群内访问地址
```

## 2.3 Service Account 配置

```yaml
# 创建 Jenkins Service Account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: jenkins
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: jenkins
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/exec", "pods/log"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: jenkins
subjects:
  - kind: ServiceAccount
    name: jenkins
    namespace: jenkins
```

# 3. Pod Template 配置

## 3.1 在 UI 中配置 Pod Template

```yaml
# Kubernetes Pod Template 配置
Pod Template:
  Name: maven-pod
  Labels: maven
  Usage: Use this node as much as possible
  Containers:
    - Name: maven
      Image: maven:3.9-eclipse-temurin-17
      Command: /bin/sh -c
      Args: cat
      Working dir: /home/jenkins/agent
      Resource Limit:
        CPU: limit 2, request 1
        Memory: limit 2048Mi, request 1024Mi
      EnvVars:
        - MAVEN_OPTS: -Xmx1024m
      Volume Mounts:
        - maven-cache: /root/.m2

  # 边车容器（sidecar）
  Containers:
    - Name: docker
      Image: docker:24-cli
      Command: /bin/sh -c
      Args: cat
      EnvVars:
        - DOCKER_HOST: tcp://localhost:2375

  Volumes:
    - Type: PersistentVolumeClaim
      Claim Name: maven-cache-pvc
      Mount Path: /root/.m2
      ReadOnly: false
    - Type: HostPath
      Host Path: /var/run/docker.sock
      Mount Path: /var/run/docker.sock

  Pod Restart Policy: Never
  Active Deadline: 600  # Pod 最大运行时间（秒）
```

## 3.2 在 Pipeline 中定义 Pod Template

```groovy
// 在 Pipeline 中动态定义 Pod Template
pipeline {
    agent {
        kubernetes {
            label 'my-pod'
            yaml '''
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: jenkins-agent
spec:
  containers:
  - name: maven
    image: maven:3.9-eclipse-temurin-17
    command:
    - sleep
    args:
    - infinity
    resources:
      requests:
        cpu: 500m
        memory: 1Gi
      limits:
        cpu: 2
        memory: 2Gi
    env:
    - name: MAVEN_OPTS
      value: "-Xmx1024m"
    volumeMounts:
    - mountPath: /root/.m2
      name: maven-cache
  - name: docker
    image: docker:24-cli
    command:
    - sleep
    args:
    - infinity
    volumeMounts:
    - mountPath: /var/run/docker.sock
      name: docker-sock
  volumes:
  - name: maven-cache
    persistentVolumeClaim:
      claimName: maven-cache-pvc
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
'''
            defaultContainer 'maven'
        }
    }
    stages {
        stage('Build') {
            steps {
                container('maven') {
                    sh 'mvn clean package'
                }
            }
        }
        stage('Docker Build') {
            steps {
                container('docker') {
                    sh 'docker build -t myapp:latest .'
                }
            }
        }
    }
}
```

## 3.3 使用 YAML 文件模板

```groovy
// 从文件读取 Pod Template
pipeline {
    agent {
        kubernetes {
            label 'k8s-agent'
            yamlFile 'ci/jenkins-pod.yaml'
            defaultContainer 'jnlp'
        }
    }
    stages {
        stage('Test') {
            steps {
                sh 'echo Running in Kubernetes Pod'
            }
        }
    }
}
```

```yaml
# ci/jenkins-pod.yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: jnlp
    image: jenkins/inbound-agent:latest-jdk17
    resources:
      requests:
        cpu: 500m
        memory: 512Mi
      limits:
        cpu: 1
        memory: 1Gi
  - name: node
    image: node:20-alpine
    command:
    - sleep
    args:
    - infinity
    resources:
      requests:
        cpu: 500m
        memory: 512Mi
  - name: docker
    image: docker:24-cli
    command:
    - sleep
    args:
    - infinity
    volumeMounts:
    - mountPath: /var/run/docker.sock
      name: docker-sock
  volumes:
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
```

# 4. Kubernetes Agent 的完整生命周期

```
1. Job 触发 → 需要匹配标签的 Agent
2. Kubernetes 插件检查缓存 → 无可用 Pod
3. 根据 Pod Template 创建 Pod
   ├── 创建 Pod 对象（API Server）
   ├── 调度到 Worker Node
   ├── 拉取容器镜像
   └── 启动所有容器
4. JNLP 容器（inbound-agent）连接 Jenkins Master
5. Jenkins Master 分配 Build 到该 Pod
6. 执行 Pipeline 步骤（可以在不同容器中执行）
7. 构建完成 → Pod 断开连接
8. 根据 Pod Retention 策略删除 Pod
```

# 5. Kubernetes Agent 的边车模式

## 5.1 多容器边车模式

```groovy
// 边车（Sidecar）容器：在 Pod 中运行辅助服务
pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: jnlp
    image: jenkins/inbound-agent:latest-jdk17
  - name: maven
    image: maven:3.9-eclipse-temurin-17
    command:
    - sleep
    args:
    - infinity
  - name: sonar-scanner
    image: sonarsource/sonar-scanner-cli:latest
    command:
    - sleep
    args:
    - infinity
'''
        }
    }
    stages {
        stage('Build') {
            steps {
                container('maven') {
                    sh 'mvn clean package'
                }
            }
        }
        stage('Code Quality') {
            steps {
                container('sonar-scanner') {
                    withSonarQubeEnv('SonarQube') {
                        sh 'sonar-scanner'
                    }
                }
            }
        }
    }
}
```

## 5.2 服务边车（数据库）

```groovy
// 在 Pod 中启动 MySQL 边车容器做集成测试
pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: jnlp
    image: jenkins/inbound-agent:latest-jdk17
  - name: maven
    image: maven:3.9-eclipse-temurin-17
    command:
    - sleep
    args:
    - infinity
  - name: mysql
    image: mysql:8.0
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: test
    - name: MYSQL_DATABASE
      value: testdb
    ports:
    - containerPort: 3306
    readinessProbe:
      exec:
        command: ["mysqladmin", "ping", "-h", "localhost"]
      initialDelaySeconds: 10
      periodSeconds: 5
'''
        }
    }
    stages {
        stage('Integration Test') {
            steps {
                container('maven') {
                    sh '''
                        # 等待 MySQL 就绪
                        until nc -z localhost 3306; do
                            echo "Waiting for MySQL..."
                            sleep 2
                        done
                        mvn verify -Pintegration-test
                    '''
                }
            }
        }
    }
}
```

# 6. 基于标签的 Pod 选择

## 6.1 节点选择器

```groovy
pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: jnlp
    image: jenkins/inbound-agent:latest-jdk17
  - name: gpu-build
    image: tensorflow/tensorflow:latest-gpu
    command:
    - sleep
    args:
    - infinity
  nodeSelector:
    cloud.google.com/gke-accelerator: nvidia-tesla-t4
'''
        }
    }
    stages {
        stage('GPU Build') {
            steps {
                container('gpu-build') {
                    sh 'python -c "import tensorflow as tf; print(tf.config.list_physical_devices())"'
                }
            }
        }
    }
}
```

## 6.2 污点和容忍

```groovy
pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: jnlp
    image: jenkins/inbound-agent:latest-jdk17
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "ci-jobs"
    effect: "NoSchedule"
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: ci-optimized
            operator: In
            values:
            - "true"
'''
        }
    }
    stages {
        stage('Build') {
            steps {
                echo 'Running on optimized node'
            }
        }
    }
}
```

# 7. kubectl 集成

## 7.1 安装 kubectl

```groovy
// 在 Pipeline 中使用 kubectl
pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: jnlp
    image: jenkins/inbound-agent:latest-jdk17
  - name: kubectl
    image: bitnami/kubectl:latest
    command:
    - sleep
    args:
    - infinity
'''
        }
    }
    stages {
        stage('Deploy') {
            steps {
                container('kubectl') {
                    withKubeConfig([credentialsId: 'kubeconfig']) {
                        sh """
                            kubectl set image deployment/myapp \
                                myapp=registry.company.com/myapp:${BUILD_NUMBER} \
                                -n production
                            kubectl rollout status deployment/myapp -n production
                        """
                    }
                }
            }
        }
    }
}
```

## 7.2 Kubernetes CLI Pipeline

```groovy
// 使用 Kubernetes CLI Plugin
stage('K8s Deploy') {
    steps {
        withKubeConfig(
            credentialsId: 'kubeconfig-prod',
            serverUrl: 'https://k8s-api.company.com',
            namespace: 'production'
        ) {
            sh 'kubectl apply -f k8s/deployment.yaml'
            sh 'kubectl rollout status deployment/myapp'
        }
    }
}
```

# 8. Helm 集成

## 8.1 Helm 部署

```groovy
pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: jnlp
    image: jenkins/inbound-agent:latest-jdk17
  - name: helm
    image: alpine/helm:3.14
    command:
    - sleep
    args:
    - infinity
    env:
    - name: HELM_EXPERIMENTAL_OCI
      value: "1"
'''
        }
    }
    stages {
        stage('Helm Deploy') {
            steps {
                container('helm') {
                    withKubeConfig([credentialsId: 'kubeconfig']) {
                        sh """
                            helm upgrade --install myapp ./chart \\
                                --namespace ${params.ENV} \\
                                --set image.tag=${BUILD_NUMBER} \\
                                --set replicaCount=${params.REPLICAS} \\
                                --wait --timeout 5m
                        """
                    }
                }
            }
        }
    }
}
```

## 8.2 Helm 测试

```groovy
stage('Helm Test') {
    steps {
        container('helm') {
            withKubeConfig([credentialsId: 'kubeconfig']) {
                sh """
                    helm test myapp --namespace ${params.ENV} \\
                        --logs --timeout 2m
                """
            }
        }
    }
}
```

# 9. Kubernetes 上的 Jenkins 高可用

## 9.1 Jenkins Master 部署

```yaml
# Jenkins Master Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: jenkins
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      serviceAccountName: jenkins
      containers:
      - name: jenkins
        image: jenkins/jenkins:lts-jdk17
        ports:
        - containerPort: 8080
          name: http
        - containerPort: 50000
          name: jnlp
        env:
        - name: JAVA_OPTS
          value: "-Xms2048m -Xmx4096m -Djenkins.install.runSetupWizard=false"
        - name: JENKINS_OPTS
          value: "--prefix=/jenkins"
        volumeMounts:
        - mountPath: /var/jenkins_home
          name: jenkins-home
        livenessProbe:
          httpGet:
            path: /login
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /login
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 5
        resources:
          requests:
            cpu: 1
            memory: 2Gi
          limits:
            cpu: 2
            memory: 4Gi
      volumes:
      - name: jenkins-home
        persistentVolumeClaim:
          claimName: jenkins-home-pvc
```

## 9.2 Service 和 Ingress

```yaml
# Service
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  namespace: jenkins
spec:
  type: ClusterIP
  ports:
  - port: 8080
    targetPort: 8080
    name: http
  - port: 50000
    targetPort: 50000
    name: jnlp
  selector:
    app: jenkins
---
# Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: jenkins
  namespace: jenkins
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - jenkins.company.com
    secretName: jenkins-tls
  rules:
  - host: jenkins.company.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: jenkins
            port:
              number: 8080
```

## 9.3 Persistent Volume

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-home-pvc
  namespace: jenkins
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
  storageClassName: standard
```

# 10. 实际案例：K8s 微服务 CI/CD Pipeline

```groovy
@Library('k8s-pipeline-lib') _

def services = ['api-gateway', 'user-service', 'order-service']

pipeline {
    agent none
    options {
        buildDiscarder(logRotator(numToKeepStr: '30'))
        timestamps()
    }
    stages {
        stage('Build All Services') {
            parallel {
                script {
                    services.each { service ->
                        stage("Build ${service}") {
                            agent {
                                kubernetes {
                                    yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: jnlp
    image: jenkins/inbound-agent:latest-jdk17
  - name: maven
    image: maven:3.9-eclipse-temurin-17
    command:
    - sleep
    args:
    - infinity
    volumeMounts:
    - mountPath: /root/.m2
      name: maven-cache
  - name: docker
    image: docker:24-cli
    command:
    - sleep
    args:
    - infinity
    volumeMounts:
    - mountPath: /var/run/docker.sock
      name: docker-sock
  volumes:
  - name: maven-cache
    persistentVolumeClaim:
      claimName: maven-cache-pvc
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
"""
                                    defaultContainer 'maven'
                                }
                            }
                            stages {
                                stage("Checkout ${service}") {
                                    steps {
                                        dir(service) {
                                            checkout scm
                                        }
                                    }
                                }
                                stage("Build ${service}") {
                                    steps {
                                        dir(service) {
                                            sh 'mvn clean package'
                                        }
                                    }
                                }
                                stage("Docker ${service}") {
                                    steps {
                                        container('docker') {
                                            dir(service) {
                                                sh """
                                                    docker build -t registry.company.com/${service}:${BUILD_NUMBER} .
                                                """
                                            }
                                        }
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
        stage('Integration Test') {
            agent {
                kubernetes {
                    yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: jnlp
    image: jenkins/inbound-agent:latest-jdk17
  - name: maven
    image: maven:3.9-eclipse-temurin-17
    command:
    - sleep
    args:
    - infinity
  - name: mysql
    image: mysql:8.0
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: test
'''
                }
            }
            steps {
                container('maven') {
                    sh './run-integration-tests.sh'
                }
            }
            post {
                always {
                    junit '**/target/failsafe-reports/*.xml'
                }
            }
        }
        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            agent {
                kubernetes {
                    yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: jnlp
    image: jenkins/inbound-agent:latest-jdk17
  - name: helm
    image: alpine/helm:3.14
    command:
    - sleep
    args:
    - infinity
'''
                }
            }
            steps {
                container('helm') {
                    withKubeConfig([credentialsId: 'kubeconfig-prod']) {
                        script {
                            services.each { service ->
                                sh """
                                    helm upgrade --install ${service} ./helm-charts/${service} \\
                                        --namespace production \\
                                        --set image.tag=${BUILD_NUMBER} \\
                                        --wait --timeout 3m
                                """
                            }
                        }
                    }
                }
            }
        }
    }
    post {
        always {
            cleanWs()
        }
        failure {
            slackSend(
                color: 'danger',
                message: "K8s Pipeline Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
            )
        }
    }
}
```
