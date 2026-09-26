# 1. 脚本工程化目标

## 1.1 可维护性
- 分层抽象：协议层、业务层、场景层
- 降低脚本重复与耦合

## 1.2 可复用性
- 公共逻辑封装为模块
- 统一请求、鉴权与断言策略

# 2. 模块化设计

## 2.1 目录组织建议

```text
k6/
  scripts/
  data/
  lib/
  scenarios/
  config/
```

## 2.2 封装请求与断言

```javascript
// lib/http_client.js
import http from "k6/http";
import { check } from "k6";

export function getJson(url, params) {
  const res = http.get(url, params);
  check(res, { "status 200": (r) => r.status === 200 });
  return res.json();
}
```

# 3. 配置管理

## 3.1 分环境配置
- 开发、测试、预发、生产使用不同配置
- 通过环境变量或配置文件切换

## 3.2 机密管理
- 不将凭证写死在脚本中
- 使用 CI 的机密管理注入

# 4. 复用与发布

## 4.1 版本化管理
- 将压测脚本纳入版本控制
- 与应用版本建立关联

## 4.2 脚本发布流程
- 提交前做 lint/format
- 关键场景附带 README 说明

# 5. 常见问题

## 5.1 模块依赖过深
- 依赖层级过深会影响维护

## 5.2 配置混乱
- 避免脚本中散落硬编码参数
