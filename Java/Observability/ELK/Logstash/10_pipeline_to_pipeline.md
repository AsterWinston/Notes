
# 1. 概述

Pipeline-to-Pipeline 允许在同一 Logstash 进程内**高效传递数据**，避免序列化和网络开销。

```
传统: 部署多个 Logstash 实例
  Logstash-1 → (网络) → Logstash-2 → ES

Pipeline-to-Pipeline: 同一进程
  Pipeline A → (Virtual Output→Input) → Pipeline B → ES
```

# 2. 配置

```yaml
# pipelines.yml
- pipeline.id: input-pipeline
  path.config: /etc/logstash/conf.d/input.conf

- pipeline.id: processing-pipeline
  path.config: /etc/logstash/conf.d/processing.conf

- pipeline.id: output-pipeline
  path.config: /etc/logstash/conf.d/output.conf
```

```ruby
# input.conf — 采集管道
input {
  beats { port => 5044 }
}

output {
  pipeline {
    send_to => ["processing-pipeline"]   # 发送到处理管道
  }
}
```

```ruby
# processing.conf — 处理管道
input {
  pipeline {
    address => "processing-pipeline"     # 接收管道
  }
}

filter {
  grok { match => { "message" => "%{COMBINEDAPACHELOG}" } }
  geoip { source => "clientip" }
}

output {
  pipeline {
    send_to => ["output-pipeline"]
  }
}
```

```ruby
# output.conf — 输出管道
input {
  pipeline {
    address => "output-pipeline"
  }
}

output {
  elasticsearch { hosts => ["http://es:9200"] }
}
```

# 3. 分发器模式

一个管道分发到多个下游管道：

```ruby
# distributor.conf
output {
  pipeline {
    send_to => ["nginx-pipeline", "app-pipeline", "error-pipeline"]
  }
}

# 下游管道自动负载均衡
```

```ruby
# 条件分发
output {
  if [type] == "nginx" {
    pipeline { send_to => ["nginx-pipeline"] }
  } else if [type] == "app" {
    pipeline { send_to => ["app-pipeline"] }
  }
}
```

# 4. 使用场景

```
场景 1: 预处理 → 分类 → 输出
  Input Pipeline (采集) → Processing Pipeline (解析+分类)
    ├── Nginx Pipeline (Grok + GeoIP → ES)
    ├── App Pipeline (JSON + Mutate → ES)
    └── Error Pipeline (告警 + 文件)

场景 2: 隔离资源
  采集管道 (轻量) + 处理管道 (重量)
  不同优先级的数据用不同 Worker 数

场景 3: 解耦测试
  上游不变，替换下游管道进行测试
```

# 5. 性能优势

```
Pipeline-to-Pipeline vs External Queue:
  - 零序列化：内存中直接传递 Event 对象
  - 零网络开销：进程内通信
  - 低延迟：亚毫秒级传递

适用条件:
  - 同一台机器上运行的多个管道
  - 需要隔离不同类型数据处理资源
  - 需要分发数据到多个下游
```

# 6. 限制

```
- 只能在线程间传递（不支持跨进程）
- 不支持持久化（不同于 PQ）
- 需要提前定义 pipelines.yml
- 循环管道需要避免（A→B→C→A）
```
