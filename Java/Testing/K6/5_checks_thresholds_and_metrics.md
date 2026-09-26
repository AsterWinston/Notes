# 1. 指标体系概览

## 1.1 指标类型
k6 内置四类指标：
- Counter：计数器，单调递增
- Gauge：瞬时值，可增可减
- Rate：比率（0-1）
- Trend：分布统计（平均、P95、P99）

## 1.2 常见内置指标
- `http_req_duration`：请求总耗时
- `http_req_failed`：请求失败率
- `http_req_connecting`：连接建立耗时
- `http_req_tls_handshaking`：TLS 握手耗时
- `http_req_waiting`：等待首字节（TTFB）
- `iterations`：迭代次数
- `iteration_duration`：一次迭代耗时
- `vus` / `vus_max`：实时/最大 VU 数
- `data_sent` / `data_received`：流量统计

## 1.3 指标采样与统计
- Trend 指标默认输出均值、P90、P95、P99 等统计
- 高并发场景应关注分位数而非平均值

# 2. Checks 机制

## 2.1 check 用法
`check` 用于业务断言，失败会计入 `checks` 指标：

```javascript
import { check } from "k6";

check(res, {
  "status is 200": (r) => r.status === 200,
  "latency < 300ms": (r) => r.timings.duration < 300
});
```

## 2.2 与错误处理的关系
- check 失败不终止迭代
- 业务失败要与系统错误区分，避免误判稳定性

# 3. Tags 与 Groups

## 3.1 tags 的作用
- 细分指标维度（接口、场景、版本）
- 支持在阈值中针对标签过滤

```javascript
http.get("https://example.com/api", { tags: { endpoint: "list" } });
```

## 3.2 group 的作用
- 将一段业务流程聚合为分组
- 便于统计整体流程性能

```javascript
import { group } from "k6";

group("checkout", () => {
  http.get("https://example.com/cart");
  http.post("https://example.com/order", { id: 1 });
});
```

# 4. 阈值（Thresholds）

## 4.1 语法与示例
阈值是性能质量闸门，常用于 CI 失败判定：

```javascript
export const options = {
  thresholds: {
    http_req_duration: ["p(95)<300"],
    http_req_failed: ["rate<0.01"]
  }
};
```

## 4.2 设计原则
- 先有基线再设阈值
- 关键链路的阈值应更严格
- 分位数指标要结合用户体验与业务目标

# 5. 自定义指标

## 5.1 自定义指标类型

```javascript
import { Trend, Rate, Counter, Gauge } from "k6/metrics";

const apiLatency = new Trend("api_latency");
const businessFail = new Rate("business_fail");
const bizCount = new Counter("biz_count");
const queueSize = new Gauge("queue_size");
```

## 5.2 采集与性能影响
- 自定义指标会带来一定开销
- 高频采样应控制粒度，避免影响压测精度

# 6. 结果输出与分析

## 6.1 summary 输出
- 关键指标应归档，便于趋势分析
- 与版本、代码变更做关联

## 6.2 指标解读方法
- 延迟上升且吞吐下降：服务端瓶颈或限流
- `http_req_blocked` 升高：客户端连接/资源不足
- `http_req_waiting` 变长：服务端处理变慢
