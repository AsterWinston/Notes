# 1. 脚本结构与模块

## 1.1 最小脚本结构
k6 脚本本质是一个 JS 模块，最少包含一个 `default` 函数：

```javascript
import http from "k6/http";

export default function () {
  http.get("https://example.com");
}
```

## 1.2 options 与脚本配置
脚本内通过 `options` 定义默认运行参数：

```javascript
export const options = {
  vus: 10,
  duration: "30s",
  thresholds: { http_req_failed: ["rate<0.01"] }
};
```

## 1.3 常用内置变量
- `__VU`：当前虚拟用户编号
- `__ITER`：当前迭代次数
- `__ENV`：环境变量读取入口

# 2. 生命周期模型

## 2.1 init 阶段
- 模块加载与全局变量初始化
- 适合加载配置、构造常量、读取静态数据
- 不建议进行网络 I/O 或高成本操作

## 2.2 setup 阶段
- 在测试开始前执行一次
- 常用于准备测试数据、获取全局 token

## 2.3 VU 执行阶段
- `default` 函数由每个 VU 循环执行
- 通过 `sleep` 模拟思考时间

## 2.4 teardown 阶段
- 测试结束后执行一次
- 适合清理测试数据、输出最终统计

```javascript
import http from "k6/http";
import { sleep } from "k6";

export const options = { vus: 5, duration: "10s" };

export function setup() {
  const res = http.post("https://example.com/login", { user: "u", pass: "p" });
  return { token: res.json("token") };
}

export default function (data) {
  const headers = { Authorization: `Bearer ${data.token}` };
  http.get("https://example.com/api", { headers });
  sleep(1);
}

export function teardown(data) {
  http.post("https://example.com/logout", null, { headers: { Authorization: `Bearer ${data.token}` } });
}
```

# 3. 数据共享与状态

## 3.1 每 VU 隔离
- 每个 VU 拥有独立的 JS 运行时与变量空间
- 共享状态会导致竞态，需避免全局可变状态

## 3.2 SharedArray 使用
适合加载大型静态数据（如用户池）：

```javascript
import { SharedArray } from "k6/data";

const users = new SharedArray("users", () => JSON.parse(open("./users.json")));
```

## 3.3 随机与可复现
- 随机数据应可控，方便复现问题
- 建议固定随机种子或记录数据生成策略

# 4. 错误处理与超时

## 4.1 fail 与异常
- `fail` 可立即终止当前 VU 执行
- 对预期内的失败使用 `check` 标记，而非直接抛异常

```javascript
import { fail } from "k6";

if (!condition) {
  fail("precondition failed");
}
```

## 4.2 请求超时
- 单次请求可设置 `timeout`
- 超时要结合业务 SLO 与网络环境

## 4.3 失败策略
- 区分业务失败与系统失败
- 失败达到阈值应终止测试并给出结论

# 5. 常见坑与最佳实践

## 5.1 在 init 中做 I/O
- init 阶段做大量网络调用会放大启动成本
- 需要动态数据时应放到 `setup`

## 5.2 大对象与内存压力
- 避免把响应体大量保存在内存中
- 对不需要的响应设置 `discardResponseBodies`

## 5.3 过多日志
- 日志打印会显著影响吞吐
- 只在调试阶段开启详细日志
