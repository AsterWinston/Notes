# 1. 输出机制概览

## 1.1 内置输出
k6 支持多种输出方式，包括标准输出与外部存储：
- 标准控制台 summary
- JSON/CSV 文件
- InfluxDB、Prometheus Remote Write 等

## 1.2 输出场景
- 本地调试：控制台输出即可
- CI 归档：JSON/CSV 或 summary 导出
- 可视化：对接监控系统

# 2. Summary 与自定义报告

## 2.1 summary 导出

```bash
k6 run --summary-export summary.json script.js
```

## 2.2 自定义 summary

```javascript
export function handleSummary(data) {
  return {
    "summary.json": JSON.stringify(data),
    stdout: "summary generated"
  };
}
```

# 3. 输出指标选择

## 3.1 关键指标优先级
- 延迟分位数与错误率优先
- 结合业务指标，避免只看吞吐

## 3.2 标签化输出
- 通过 tags 分维度聚合
- 便于区分不同接口或场景

# 4. 结果解释与呈现

## 4.1 指标与结论
- 结果必须与阈值绑定
- 输出应包含“是否达标”的明确判断

## 4.2 可视化建议
- Grafana 面板按业务流程组织
- 关注延迟与错误率的趋势变化

# 5. 常见问题

## 5.1 输出过大
- 控制输出频率和维度

## 5.2 输出格式不一致
- 统一输出规范便于归档与分析
