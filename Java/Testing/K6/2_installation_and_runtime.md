# 1. 安装与版本管理

## 1.1 二进制安装
k6 提供多平台二进制包，解压后将可执行文件加入 PATH 即可使用。建议统一使用固定版本，避免团队成员因版本差异导致结果不一致。

```bash
k6 version
```

## 1.2 包管理器安装
常见的安装方式包括系统包管理器或企业内部镜像。示例命令以实际平台为准：

```bash
# macOS (Homebrew)
brew install k6

# Windows (Chocolatey)
choco install k6
```

## 1.3 Docker 方式
适用于 CI/CD 或无本地环境场景，避免依赖污染：

```bash
docker run --rm -i grafana/k6 run - < script.js
```

## 1.4 版本选择与升级策略
- 生产压测建议固定版本，升级前先在低风险环境回归
- 关注变更日志，尤其是执行器、指标与输出格式的变化

# 2. 运行时基础

## 2.1 运行命令与参数
最常用的运行方式：

```bash
k6 run script.js
```

常见参数示例：

```bash
k6 run --vus 50 --duration 2m script.js
k6 run --iterations 1000 --vus 20 script.js
```

## 2.2 配置来源与管理策略
k6 支持多种配置来源：脚本 `options`、命令行参数、环境变量、配置文件。建议在脚本或配置文件中管理默认值，命令行用于临时覆盖，避免“参数漂移”。

## 2.3 环境变量与机密
通过环境变量注入运行参数或敏感信息：

```bash
K6_VUS=20 K6_DURATION=30s k6 run script.js
K6_OUT=csv=output.csv k6 run script.js
```

在脚本中读取：

```javascript
const baseUrl = __ENV.BASE_URL || "https://example.com";
```

## 2.4 日志与输出
- 控制日志级别，避免高频打印拖慢执行
- 结果输出应统一归档，便于趋势分析

# 3. 基础命令与运行示例

## 3.1 典型运行方式

```bash
k6 run --vus 10 --duration 30s script.js
k6 run --summary-export summary.json script.js
```

## 3.2 配置文件示例

```bash
k6 run --config config.json script.js
```

```json
{
  "vus": 10,
  "duration": "30s",
  "summaryTrendStats": ["avg", "p(95)", "p(99)"]
}
```

# 4. 运行资源与限制

## 4.1 CPU 与内存估算
- VU 数增大将明显提升 CPU 与内存占用
- 长时间测试应预留足够内存，避免 GC 抖动

## 4.2 连接数与系统限制
- Linux 需要关注文件描述符上限（`ulimit -n`）
- 高并发请求会导致端口耗尽或连接排队

## 4.3 网络与代理
- 内网测试需确认 DNS、代理与防火墙策略
- 可通过环境变量配置 HTTP/HTTPS 代理

# 5. 常见问题与排查

## 5.1 启动失败
- PATH 未配置或权限不足
- 脚本语法错误导致解析失败

## 5.2 HTTPS/TLS 问题
- 证书链不完整或使用自签名证书
- 必要时通过选项开启跳过验证（仅限测试环境）

## 5.3 时间与时区
- 脚本中的时间处理建议统一使用 UTC 或明确时区
- 跨机执行时确保系统时间同步
