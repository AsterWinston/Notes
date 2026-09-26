# 1. 可观测性集成目标

## 1.1 为什么要集成监控
- 仅看 k6 指标无法定位服务端瓶颈
- 需要关联应用、数据库、系统指标

## 1.2 典型集成路径
- k6 指标输出到 InfluxDB 或 Prometheus
- Grafana 统一展示

# 2. InfluxDB 集成

## 2.1 基本配置

```bash
K6_OUT=influxdb=http://localhost:8086/k6 k6 run script.js
```

## 2.2 使用建议
- 控制标签维度，避免基数爆炸
- 为不同测试添加 run_id 标签

# 3. Prometheus Remote Write

## 3.1 基本配置

```bash
K6_OUT=prometheus-remote-write=http://localhost:9090/api/v1/write k6 run script.js
```

## 3.2 注意事项
- 关注写入延迟与采样频率
- 保持指标命名与标签一致

# 4. Grafana 展示

## 4.1 面板设计
- 以业务流程为主线
- 同屏展示延迟、吞吐、错误率与系统资源

## 4.2 关联分析
- 将应用日志、APM 与 k6 指标对齐
- 观察峰值时刻的系统指标变化

# 5. 常见问题

## 5.1 指标丢失
- 检查写入端点与网络
- 关注 k6 的输出错误日志

## 5.2 标签基数过高
- 过多标签会导致存储膨胀
- 对关键维度做白名单控制
