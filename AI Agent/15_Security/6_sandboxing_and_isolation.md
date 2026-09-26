# 1. 为什么需要沙箱

**结论先行**：Agent 可以执行代码、命令、浏览器操作，这些能力一旦被滥用即可从"助手"变成"内网跳板"。凡是 Agent 能执行代码或命令的场景，必须有沙箱（Sandbox，沙箱）。

## 1.1 能力与风险

Agent 的高危能力与对应风险：

| 能力 | 正常用途 | 被滥用后 |
|------|----------|----------|
| 执行代码 | 数据分析 | 读取敏感文件、反弹 Shell |
| 执行命令 | 运维操作 | 执行恶意命令 |
| 浏览网页 | 信息获取 | SSRF、内网扫描 |
| 调用 API | 业务集成 | 越权访问、数据外泄 |

```text
Agent 可执行代码/命令时，攻击者可：
- 读取服务器敏感文件
- 发起内网请求（SSRF）
- 执行恶意命令
- 耗尽 CPU/内存
- 作为跳板攻击内网
```

### 1.1.1 攻击示例

攻击者通过提示注入诱导 Agent 执行以下代码，若无沙箱将直接危害宿主：

```python
print(open("/root/.aws/credentials").read())            # 读取宿主机凭证

import urllib.request                                   # 访问云元数据
print(urllib.request.urlopen(
    "http://169.254.169.254/latest/meta-data/").read())

import os                                               # fork 炸弹
while True:
    os.fork()
```

## 1.2 威胁模型

沙箱的威胁模型包含两类对手：

| 对手 | 动机 | 典型手法 |
|------|------|----------|
| 外部攻击者 | 通过提示注入控制 Agent | 诱导执行恶意代码 |
| 内部误用 | 模型幻觉生成危险代码 | 误删文件、死循环 |

**边界与坑**：很多团队认为"模型不会主动作恶"，但提示注入可以让模型变成攻击者的执行器，威胁模型必须假设输入完全不可信。

**生产建议**：为每个执行入口标注"最坏情况影响"，若最坏情况是内网沦陷，则沙箱强度必须达到进程级/容器级隔离。

# 2. 沙箱的维度

**结论先行**：沙箱不是单一开关，而是文件系统、网络、进程、资源、权限、系统调用六个维度的组合约束。

## 2.1 六维隔离

| 维度 | 隔离内容 |
|------|----------|
| 文件系统 | 只能访问指定目录 |
| 网络 | 限制出网、禁止内网 |
| 进程 | 限制进程数、禁止 fork |
| 资源 | CPU、内存、磁盘、时间 |
| 权限 | 非 root、最小能力 |
| 系统调用 | seccomp 限制 |

### 2.1.1 维度协作

```text
        +---------------- 容器边界 ----------------+
        |  文件系统(只读)   网络(none)             |
        |  进程(pids)      资源(cpu/mem)           |
        |  权限(non-root)  系统调用(seccomp)       |
        |                                          |
        |            Agent 执行的代码              |
        +------------------------------------------+
                        |
                   宿主内核（共享）
```

## 2.2 隔离强度对比

| 方案 | 隔离级别 | 性能损耗 | 适用 |
|------|----------|:--------:|------|
| 语言级沙箱 | 弱 | 极低 | 纯计算 |
| 进程 + seccomp | 中 | 低 | 命令执行 |
| 容器 | 强 | 中 | 代码执行 |
| 微虚拟机 | 最强 | 高 | 高敏感 |

**生产建议**：按敏感度选择隔离级别，代码执行优先容器，高敏感场景用微虚拟机（如 Firecracker）；不要用"语言级沙箱"承载不可信代码。

# 3. 代码执行沙箱

**结论先行**：容器隔离是代码执行的首选方案，通过一次性容器 + 严格限制参数实现"用完即毁"。

## 3.1 容器隔离（推荐）

### 3.1.1 一次性容器

```java
public class ContainerCodeExecutor {

    public ExecutionResult execute(String code, Duration timeout) {
        String containerName = "sandbox-" + UUID.randomUUID();

        try {
            Process process = new ProcessBuilder(
                    "docker", "run", "--rm",
                    "--name", containerName,
                    "--network", "none",                    // 无网络
                    "--memory", "256m",                     // 内存限制
                    "--cpus", "0.5",                        // CPU 限制
                    "--pids-limit", "64",                   // 进程限制
                    "--read-only",                          // 只读文件系统
                    "--tmpfs", "/tmp:size=64m",             // 临时可写
                    "--user", "1000:1000",                  // 非 root
                    "--security-opt", "no-new-privileges",
                    "python:3.12-slim",
                    "python", "-c", code)
                    .redirectErrorStream(true)
                    .start();

            boolean finished = process.waitFor(
                    timeout.toMillis(), TimeUnit.MILLISECONDS);

            if (!finished) {
                process.destroyForcibly();
                return ExecutionResult.timeout();
            }
            return ExecutionResult.ok(readOutput(process));

        } catch (Exception e) {
            return ExecutionResult.error(e.getMessage());
        }
    }
}
```

### 3.1.2 关键限制参数

| 参数 | 作用 |
|------|------|
| `--network none` | 禁止网络 |
| `--memory 256m` | 内存上限 |
| `--cpus 0.5` | CPU 上限 |
| `--pids-limit` | 防 fork 炸弹 |
| `--read-only` | 只读根文件系统 |
| `--user` | 非 root |
| `--cap-drop ALL` | 丢弃所有能力 |
| `--security-opt no-new-privileges` | 禁止提权 |

### 3.1.3 输出处理与大小限制

容器输出也必须限制：超大输出会撑爆内存或日志系统。

```java
private String readOutput(Process p) throws IOException {
    try (InputStream in = p.getInputStream()) {
        byte[] buf = in.readNBytes(MAX_OUTPUT_BYTES);   // 例如 64KB
        return new String(buf, StandardCharsets.UTF_8);
    }
}
```

## 3.2 非容器方案对比

| 方案 | 优点 | 缺点 |
|------|------|------|
| Docker | 生态成熟 | 需守护进程 |
| gVisor | 系统调用拦截强 | 兼容性差 |
| Firecracker | 强隔离、启动快 | 运维复杂 |
| WASM | 极轻量 | 生态受限 |

**边界与坑**：`--network none` 会让需要联网的代码失败，若业务必须联网，应改为"仅允许访问白名单域名"的出网代理，而不是直接放开。

**生产建议**：为容器镜像预装最小依赖，禁用包管理器；每次执行都新建容器，禁止复用，避免状态残留。

# 4. 浏览器沙箱

**结论先行**：浏览器自动化同样需要沙箱，重点是域名白名单与内网地址屏蔽，防止 SSRF 与内网探测。

## 4.1 域名与网络限制

| 限制 | 目的 |
|------|------|
| 域名白名单 | 只访问可信站点 |
| 内网地址屏蔽 | 防 SSRF |
| 协议白名单 | 禁 `file://`、`gopher://` |
| 下载限制 | 防恶意文件落盘 |

## 4.2 实现

```java
// Playwright：限制可访问域名与操作
public class SandboxedBrowser {

    public String browse(String url) {
        // ① 域名白名单
        if (!isAllowedDomain(url)) {
            throw new SecurityException("域名不在白名单: " + url);
        }

        // ② 禁止访问内网地址（防 SSRF）
        if (isInternalAddress(url)) {
            throw new SecurityException("禁止访问内网地址");
        }

        return playwright.open(url);
    }

    private boolean isInternalAddress(String url) {
        try {
            InetAddress addr = InetAddress.getByName(new URI(url).getHost());
            return addr.isSiteLocalAddress()
                    || addr.isLoopbackAddress()
                    || addr.isLinkLocalAddress();
        } catch (Exception e) {
            return true;                     // 解析失败也拒绝
        }
    }
}
```

### 4.2.1 域名白名单实现

```java
private static final Set<String> ALLOWED_DOMAINS =
        Set.of("docs.example.com", "api.example.com");

private boolean isAllowedDomain(String url) {
    String host = URI.create(url).getHost();
    if (host == null) return false;
    return ALLOWED_DOMAINS.stream()
            .anyMatch(d -> host.equals(d) || host.endsWith("." + d));
}
```

**边界与坑**：只做一次 DNS 解析校验存在"检查时安全、使用时被重绑定"的时间差（TOCTOU），必须把校验结果与实际连接绑定。

**生产建议**：浏览器运行在无出网权限的容器内，所有外网访问经统一代理，代理侧再做域名白名单与 IP 校验。

# 5. SSRF 防护（关键）

**结论先行**：Agent 调用外部 URL 时必须防 SSRF（Server-Side Request Forgery，服务端请求伪造），核心是屏蔽内网地址与云元数据地址。

## 5.1 攻击面

| 目标 | 危害 |
|------|------|
| 云元数据 169.254.169.254 | 窃取临时凭证 |
| 内网服务 | 越权访问、横向移动 |
| 本地回环 | 访问本机管理接口 |
| `file://` | 读取本地文件 |

## 5.2 防护实现

```java
@Component
public class SsrfGuard {

    private static final Set<String> BLOCKED_HOSTS = Set.of(
            "169.254.169.254",               // 云元数据服务
            "metadata.google.internal",
            "localhost", "127.0.0.1");

    public void validate(String url) {
        URI uri = URI.create(url);

        // ① 协议白名单
        if (!Set.of("http", "https").contains(uri.getScheme())) {
            throw new SecurityException("不支持的协议: " + uri.getScheme());
        }

        // ② 主机黑名单
        if (BLOCKED_HOSTS.contains(uri.getHost())) {
            throw new SecurityException("禁止访问: " + uri.getHost());
        }

        // ③ 解析后检查 IP（防 DNS 重绑定）
        InetAddress addr = InetAddress.getByName(uri.getHost());
        if (addr.isSiteLocalAddress() || addr.isLoopbackAddress()
                || addr.isLinkLocalAddress() || addr.isAnyLocalAddress()) {
            throw new SecurityException("禁止访问内网地址");
        }

        // ④ 端口白名单
        int port = uri.getPort() == -1 ? defaultPort(uri.getScheme()) : uri.getPort();
        if (!Set.of(80, 443, 8080).contains(port)) {
            throw new SecurityException("端口不允许: " + port);
        }
    }
}
```

> 云元数据地址 169.254.169.254 是 SSRF 的头号目标，必须屏蔽。

### 5.2.1 端口与协议白名单

| 项 | 白名单 | 说明 |
|----|--------|------|
| 协议 | http、https | 禁 `file`、`gopher`、`dict` |
| 端口 | 80、443、8080 | 禁 22、6379、9200 等 |
| 主机 | 业务域名 | 禁 IP 直连与内网域名 |

## 5.3 DNS 重绑定与绕过

### 5.3.1 常见绕过手法

| 绕过 | 原理 | 防御 |
|------|------|------|
| DNS 重绑定 | 首次解析为公网，连接时变内网 | 校验与连接用同一 IP |
| 十进制 IP | `http://2130706433/` | 解析后按 IP 校验 |
| 302 跳转 | 首跳公网，二跳内网 | 禁止自动跟随跳转 |
| IPv6 映射 | `::ffff:127.0.0.1` | 归一化后校验 |

### 5.3.2 安全请求实现

```java
public String safeFetch(String url) throws IOException {
    ssrfGuard.validate(url);
    URI uri = URI.create(url);
    InetAddress addr = InetAddress.getByName(uri.getHost());

    HttpClient client = HttpClient.newBuilder()
            .followRedirects(HttpClient.Redirect.NEVER)   // 禁自动跳转
            .build();
    // 用已校验的 IP 直连，Host 头保留原域名
    HttpRequest req = HttpRequest.newBuilder()
            .uri(URI.create(uri.getScheme() + "://" + addr.getHostAddress()
                    + ":" + port(uri) + uri.getRawPath()))
            .header("Host", uri.getHost())
            .build();
    return client.send(req, HttpResponse.BodyHandlers.ofString()).body();
}
```

**边界与坑**：仅用黑名单永远不完备，应改为"白名单 + 解析后 IP 校验 + 禁跳转"的组合，并限制响应体大小与超时。

**生产建议**：把出网能力集中到统一网关，网关做协议、域名、IP、端口、大小、超时的全量管控，业务侧无直接出网权限。

# 6. 文件系统隔离

**结论先行**：文件访问必须防路径穿越与符号链接，所有路径都要先归一化再校验前缀。

## 6.1 路径穿越防护

| 攻击 | 防护 |
|------|------|
| `../../etc/passwd` | `normalize()` + `startsWith()` |
| 符号链接 | 检查 `isSymbolicLink` |
| 绝对路径 | 强制 resolve 到根目录 |

## 6.2 实现与边界

```java
public class SafeFileAccess {

    private final Path allowedRoot = Path.of("/data/workspace").toAbsolutePath();

    public String read(String path) throws IOException {
        Path target = allowedRoot.resolve(path).normalize().toAbsolutePath();

        // 路径穿越防护
        if (!target.startsWith(allowedRoot)) {
            throw new SecurityException("路径越界: " + path);
        }
        // 符号链接防护
        if (Files.isSymbolicLink(target)) {
            throw new SecurityException("不允许访问符号链接");
        }
        return Files.readString(target);
    }
}
```

### 6.2.1 真实路径校验

```java
public String safeRead(String path) throws IOException {
    Path target = allowedRoot.resolve(path).normalize();
    // 解析真实路径（跟随符号链接）后再校验前缀
    Path real = target.toRealPath();
    if (!real.startsWith(allowedRoot.toRealPath())) {
        throw new SecurityException("路径越界: " + path);
    }
    return Files.readString(real);
}
```

**边界与坑**：`normalize()` 只能消除 `..`，若父目录中存在符号链接，归一化后的路径仍可能指向外部；需用 `toRealPath()` 解析真实路径后再校验。

**生产建议**：用独立的只读挂载提供数据，工作目录用 tmpfs；禁止 Agent 访问宿主机的 `/proc`、`/sys` 与 Docker 套接字。

# 7. 资源限制

**结论先行**：资源限制是防止"单个请求拖垮整个服务"的必备项，覆盖 CPU、内存、磁盘、时间、进程数。

## 7.1 资源维度

| 资源 | 风险 | 限制手段 |
|------|------|----------|
| CPU | 死循环 | cgroups/超时 |
| 内存 | OOM | 内存上限 |
| 磁盘 | 写满 | 配额/只读 |
| 时间 | 长任务 | 墙钟超时 |
| 进程 | fork 炸弹 | pids-limit |

## 7.2 实现

```java
public class ResourceLimiter {

    public ExecutionResult executeWithLimits(Supplier<String> task) {
        ExecutorService executor = Executors.newSingleThreadExecutor();
        Future<String> future = executor.submit(task::get);

        try {
            return ExecutionResult.ok(future.get(10, TimeUnit.SECONDS));
        } catch (TimeoutException e) {
            future.cancel(true);
            return ExecutionResult.timeout();
        } finally {
            executor.shutdownNow();
        }
    }
}
```

### 7.2.1 容器级硬限制

应用层超时是"软限制"，真正的兜底在容器：

| 资源 | 软限制（应用层） | 硬限制（容器） |
|------|------------------|----------------|
| 时间 | `future.get(timeout)` | 执行器墙钟超时 |
| CPU | 无 | `--cpus` |
| 内存 | 无 | `--memory` |
| 进程 | 无 | `--pids-limit` |

**边界与坑**：`future.cancel(true)` 依赖任务响应中断，若任务内部是纯计算循环且不检查中断标志，线程无法被真正停止；必须配合容器级 CPU/内存硬限制。

**生产建议**：应用层超时用于快速失败，容器级 cgroups 用于兜底；两者都要有，不能只靠其中一层。

# 8. 命令执行与 Shell 注入

**结论先行**：Agent 拼接命令时，用户输入会变成 Shell 注入入口，必须避免 `Runtime.exec(String)` 与字符串拼接。

## 8.1 注入面

| 写法 | 风险 |
|------|------|
| `sh -c "..."` | 整串交给 Shell 解释 |
| 字符串拼接参数 | 参数可含 `;`、`|`、`$()` |
| 命令白名单缺失 | 可执行任意命令 |

## 8.2 防护实现

```java
public Process runCommand(List<String> command, Duration timeout)
        throws IOException {
    // 白名单校验首个参数（可执行文件）
    if (!ALLOWED_COMMANDS.contains(command.get(0))) {
        throw new SecurityException("命令不允许: " + command.get(0));
    }
    // 以参数数组形式执行，不经过 Shell
    Process p = new ProcessBuilder(command)
            .redirectErrorStream(true)
            .start();

    boolean ok = p.waitFor(timeout.toMillis(), TimeUnit.MILLISECONDS);
    if (!ok) {
        p.destroyForcibly();
        throw new SecurityException("命令执行超时");
    }
    return p;
}
```

### 8.2.1 危险命令与参数清单

| 命令 | 危险点 | 处理 |
|------|--------|------|
| `rm` | 删除文件 | 白名单排除或沙箱只读 |
| `curl`/`wget` | 出网 | 走代理白名单 |
| `chmod`/`chown` | 提权 | 禁止 |
| `nc`/`ncat` | 反弹 Shell | 禁止 |
| `find -exec` | 任意命令 | 校验参数 |

**边界与坑**：`ProcessBuilder` 传入数组可避免 Shell 解释，但若命令本身（如 `find`）支持危险选项，仍可能越界；命令参数需逐项校验。

**生产建议**：优先用固定脚本封装操作，而不是允许模型自由拼命令；对危险命令（删除、网络、提权）强制人工审批。

# 9. 生产注意事项

**结论先行**：沙箱失效往往源于"漏了一个维度"，需以清单方式逐项核对。

## 9.1 风险与建议

| 风险 | 建议 |
|------|------|
| 无沙箱执行代码 | 必须容器隔离 |
| SSRF | 屏蔽内网与元数据地址 |
| 路径穿越 | normalize + startsWith |
| 无资源限制 | CPU/内存/时间/进程 |
| root 运行 | 非 root + 丢能力 |
| 无出网管控 | 默认禁网，按需放行 |

## 9.2 上线检查清单

- 执行入口是否全部走沙箱，有无旁路。
- 网络是否默认禁用，出网是否经代理白名单。
- 是否屏蔽云元数据地址与内网网段。
- 文件访问是否归一化并解析真实路径。
- 是否设置 CPU/内存/时间/进程硬限制。
- 容器是否非 root 且丢弃全部能力。
- 是否禁止自动跟随 HTTP 跳转。

**边界与坑**：默认放行、按需封禁的思路迟早会漏；安全策略应采用"默认拒绝、显式放行"的白名单模型。

## 9.3 红队演练

定期用逃逸样本验证沙箱有效性，样本至少覆盖：

| 类别 | 样本 |
|------|------|
| 文件逃逸 | `../../etc/passwd`、符号链接 |
| 网络逃逸 | 元数据地址、DNS 重绑定 |
| 资源耗尽 | fork 炸弹、内存炸弹 |
| 提权 | setuid、`no-new-privileges` 绕过 |

演练结果纳入安全基线，任何新发现的逃逸路径都要补上回归用例。

# 10. 小结

- 凡是执行代码/命令/浏览器的场景，必须有沙箱。
- 容器隔离是代码执行的首选方案。
- SSRF 防护要屏蔽内网与云元数据地址。
- 文件访问要防路径穿越与符号链接。
- 资源限制（CPU/内存/时间/进程）是必备项。
- 采用"默认拒绝、显式放行"，并集中管控出网。
