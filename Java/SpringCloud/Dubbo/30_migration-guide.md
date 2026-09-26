
# 1. 迁移概述

Dubbo 3.x 是面向云原生的重大升级。相对于 2.7.x，3.x 在服务发现模型、通信协议、性能优化等方面有根本性变化。

```
Dubbo 2.7.x ─────────────────────→ Dubbo 3.3.x
  ├── 接口级服务发现                   ├── 应用级服务发现（默认）
  ├── Dubbo 协议为主                   ├── Triple 协议（推荐）
  ├── Spring Boot 2.x                ├── Spring Boot 3.x
  ├── JDK 8                          ├── JDK 17+
  └── javax.* 命名空间                └── jakarta.* 命名空间
```

# 2. 版本兼容矩阵

| Dubbo 版本 | Spring Boot | Spring Cloud | JDK | 服务发现模式 |
|-----------|-------------|--------------|-----|-------------|
| 2.7.x | 2.x | Hoxton / 2020.x | 8 | 接口级 |
| 3.0.x | 2.x | 2021.x | 8 | 接口级（默认）/ 应用级 |
| 3.1.x | 2.x / 3.x | 2021.x / 2022.x | 8 / 17 | 应用级（默认） |
| 3.2.x | 3.x | 2022.x | 17 | 应用级 |
| 3.3.x | 3.x | 2023.x | 17 / 21 | 应用级 |

# 3. 迁移步骤

## 3.1 第一步：升级 JDK 与 Spring Boot

```xml
<!-- 父 POM 升级 -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.0</version>   <!-- 从 2.7.x 升级 -->
</parent>

<properties>
    <java.version>17</java.version>    <!-- 从 1.8 升级 -->
    <dubbo.version>3.3.0</dubbo.version>
</properties>
```

## 3.2 第二步：替换 javax 为 jakarta

Dubbo 3.2+ 依赖 Spring Boot 3.x，Spring Boot 3.x 使用 Jakarta EE 9+。

```java
// 旧（Dubbo 2.7 + Spring Boot 2.x）
import javax.servlet.http.HttpServletRequest;
import javax.validation.constraints.NotNull;

// 新（Dubbo 3.3 + Spring Boot 3.x）
import jakarta.servlet.http.HttpServletRequest;
import jakarta.validation.constraints.NotNull;
```

**全局替换命令**（IDE 批量操作）：
- `javax.servlet` → `jakarta.servlet`
- `javax.validation` → `jakarta.validation`
- `javax.persistence` → `jakarta.persistence`
- `javax.annotation` → `jakarta.annotation`

## 3.3 第三步：切换服务发现模式

**关键决策**：Dubbo 3 默认使用应用级服务发现，但兼容接口级。

```yaml
# 推荐：应用级发现（Dubbo 3 默认）
dubbo:
  application:
    register-mode: instance    # 可选: instance（应用级）/ interface（接口级）/ all（双注册）

  # 如有特殊需求可暂时保持接口级（确保兼容性过渡）
  # application:
  #   register-mode: interface
```

**双注册模式**（迁移过渡期）：

```yaml
dubbo:
  application:
    register-mode: all  # 同时向注册中心注册应用级和接口级服务

  provider:
    parameters:
      migration.step: APPLICATION_FIRST  # 优先使用应用级发现
```

## 3.4 第四步：协议切换（可选）

```yaml
# Dubbo 2.7 典型配置
dubbo:
  protocol:
    name: dubbo
    port: 20880

# Dubbo 3 推荐配置
dubbo:
  protocol:
    name: triple      # 从 dubbo → triple（gRPC 兼容）
    port: 50051       # Triple 默认端口
```

若暂不切换协议，Dubbo 协议在 3.x 中仍然可用且性能有提升。

## 3.5 第五步：依赖调整

```xml
<!-- Dubbo 2.7 -->
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-spring-boot-starter</artifactId>
    <version>2.7.23</version>
</dependency>

<!-- Dubbo 3.3 -->
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-spring-boot-starter</artifactId>
    <version>3.3.0</version>
</dependency>

<!-- 新增：Dubbo 3 应用级发现所需的元数据上报 -->
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-metadata-report-nacos</artifactId>
    <version>${dubbo.version}</version>
</dependency>
```

# 4. 配置变更对照表

| 配置项 | Dubbo 2.7 | Dubbo 3.3 | 说明 |
|--------|-----------|-----------|------|
| `dubbo.application.register-mode` | 无此配置 | `instance`（默认） | 新增 |
| `dubbo.protocol.name` | `dubbo` | `triple`（推荐） | 新增 Triple |
| `dubbo.metadata-report.*` | 可选 | 推荐配置 | 应用级发现依赖 |
| `dubbo.scan.base-packages` | 必需 | 可选 | 3.x 增强了自动扫描 |
| `dubbo.registry.parameters.namespace` | 常用 | 推荐 `namespace` | 统一 |
| `@org.apache.dubbo.config.annotation.Service` | 使用 | 已弃用 | 改用 `@DubboService` |
| `@org.apache.dubbo.config.annotation.Reference` | 使用 | 已弃用 | 改用 `@DubboReference` |

## 4.1 注解迁移

```java
// Dubbo 2.7
import org.apache.dubbo.config.annotation.Service;
import org.apache.dubbo.config.annotation.Reference;

@Service(version = "1.0.0", timeout = 3000)
public class UserServiceImpl implements UserService { }

@Reference(version = "1.0.0", timeout = 3000)
private UserService userService;

// Dubbo 3.3（推荐）
import org.apache.dubbo.config.annotation.DubboService;
import org.apache.dubbo.config.annotation.DubboReference;

@DubboService(version = "1.0.0", timeout = 3000)
public class UserServiceImpl implements UserService { }

@DubboReference(version = "1.0.0", timeout = 3000)
private UserService userService;
```

## 4.2 新增配置项

```yaml
dubbo:
  # Dubbo 3 新增配置
  application:
    register-mode: instance            # 服务注册模式
    enable-empty-protection: true      # 空保护机制
    enable-file-cache: true            # 本地文件缓存

  metadata-report:                     # 元数据上报（应用级发现必需）
    address: nacos://127.0.0.1:8848
    parameters:
      namespace: dev

  provider:
    warmup: 120000                     # 预热时间(ms)
    export-async: false                # 异步导出
```

# 5. 兼容性策略

## 5.1 Provider 先升级

推荐"先升级 Provider，再升级 Consumer"的策略：

```
第1步: Provider 升级到 Dubbo 3（register-mode: all，双注册）
第2步: Consumer 升级到 Dubbo 3（仍使用接口级发现）
第3步: Consumer 切换为应用级发现
第4步: Provider 切换为 register-mode: instance（纯应用级）
```

## 5.2 版本兼容性

Dubbo 3 Provider 可以同时支持 2.7 和 3.x Consumer 调用：

```yaml
# Provider 配置双注册
dubbo:
  application:
    register-mode: all
  protocol:
    name: dubbo      # 保留 Dubbo 协议兼容 2.7 Consumer
```

**兼容性测试矩阵**：

| Provider 版本 | Consumer 版本 | 接口级发现 | 应用级发现 |
|--------------|--------------|-----------|-----------|
| Dubbo 2.7 | Dubbo 2.7 | Y | N |
| Dubbo 3.x (all) | Dubbo 2.7 | Y | N |
| Dubbo 3.x (all) | Dubbo 3.x | Y | Y |
| Dubbo 3.x (instance) | Dubbo 2.7 | N | N |
| Dubbo 3.x (instance) | Dubbo 3.x | N | Y |

# 6. 常见迁移问题

## 6.1 No provider available 但服务已注册

**原因**：Consumer 使用应用级发现，但 Provider 只注册了接口级服务。

**解决**：Provider 开启双注册 `register-mode: all`。

## 6.2 元数据上报失败

**现象**：

```
WARN [DubboMetadataReport] Failed to publish metadata
```

**解决**：

```yaml
# 检查元数据中心配置
dubbo:
  metadata-report:
    address: nacos://127.0.0.1:8848  # 必须与应用注册中心一致
    parameters:
      namespace: dev
```

## 6.3 @Service 与 @DubboService 冲突

Spring Boot 3.x 中 `@Service` 已弃用，必须迁移到 `@DubboService`。否则服务导出可能失败。

## 6.4 Dubbo 协议调用被 Triple 拦截

Dubbo 3 中，如果 Provider 只暴露了 Triple 协议，Dubbo 协议 Consumer 无法调用。

**解决**：Provider 同时暴露两个协议：

```yaml
dubbo:
  protocols:
    dubbo-protocol:          # 兼容 2.7 Consumer
      name: dubbo
      port: 20880
    triple-protocol:         # 供 3.x Consumer
      name: triple
      port: 50051
```

## 6.5 序列化兼容性

Dubbo 2.7 默认使用 `hessian2`，Dubbo 3 默认使用 `fastjson2`：

```yaml
# 确保 Consumer 和 Provider 序列化方式一致
dubbo:
  provider:
    serialization: hessian2   # 与 2.7 保持一致过渡
  consumer:
    serialization: hessian2
```

## 6.6 线程模型变更

Dubbo 3 默认使用 `all` dispatcher（所有消息进业务线程池），Dubbo 2.7 默认使用 `all`（相同）。但如果之前自定义了 dispatcher，需确认行为一致。

```yaml
dubbo:
  protocol:
    dispatcher: all  # all | direct | message | execution | connection
```

# 7. 迁移检查清单

| 检查项 | 说明 | 状态 |
|--------|------|------|
| JDK 版本 | 升级到 JDK 17+ | ☐ |
| Spring Boot 版本 | 升级到 3.3.x | ☐ |
| javax → jakarta | 全局替换导入包 | ☐ |
| @Service → @DubboService | 注解迁移 | ☐ |
| @Reference → @DubboReference | 注解迁移 | ☐ |
| 服务发现模式 | 确认 register-mode | ☐ |
| 元数据中心 | 配置 metadata-report | ☐ |
| 序列化方式 | Provider/Consumer 统一 | ☐ |
| 协议兼容 | 确认 Dubbo/Triple 协议选择 | ☐ |
| 注册中心版本 | Nacos 2.1+ / ZK 3.8+ | ☐ |
| 监控/治理 | Dubbo Admin 升级到对应版本 | ☐ |
| 集成组件 | Sentinel/Seata/SkyWalking 版本对齐 | ☐ |
| 全链路回归测试 | 核心业务流程验证 | ☐ |
