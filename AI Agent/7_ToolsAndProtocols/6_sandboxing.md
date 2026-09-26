# 1. 沙箱的目标

结论先行：沙箱让不可信的执行（代码/命令/浏览器）运行在受限环境中，即使被滥用也影响不到宿主机与内网。

## 1.1 威胁模型

### 1.1.1 威胁来源

```
让不可信的执行（代码/命令/浏览器）在受限环境中运行：
- 即使被滥用，也影响不到宿主机与内网
```

| 威胁 | 例子 |
|------|------|
| 文件读写 | 读取 `/etc/passwd`、密钥文件 |
| 网络外联 | 回连攻击者、扫描内网 |
| 资源耗尽 | 死循环、fork 炸弹 |
| 权限提升 | 利用内核漏洞提权 |
| 数据外泄 | 通过 DNS/HTTP 带出数据 |

> 详细的安全原理见 `15_Security/6_sandboxing_and_isolation.md`，本文聚焦**工具执行视角**。

### 1.1.2 目标清单

| 目标 | 手段 |
|------|------|
| 隔离文件系统 | 只读根 + 受限挂载 |
| 隔离网络 | 默认禁网 |
| 限制资源 | CPU/内存/进程/时间 |
| 降权 | 非 root + 丢能力 |
| 可审计 | 记录执行与输出 |

## 1.2 沙箱与安全的关系

### 1.2.1 纵深防御

沙箱是最后一道防线，前面还有输入校验、静态检查、权限控制，多层叠加才安全。

```
输入校验 → 静态检查 → 权限控制 → 沙箱隔离 → 监控告警
```

### 1.2.2 边界与坑

- 只靠沙箱：配置一旦有疏漏即失守，需多层防御。
- 沙箱万能论：内核漏洞可逃逸，需及时更新与加固。
- 沙箱不记录：出事无法溯源。

### 1.2.3 信任边界

```
可信区（Trusted）          不可信区（Untrusted）
  控制面、调度器    ──▶    代码、命令、浏览器、上传文件
  凭据、数据库              沙箱内不得持有任何业务凭据
```

# 2. 隔离层级

结论先行：隔离强度从进程、容器、微虚拟机到虚拟机递增，成本同向增长，按风险与性能要求选择。

## 2.1 四个层级

### 2.1.1 层级对比表

```
① 进程隔离   独立进程，限制资源
② 容器隔离   独立文件系统、网络、资源（推荐）
③ 虚拟机隔离 最强隔离，成本最高
④ 微虚拟机   Firecracker/gVisor，兼顾隔离与性能
```

| 层级 | 隔离强度 | 启动速度 | 成本 |
|------|:--------:|:--------:|:----:|
| 进程 | 弱 | 快 | 低 |
| 容器 | 中强 | 快 | 低 |
| 微虚拟机 | 强 | 中 | 中 |
| 虚拟机 | 最强 | 慢 | 高 |

### 2.1.2 隔离层次图

```
┌──────────────────────────────────────┐
│ 宿主机（Host）                        │
│  ┌──────────────┐  ┌──────────────┐  │
│  │ 容器 A        │  │ 容器 B        │  │
│  │ 独立文件系统  │  │ 独立网络      │  │
│  └──────────────┘  └──────────────┘  │
└──────────────────────────────────────┘
```

## 2.2 层级选择

### 2.2.1 选择依据

| 场景 | 推荐层级 |
|------|----------|
| 开发/测试 | 进程 |
| 一般生产 | 容器 |
| 高安全多租户 | 微虚拟机 |
| 强合规 | 虚拟机 |

### 2.2.2 边界与坑

- 用进程隔离跑不可信代码：几乎等同于不隔离。
- 一味追求最强隔离：启动慢、成本高，可能不必要。
- 忽略共享内核：容器与宿主机共享内核，是主要攻击面。

### 2.2.3 逃逸与缓解

| 逃逸方式 | 缓解 |
|----------|------|
| 内核漏洞 | 及时打补丁、seccomp |
| 配置错误 | 参数清单化、自动化校验 |
| 特权容器 | 禁 `--privileged`、丢能力 |
| 挂载敏感路径 | 挂载白名单 |

# 3. 容器隔离清单

结论先行：容器隔离是否有效取决于参数是否完整——网络、资源、文件系统、权限四类都要配置。

## 3.1 完整参数

### 3.1.1 完整 Java 示例

```java
ProcessBuilder pb = new ProcessBuilder(
        "docker", "run", "--rm",
        "--name", containerName,

        // 网络：默认禁用
        "--network", "none",

        // 资源限制
        "--memory", "256m",
        "--memory-swap", "256m",           // 禁止 swap
        "--cpus", "0.5",
        "--pids-limit", "64",

        // 文件系统：只读根 + 临时可写
        "--read-only",
        "--tmpfs", "/tmp:size=64m,noexec",  // noexec 禁止执行
        "-v", workDir + ":/workspace:rw",

        // 权限：非 root + 丢能力
        "--user", "1000:1000",
        "--cap-drop", "ALL",
        "--security-opt", "no-new-privileges",

        // seccomp 限制系统调用
        "--security-opt", "seccomp=default.json",

        image, command);
```

### 3.1.2 参数防护表

| 参数 | 防护 |
|------|------|
| `--network none` | 禁止出网与内网访问 |
| `--memory` | 内存耗尽 |
| `--cpus` | CPU 耗尽 |
| `--pids-limit` | fork 炸弹 |
| `--read-only` | 篡改系统文件 |
| `--tmpfs noexec` | 执行上传的二进制 |
| `--cap-drop ALL` | 特权操作 |
| `no-new-privileges` | 提权 |

## 3.2 各维度加固

### 3.2.1 资源、文件系统与权限

| 维度 | 关键配置 | 缺失后果 |
|------|----------|----------|
| 资源 | memory/cpus/pids | 宿主机资源被耗尽 |
| 文件系统 | read-only + tmpfs | 系统文件被篡改 |
| 权限 | user + cap-drop | 容器内可提权 |
| 网络 | network none | 内网被扫描 |
| 系统调用 | seccomp | 内核攻击面扩大 |

### 3.2.2 边界与坑

- 只设 `--memory` 不设 `--memory-swap`：容器可借用 swap，内存限制形同虚设。
- `tmpfs` 不设 `noexec`：攻击者可写入并执行二进制。
- 不配 seccomp：默认能力集可能仍偏宽，需按需收紧。

### 3.2.3 配置校验

```java
public void validate(List<String> args) {
    require(args, "--network", "none");
    require(args, "--read-only");
    require(args, "--cap-drop", "ALL");
    if (args.contains("--privileged")) {
        throw new IllegalArgumentException("禁止特权容器");
    }
}
```

# 4. 超时与清理

结论先行：任何执行都必须有超时，超时后必须强制清理容器，否则会累积僵尸容器。

## 4.1 超时机制

### 4.1.1 完整 Java 示例

```java
public ExecutionResult executeWithTimeout(ProcessBuilder pb, Duration timeout) {
    Process process = pb.start();
    String containerName = extractContainerName(pb);

    try {
        if (!process.waitFor(timeout.toMillis(), TimeUnit.MILLISECONDS)) {
            // 超时 → 强制清理
            new ProcessBuilder("docker", "kill", containerName)
                    .start().waitFor(5, TimeUnit.SECONDS);
            return ExecutionResult.timeout();
        }
        return ExecutionResult.ok(readOutput(process));
    } catch (Exception e) {
        cleanupContainer(containerName);
        return ExecutionResult.error(e.getMessage());
    }
}
```

### 4.1.2 清理策略

| 策略 | 说明 |
|------|------|
| `--rm` | 退出即删除容器 |
| 显式 kill | 超时后主动 kill |
| 定时巡检 | 定期清理孤儿容器 |
| 标签标记 | 给沙箱容器打标签便于批量清理 |

> **必须保证超时后容器被清理**，否则会累积僵尸容器。

## 4.2 边界与坑

### 4.2.1 僵尸容器

`docker run` 被中断、JVM 崩溃、网络抖动都可能导致容器残留，需定时巡检清理。

```java
// 定期清理超过 10 分钟仍存活的沙箱容器
docker ps --filter "label=sandbox=true" --format "{{.Names}} {{.RunningFor}}"
```

### 4.2.2 边界与坑

- 超时只杀进程不杀容器：容器继续运行占用资源。
- 无兜底巡检：异常路径残留无法清理。
- 清理不幂等：重复 kill 会报错，需忽略不存在的情况。

### 4.2.3 分层超时

| 层级 | 超时 |
|------|------|
| 单次执行 | 30 秒 |
| 容器创建 | 10 秒 |
| 结果读取 | 5 秒 |
| 池等待 | 5 秒 |

> 任一环节都应有独立超时，避免整体卡死。

# 5. 沙箱池化

结论先行：频繁创建容器开销大，可用容器池复用，但必须做好状态重置。

## 5.1 池化实现

### 5.1.1 完整 Java 示例

```java
public class SandboxPool {

    private final BlockingQueue<String> idle = new LinkedBlockingQueue<>();
    private final int poolSize;

    public SandboxPool(int poolSize) {
        this.poolSize = poolSize;
        for (int i = 0; i < poolSize; i++) {
            idle.offer(createIdleSandbox());
        }
    }

    public <T> T withSandbox(Function<String, T> action) {
        String sandboxId = idle.poll(5, TimeUnit.SECONDS);
        if (sandboxId == null) {
            throw new SandboxUnavailableException("沙箱资源耗尽");
        }
        try {
            return action.apply(sandboxId);
        } finally {
            reset(sandboxId);            // 重置状态
            idle.offer(sandboxId);
        }
    }
}
```

### 5.1.2 重置与复用

```java
private void reset(String sandboxId) {
    // 清理工作目录、进程、临时文件
    runInSandbox(sandboxId, "rm -rf /workspace/* /tmp/*");
    killSandboxProcesses(sandboxId);
}
```

## 5.2 池化权衡

### 5.2.1 优缺点表

| 优点 | 缺点 |
|------|------|
| 降低启动延迟 | 状态需重置 |
| 提高吞吐 | 内存常驻 |

### 5.2.2 边界与坑

- 池化后不复位：上次执行的文件、进程、环境变量污染下次执行。
- 池过小：请求排队，超时率上升。
- 池过大：内存常驻，浪费资源。
- 长任务占用连接：应限制单次执行时长，避免池被长期占用。

### 5.2.3 池与按需的取舍

| 维度 | 池化 | 按需创建 |
|------|------|----------|
| 启动延迟 | 低 | 高 |
| 隔离洁净度 | 依赖复位 | 天然洁净 |
| 资源占用 | 常驻 | 按需 |
| 适用 | 高频短任务 | 低频长任务 |

# 6. 网络访问控制

结论先行：网络默认禁用；确需联网时，只允许经白名单代理访问指定域名。

## 6.1 默认禁网

### 6.1.1 网络模式表

| 模式 | 说明 | 适用 |
|------|------|------|
| none | 无网络 | 默认推荐 |
| 白名单代理 | 经代理出网 | 需安装依赖 |
| 桥接 | 完整网络 | 不推荐 |

### 6.1.2 白名单代理

```java
// 方案：使用带代理的白名单网络
"--network", "sandbox-net",
"-e", "HTTP_PROXY=http://proxy:8080",
"-e", "HTTPS_PROXY=http://proxy:8080",
"-e", "NO_PROXY=localhost"
```

代理侧做域名白名单：

```
允许：pypi.org, files.pythonhosted.org
拒绝：内网地址、云元数据地址（169.254.169.254）
```

> 优先**预装依赖镜像**，避免运行时联网。

## 6.2 边界与坑

### 6.2.1 元数据地址

云环境的元数据地址（如 `169.254.169.254`）可获取临时凭据，必须显式拒绝，不能只依赖"内网不可达"。

### 6.2.2 边界与坑

- 代理不校验域名：等同于全开放网络。
- 忽略 DNS 外泄：即使禁 HTTP，DNS 查询也可带出数据。
- 代理可被绕过：代码直接连 IP 会绕过 HTTP 代理，需网络层强制。

# 7. 文件系统隔离

结论先行：只挂载必要目录并限制权限，绝不挂载敏感路径。

## 7.1 挂载原则

### 7.1.1 挂载清单

```java
// 只挂载必要目录，且限制权限
"-v", workDir + ":/workspace:rw",       // 工作目录可写
"-v", libDir + ":/lib:ro",              // 依赖只读

// 绝不挂载
// /var/run/docker.sock   ← 会获得宿主机 docker 权限
// /etc, /proc, /sys      ← 系统目录
// 用户的 home 目录        ← 隐私
```

### 7.1.2 完整 Java 示例

```java
public ProcessBuilder buildMounts(Path workDir, Path libDir) {
    return new ProcessBuilder(
            "docker", "run", "--rm",
            "--read-only",
            "-v", workDir + ":/workspace:rw",
            "-v", libDir + ":/opt/lib:ro",
            "--tmpfs", "/tmp:size=64m,noexec",
            "--network", "none",
            image, command);
}
```

## 7.2 路径安全

### 7.2.1 路径校验

```java
public Path resolveSafe(Path base, String userPath) {
    Path resolved = base.resolve(userPath).normalize();
    if (!resolved.startsWith(base)) {
        throw new SecurityException("路径越界: " + userPath);
    }
    return resolved;
}
```

### 7.2.2 边界与坑

- 挂载 `docker.sock`：容器可直接控制宿主机 Docker，等同于拿到宿主机 root。
- 挂载目录可写且过宽：代码可篡改宿主文件。
- 路径不校验：`../../etc/passwd` 可越界读取。

# 8. 沙箱内运行的 Agent 工具

结论先行：不同工具对沙箱的要求不同，需按工具类型配置隔离级别。

## 8.1 工具与沙箱要求

### 8.1.1 要求表

| 工具 | 沙箱要求 |
|------|----------|
| 代码执行 | 容器 + 禁网 + 资源限制 |
| Shell 命令 | 白名单命令 + 容器 |
| 文件操作 | 限定目录 + 路径校验 |
| 浏览器 | 容器 + 域名白名单 |
| 数据库 | 只读账号 + 查询限制 |

### 8.1.2 完整 Java 示例

```java
public ToolResult runShell(String command) {
    if (!commandWhitelist.allows(command)) {
        return ToolResult.fail("命令不在白名单内");
    }
    return sandboxExecutor.execute(command, Duration.ofSeconds(15));
}
```

## 8.2 边界与坑

### 8.2.1 工具逃逸面

| 工具 | 逃逸面 |
|------|--------|
| Shell | 命令注入、管道串联 |
| 浏览器 | 恶意页面、下载执行 |
| 数据库 | 注入、越权查询 |
| 文件 | 路径穿越、符号链接 |

### 8.2.2 边界与坑

- 命令白名单只校验前缀：`ls; rm -rf /` 可绕过。
- 文件操作不校验软链：符号链接可指向敏感文件。
- 浏览器不隔离：恶意页面可攻击宿主浏览器。

# 9. 监控

结论先行：沙箱必须监控执行量、超时率、失败率与池可用数，异常时告警。

## 9.1 指标采集

### 9.1.1 完整 Java 示例

```java
registry.counter("sandbox.executions", "language", lang).increment();
registry.counter("sandbox.timeouts").increment();
registry.counter("sandbox.errors").increment();
registry.gauge("sandbox.pool.available", pool, p -> p.available());
```

### 9.1.2 告警规则

| 指标 | 告警 |
|------|------|
| 超时率 | 突增 |
| 池可用数 | 持续为 0 |
| 执行失败率 | 超阈值 |

## 9.2 边界与坑

### 9.2.1 指标盲区

| 盲区 | 后果 |
|------|------|
| 无执行耗时 | 无法定位慢任务 |
| 无输出大小 | 无法发现异常打印 |
| 无退出码 | 无法区分正常/异常 |

### 9.2.2 边界与坑

- 只看成功率：超时与失败被混为一谈。
- 无池水位监控：池耗尽后请求全部失败才发现。
- 无审计日志：无法追溯谁执行了什么。

# 10. 生产注意事项

结论先行：上线前核对沙箱隔离、敏感挂载、超时清理、资源限制、网络与监控六件事。

## 10.1 风险清单

| 风险 | 建议 |
|------|------|
| 无沙箱 | 必须隔离 |
| 挂载 docker.sock | 绝对禁止 |
| 无超时清理 | 强制杀容器 |
| 无资源限制 | CPU/内存/进程 |
| 有内网访问 | 默认禁网 |
| 容器泄漏 | 池化 + 定期清理 |
| 无监控 | 监控超时与池可用 |

## 10.2 部署检查表

- 沙箱容器参数按第 3 节清单完整配置。
- 默认禁网，联网走白名单代理。
- 绝不挂载 `docker.sock`、`/etc`、`/proc`、`/sys`。
- 超时后强制 kill，并有定时巡检。
- 池化沙箱每次使用后复位。
- 执行、超时、失败均进监控与审计。

# 11. 小结

## 11.1 核心要点

- 沙箱让不可信执行影响不到宿主机与内网。
- 容器隔离是生产首选，配置要完整。
- 超时后必须清理容器，可用池化提升性能。
- 网络默认禁用，必要时用白名单代理。
- 绝不挂载 docker.sock 等敏感路径。

## 11.2 落地建议

1. 从"禁网 + 限资源 + 非 root"三件套开始，快速建立基本隔离。
2. 用 `--rm` 加定时巡检双保险，防止容器泄漏。
3. 池化仅在启动延迟成为瓶颈时引入，并做好复位。
4. 建立超时率与池水位告警，尽早发现异常。
