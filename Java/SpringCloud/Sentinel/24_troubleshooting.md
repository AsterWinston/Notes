
# 1. 常见问题速查

| 现象 | 可能原因 | 解决步骤 |
|------|---------|---------|
| 规则不生效 | 资源未触发调用 | 先请求一次接口，Sentinel 懒加载 |
| Dashboard 无数据 | 客户端未连接 | 检查 `transport.port` 和网络 |
| 限流阈值不准确 | 统计窗口不一致 | 确认 `sampleCount` 和 `intervalInMs` |
| 内存占用高 | 规则过多/资源过多 | 减少规则、清理无用资源 |
| 集群流控失败 | Token Server 不可达 | 检查连接和超时配置 |

# 2. 规则不生效排查

```bash
# 1. 确认规则是否加载
curl http://localhost:8719/getRules?type=flow

# 2. 查看资源是否被调用
curl http://localhost:8719/metrics

# 3. 查看拦截日志
tail -f ~/logs/csp/sentinel-record.log.*

# 4. 查看 Dashboard 日志
tail -f ~/logs/csp/sentinel-dashboard.log
```

```java
// 编程验证规则
@RestController
public class ProbeController {

    @GetMapping("/sentinel/probe/rules")
    public Map<String, List<?>> probeRules() {
        Map<String, List<?>> rules = new HashMap<>();
        rules.put("flow", FlowRuleManager.getRules());
        rules.put("degrade", DegradeRuleManager.getRules());
        rules.put("system", SystemRuleManager.getRules());
        return rules;
    }
}
```

# 3. Dashboard 无数据

```bash
# 1. 检查客户端是否启动
curl http://localhost:8719/health
# 返回 "success" → 客户端正常

# 2. 检查 Dashboard 连接
curl http://localhost:8080/registry/machine
# 应返回已注册的机器列表

# 3. 检查网络连通性
telnet sentinel-dashboard 8080

# 4. 检查客户端配置
# spring.cloud.sentinel.transport.dashboard
# spring.cloud.sentinel.transport.port
```

**检查清单**：

- [ ] `spring.cloud.sentinel.transport.dashboard` 地址是否可通
- [ ] `spring.cloud.sentinel.transport.port` 端口是否被占用
- [ ] 防火墙是否放通了 Dashboard 端口和 Client 端口
- [ ] 客户端 IP 是否被 Dashboard 识别（检查 `client-ip` 配置）

# 4. 集群流控异常

```bash
# 1. 检查 Token Server 状态
curl http://token-server:18730/health

# 2. 检查 Client 是否连接 Token Server
curl http://localhost:8719/cluster/client/info

# 3. 检查 Token Server 日志
tail -f ~/logs/csp/token-server.log
```

常见集群问题：

| 问题 | 解决 |
|------|------|
| Token Server 不可达 | 检查网络和端口 |
| Token 请求超时 | 调大 `requestTimeout` |
| 双 Token Server 脑裂 | 检查 `cluster.server` 配置唯一性 |
| Client 不认领 Token | 重启 Client 触发重新注册 |

# 5. 内存过高

```bash
# 1. 查看 Sentinel 内存主要消耗来源
jmap -histo:live <pid> | grep sentinel

# 2. 常见大对象
# ClusterNodeManager → 所有资源的 ClusterNode
# StatisticNode → 每个资源的秒/分钟窗口
# MetricBucket → 每个窗口的指标数据
```

**优化建议**：

```java
// 限制统计的资源数量
SentinelConfig.setConfig("csp.sentinel.statistic.max.rt", "5000");

// 清理废弃资源的统计
ClusterNodeManager.clear();

// 关闭分钟级统计（仅保留秒级）
// 需自定义 StatisticNode 或修改社区代码
```

# 6. 常见异常码

| 异常 | 含义 | 处理 |
|------|------|------|
| `FlowException` | 流控拒绝 | 等待或提升阈值 |
| `DegradeException` | 熔断拒绝 | 等待恢复或人工介入 |
| `SystemBlockException` | 系统保护 | 降低流量或扩容 |
| `AuthorityException` | 授权拒绝 | 检查来源应用配置 |
| `ParamFlowException` | 热点参数限流 | 检查参数例外项 |

# 7. 诊断命令

## 7.1 QOS 命令

Sentinel Client 启动的 HTTP Server 支持以下命令：

```bash
# 查看所有资源统计
curl http://localhost:8719/metrics

# 查看特定资源的统计
curl http://localhost:8719/metrics?resourceName=getUser

# 查看所有规则
curl http://localhost:8719/getRules?type=flow

# 查看调用树
curl http://localhost:8719/tree

# 查看系统状态
curl http://localhost:8719/system

# 健康检查
curl http://localhost:8719/health
```

## 7.2 JMX 诊断

```bash
# 通过 JMX 查看 Sentinel MBean
# 使用 JConsole 连接 JMX 端口
jconsole <pid>

# Sentinel MBean 路径:
# com.alibaba.csp.sentinel:type=FlowRuleManager
# com.alibaba.csp.sentinel:type=DegradeRuleManager
# com.alibaba.csp.sentinel:type=ClusterNode
```

# 8. 快速自检清单

```
□ 1. curl localhost:8719/health → success?
□ 2. Dashboard 能看到应用?
□ 3. 规则已加载? curl localhost:8719/getRules?type=flow
□ 4. 资源被调用后规则生效?
□ 5. 拦截日志有记录? tail ~/logs/csp/sentinel-record.log
□ 6. Nacos 中有规则配置? 访问 Nacos Console 确认
□ 7. 网络双向可达? Client ↔ Dashboard, Client ↔ Nacos
□ 8. 时间同步? 所有节点时间误差 < 5s
□ 9. 版本兼容? Sentinel Client 和 Dashboard 版本一致
□ 10. 端口冲突? netstat -an | grep 8719
```
