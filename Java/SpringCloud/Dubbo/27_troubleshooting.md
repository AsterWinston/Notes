
# 1. 常见问题速查

## 1.1 Provider 找不到

**现象**：
```
No provider available for the service org.example.UserService:1.0.0
```

**排查步骤**：

```bash
# 1. 检查 Provider 是否注册成功
curl http://localhost:22222/ls
# 查看是否有目标服务

# 2. 检查注册中心
# Nacos: http://localhost:8848/nacos → 服务列表 → 查找目标服务
# ZK: zkCli.sh → ls /dubbo/org.example.UserService/providers

# 3. 检查 Consumer 订阅状态
curl http://localhost:22222/ls  # Consumer QOS 端口查看

# 4. 检查网络连通性
telnet <provider-ip> <provider-port>
```

**常见原因**：

| 原因 | 解决方案 |
|------|---------|
| Provider 未启动 | 启动 Provider |
| `check=false` 但 Provider 确实挂了 | 等待自动恢复或手动启动 |
| 网络不通 | 检查防火墙/安全组 |
| version/group 不匹配 | 统一 Consumer 和 Provider 的 version/group |
| 注册中心地址错误 | 检查 `dubbo.registry.address` |
| Provider 端口冲突 | 更换端口 |
| 注册中心 namespace 不一致 | 统一 Nacos namespace |

## 1.2 序列化异常

**现象**：
```
SerializationException: Cannot deserialize class xxx
HessianProtocolException: xxx cannot be cast to xxx
```

**排查步骤**：

```bash
# 1. 检查序列化方式是否一致
# Provider 和 Consumer 的 dubbo.protocol.serialization 必须一致

# 2. 检查传输对象是否实现 Serializable
javap -verbose com.example.api.User | grep Serializable
```

**常见原因**：

| 原因 | 解决方案 |
|------|---------|
| 序列化方式不一致 | 统一序列化配置 |
| 对象不实现 Serializable | 实现 `java.io.Serializable` |
| 类版本不一致（serialVersionUID 不同） | 显式定义 `serialVersionUID` |
| Provider/Consumer 的 API jar 版本不同 | 统一 API 版本 |
| 内部类序列化问题 | 改为静态内部类 |

## 1.3 超时异常

**现象**：
```
RpcException: Invoke remote method timeout
```

**排查步骤**：

```bash
# 1. 查看方法耗时
# 在 Provider 日志/监控中查找耗时 > timeout 的调用

# 1. 检查 Provider 线程池
curl http://localhost:22222/threadpool
# 如果 active = max 且 queue 满 → 线程池瓶颈

# 1. 检查 GC
curl http://localhost:22222/jvm
# 频繁 Full GC → 导致超时
```

**常见原因**：

| 原因 | 解决方案 |
|------|---------|
| 业务处理慢 | 优化业务逻辑、加缓存 |
| 线程池满 | 增大 threads、优化队列 |
| GC 停顿 | 调优 JVM GC 参数 |
| 数据库慢查询 | 优化 SQL、加索引 |
| 下游服务超时 | 逐级排查调用链 |
| timeout 设置过短 | 区分快慢接口设置不同 timeout |

## 1.4 线程池耗尽

**现象**：
```
RejectedExecutionException: Thread pool is EXHAUSTED!
```

**排查步骤**：

```bash
# 1. 查看线程池状态
curl http://localhost:22222/threadpool

# 1. 查看线程栈
# dubbo-{timestamp}.jstack 文件会自动生成在启动目录

# 线程栈分析：
# - 大量线程在 WAITING (等待 DB/Cache 响应) → 下游慢
# - 大量线程在 RUNNABLE (CPU 计算) → 业务逻辑重
```

**解决方案**：

```yaml
# 增大线程池
dubbo:
  protocol:
    threads: 500
    queues: 200   # 设置有界队列避免 OOM

# 或改用 eager 模式（队列满时创建新线程，而非直接拒绝）
dubbo:
  protocol:
    threadpool: eager
    threads: 500
    corethreads: 100
```

# 2. 网络排查

## 2.1 TCP 连接排查

```bash
# Linux
# 查看 ESTABLISHED 连接数
netstat -an | grep 20880 | grep ESTABLISHED | wc -l

# 查看连接状态分布
ss -tan state established '( dport = :20880 or sport = :20880 )'

# Windows
netstat -an | findstr 20880
```

## 2.2 抓包分析

```bash
# tcpdump 抓包 Dubbo 协议
tcpdump -i eth0 -w dubbo.pcap tcp port 20880

# Wireshark 分析 Dubbo 协议
# 过滤: tcp.port == 20880
# 查看 Dubbo 帧的 Magic (0xdabb) 确认协议
```

## 2.3 粘包/拆包问题

```
Dubbo 帧格式:
  [2B Magic][1B Flag][1B Status][8B ID][4B Length][NB Body]

如果收到不完整帧或多帧粘连，DubboCountCodec 自动处理
问题通常源于网络层或 Netty 配置异常
```

# 3. 注册中心排查

## 3.1 检查注册状态

```bash
# Nacos
curl "http://127.0.0.1:8848/nacos/v1/ns/service/list?pageNo=1&pageSize=100"

# 查看服务实例
curl "http://127.0.0.1:8848/nacos/v1/ns/instance/list?serviceName=user-service"

# ZK
zkCli.sh -server 127.0.0.1:2181
ls /dubbo/org.example.UserService/providers
```

## 3.2 查看本地缓存

```bash
# Dubbo 本地注册中心缓存（兜底数据）
cat ~/.dubbo/dubbo-registry-{app}-{addr}.cache
```

# 4. 内存泄漏排查

## 4.1 DefaultFuture 泄漏

```java
// DefaultFuture 以 requestId 为 key 存入全局 Map
// 如果 Provider 不响应或响应超时未清理 → 内存泄漏

// 排查方法
jcmd <pid> GC.class_histogram | grep DefaultFuture
// 如果数量异常大（超过 active connections × 并发量）→ 泄漏
```

## 4.2 ThreadLocal 泄漏

```java
// RpcContext 等使用了 ThreadLocal
// 如果线程池中的线程复用时不清理 ThreadLocal → 泄漏

// 排查方法
jcmd <pid> GC.class_histogram | grep ThreadLocal
```

# 5. Dubbo QOS 排查命令

| 命令 | 说明 | 示例 |
|------|------|------|
| `ls` | 列出所有服务 | `curl http://localhost:22222/ls` |
| `online` | 上线服务 | `curl -X POST http://localhost:22222/online` |
| `offline` | 下线服务 | `curl -X POST http://localhost:22222/offline` |
| `live` | 健康检查 | `curl http://localhost:22222/live` |
| `info` | 查看应用信息 | `curl http://localhost:22222/info` |
| `threadpool` | 查看线程池 | `curl http://localhost:22222/threadpool` |
| `jvm` | 查看 JVM 指标 | `curl http://localhost:22222/jvm` |
| `registry` | 查看注册中心状态 | `curl http://localhost:22222/registry` |
| `logger` | 动态调整日志级别 | `curl -X POST http://localhost:22222/logger?level=DEBUG` |

# 6. JVM 排查

```bash
# 查看线程栈（排查死锁/阻塞）
jstack <pid> > thread.dump

# 查看堆内存（排查内存泄漏）
jmap -histo:live <pid> | head -50

# 查看 GC 情况
jstat -gcutil <pid> 1000

# 生成 heap dump
jmap -dump:format=b,file=heap.hprof <pid>
```

# 7. 调试技巧

## 7.1 开启 Dubbo 详细日志

```yaml
logging:
  level:
    org.apache.dubbo: DEBUG
    org.apache.dubbo.registry: TRACE
    org.apache.dubbo.remoting: DEBUG
```

## 7.2 直连调试

```java
// 绕过注册中心，直连 Provider 进行调试
@DubboReference(url = "dubbo://192.168.1.100:20880")
private UserService userService;
```

## 7.3 Echo 测试

```java
@DubboReference
private UserService userService;

public void echoTest() {
    // 测试 Provider 连通性（不执行业务逻辑）
    boolean available = ((EchoService) userService)
        .$echo("OK");
    System.out.println("Provider available: " + available);
}
```

## 7.4 本地存根调试

```java
// 在 Consumer 端插入调试逻辑
public class UserServiceStub implements UserService {
    private final UserService userService;

    public UserServiceStub(UserService userService) {
        this.userService = userService;
    }

    @Override
    public User getUser(Long id) {
        System.out.println("[STUB] Calling getUser with id=" + id);
        long start = System.currentTimeMillis();
        User user = userService.getUser(id);
        System.out.println("[STUB] Cost: " + (System.currentTimeMillis() - start) + "ms");
        return user;
    }
}
```

# 8. 常见异常码

| 异常码 | 含义 | 常见原因 |
|--------|------|---------|
| 0 | OK | 正常 |
| 20 | CLIENT_TIMEOUT | Consumer 端等待超时 |
| 30 | SERVER_TIMEOUT | Provider 处理超时 |
| 31 | SERVICE_NOT_FOUND | 找不到目标服务 |
| 40 | BAD_REQUEST | 请求包格式错误 |
| 50 | BAD_RESPONSE | 响应包格式错误 |
| 60 | SERVICE_ERROR | Provider 处理异常 |
| 70 | CLIENT_ERROR | Consumer 端错误 |
| 2 | FORBIDDEN_EXCEPTION | 访问受限（路由/鉴权拒绝） |
| 3 | LIMIT_EXCEEDED_EXCEPTION | 限流/并发达到上限 |

# 9. 快速自检清单

```
□ Provider 已启动且注册中心可见
□ Consumer 日志中有 SUBSCRIBE 成功记录
□ Consumer 和 Provider 的 interface/version/group 一致
□ Serialization 一致（hessian2/kryo/protobuf）
□ 网络可达（telnet 测试）
□ 线程池未满（检查 QOS threadpool 输出）
□ timeout 设置合理（区分快/慢接口）
□ 端口未被占用（netstat/lsof 检查）
□ API jar 版本一致（检查 serialVersionUID）
□ 注册中心地址正确（namespace/group 匹配）
```
