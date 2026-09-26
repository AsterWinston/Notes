# 1. 数据驱动与参数化

## 1.1 为什么要数据驱动
- 模拟真实用户分布与行为差异
- 避免缓存命中导致测试结果偏乐观

## 1.2 常见数据来源
- CSV/JSON 静态文件
- 环境变量或配置中心
- 动态接口返回

# 2. 数据加载与分配策略

## 2.1 init 阶段加载
适合加载静态数据，并通过 `SharedArray` 共享：

```javascript
import { SharedArray } from "k6/data";

const users = new SharedArray("users", () => JSON.parse(open("./users.json")));
```

## 2.2 数据分片与分配
- 按 VU 编号取数据分片
- 避免多个 VU 争用同一数据

```javascript
const user = users[(__VU - 1) % users.length];
```

# 3. 关联与动态提取

## 3.1 关联的意义
- 登录、下单等流程需要动态 token 或 id
- 关联错误是压测结果失真的主要原因

## 3.2 JSON 提取

```javascript
const res = http.post("https://example.com/login", payload);
const token = res.json("token");
```

## 3.3 HTML/文本提取
- 对 HTML 返回可用正则或第三方解析库
- 复杂解析建议在服务端提供 JSON API

# 4. 数据一致性与可复现

## 4.1 固定随机策略
- 使用固定随机种子或稳定数据池
- 记录数据版本，避免“数据漂移”

## 4.2 数据污染与回收
- 写入型测试需支持数据回收
- 重要业务数据建议使用隔离环境

# 5. 常见问题与最佳实践

## 5.1 数据量不足
- VU 数高于数据量会导致重复与冲突

## 5.2 动态数据失效
- token 过期或缓存失效需提前处理

## 5.3 关联链路过长
- 多级关联会放大失败率
- 必要时拆分场景或预生成数据
