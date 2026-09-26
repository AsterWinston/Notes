
# 1. 配置管理概述

配置管理是 Nacos 的另一核心功能。它提供了一种中心化、外部化和动态化的方式来管理所有环境的配置，消除了配置变更时重新部署应用和服务的需求。

## 1.1 配置管理解决的问题

- **硬编码问题**：将配置从代码中分离，避免修改配置需要重新编译部署
- **多环境管理**：统一管理开发、测试、预发、生产多套环境配置
- **动态刷新**：配置变更实时生效，无需重启应用
- **版本回溯**：配置历史版本管理，支持一键回滚
- **权限控制**：配置的读写权限管理，防止误操作

# 2. 配置模型

## 2.1 三级配置模型

```
Namespace (命名空间 / 租户)
  └── Group (分组)
        └── DataID (配置ID)
              ├── Content (配置内容)
              ├── MD5 (内容摘要)
              ├── Type (格式类型)
              ├── Description (描述)
              └── Version History (历史版本)
```

## 2.2 Data ID 命名规则

Spring Cloud 应用中的 Data ID 组成规则：

```
${prefix}-${spring.profiles.active}.${file-extension}
```

- `prefix`：默认为 `spring.application.name`，可通过 `spring.cloud.nacos.config.prefix` 配置
- `spring.profiles.active`：当前环境对应的 Profile
- `file-extension`：配置内容的数据格式，通过 `spring.cloud.nacos.config.file-extension` 配置

**示例**：

```
order-service-dev.yaml      → 应用: order-service, 环境: dev, 格式: yaml
order-service.yaml           → 应用: order-service, 无环境, 格式: yaml
user-service-prod.properties → 应用: user-service, 环境: prod, 格式: properties
```

# 3. Spring Cloud 集成

## 3.1 依赖配置

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
    <version>2021.0.5.0</version>
</dependency>
```

## 3.2 Bootstrap 配置（重要）

Nacos 配置中心的数据需要在应用启动的早期阶段加载，因此必须放在 `bootstrap.yml/properties` 中：

```yaml
# bootstrap.yml
spring:
  application:
    name: order-service
  profiles:
    active: dev
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
        namespace: 4e4a1b4a-5b3c-4efc-8b8d-1f4a0c5e2d6f
        group: DEFAULT_GROUP
        file-extension: yaml
        prefix: ${spring.application.name}
        shared-configs:
          - data-id: common-config.yaml
            group: DEFAULT_GROUP
            refresh: true
        extension-configs:
          - data-id: database-config.yaml
            group: DATABASE_GROUP
            refresh: true
        refresh-enabled: true
        timeout: 3000
```

## 3.3 配置加载优先级

Nacos 配置加载遵循以下优先级（数字越大优先级越高）：

```
1. shared-configs[0]           (共享配置，最低优先级)
2. shared-configs[1]
3. ...
4. extension-configs[0]        (扩展配置)
5. extension-configs[1]
6. ${prefix}-${profile}.${ext} (当前环境应用配置)
7. ${prefix}.${ext}            (应用通用配置，最高优先级)
```

## 3.4 动态刷新配置

`@RefreshScope` 注解实现配置动态刷新：

```java
@RestController
@RequestMapping("/config")
@RefreshScope  // 标记此 Bean 的配置需要动态刷新
public class ConfigController {

    @Value("${user.name:default}")
    private String userName;

    @Value("${user.age:0}")
    private Integer age;

    @GetMapping("/user")
    public Map<String, Object> getUserConfig() {
        return Map.of("name", userName, "age", age);
    }
}
```

动态刷新流程：

```
Nacos Console 修改配置
        │
        ▼
Nacos Server 配置变更
        │
        ▼
Nacos Client 收到通知（长轮询 / gRPC 推送）
        │
        ▼
Spring Cloud Nacos 发布 RefreshEvent
        │
        ▼
ContextRefresher.refresh() 处理
        │
        ▼
销毁被 @RefreshScope 标记的 Bean
        │
        ▼
下次访问时重新创建 Bean（从 Environment 读取新配置值）
        │
        ▼
配置生效（无需重启）
```

# 4. 配置管理 API

## 4.1 REST API

```bash
# 发布/更新配置
curl -X POST 'http://127.0.0.1:8848/nacos/v1/cs/configs' \
  -d 'dataId=order-service-dev.yaml&group=DEFAULT_GROUP&content=server.port: 8080'

# 获取配置
curl -X GET 'http://127.0.0.1:8848/nacos/v1/cs/configs?dataId=order-service-dev.yaml&group=DEFAULT_GROUP'

# 删除配置
curl -X DELETE 'http://127.0.0.1:8848/nacos/v1/cs/configs?dataId=order-service-dev.yaml&group=DEFAULT_GROUP'

# 监听配置变更
curl -X POST 'http://127.0.0.1:8848/nacos/v1/cs/configs/listener' \
  -d 'Listening-Configs=order-service-dev.yaml%02DEFAULT_GROUP%02abc123%01'

# 查询配置历史版本
curl -X GET 'http://127.0.0.1:8848/nacos/v1/cs/history?dataId=order-service-dev.yaml&group=DEFAULT_GROUP'
```

## 4.2 Java SDK

```java
// 获取配置
String serverAddr = "127.0.0.1:8848";
String dataId = "order-service-dev.yaml";
String group = "DEFAULT_GROUP";
Properties properties = new Properties();
properties.put("serverAddr", serverAddr);

ConfigService configService = NacosFactory.createConfigService(properties);
String content = configService.getConfig(dataId, group, 5000);
System.out.println("Config content: " + content);

// 发布配置
configService.publishConfig(dataId, group, "server.port: 8081\nspring.datasource.url: jdbc:mysql://...");

// 删除配置
configService.removeConfig(dataId, group);

// 监听配置变更
configService.addListener(dataId, group, new Listener() {
    @Override
    public Executor getExecutor() {
        return null; // 使用默认线程池
    }
    
    @Override
    public void receiveConfigInfo(String configInfo) {
        System.out.println("Config changed: " + configInfo);
        // 处理配置变更逻辑
    }
});
```

# 5. 灰度发布（Beta 配置）

Nacos 支持配置的灰度发布，允许将新配置推送给特定 IP 列表的实例，实现配置层面的金丝雀发布。

## 5.1 测试灰度发布

```bash
# 发布 Beta 配置（仅对特定 IP 生效）
curl -X POST 'http://127.0.0.1:8848/nacos/v1/cs/configs' \
  -d 'dataId=order-service-dev.yaml&group=DEFAULT_GROUP&content=server.port: 8082&betaIps=192.168.1.10,192.168.1.11'

# 查询 Beta 配置
curl -X GET 'http://127.0.0.1:8848/nacos/v1/cs/configs?dataId=order-service-dev.yaml&group=DEFAULT_GROUP&beta=true'

# 停止 Beta（删除 Beta 配置，恢复正式配置）
curl -X DELETE 'http://127.0.0.1:8848/nacos/v1/cs/configs?dataId=order-service-dev.yaml&group=DEFAULT_GROUP&beta=true'
```

## 5.2 灰度发布流程

```
1. 创建 Beta 配置（指定灰度 IP）
        │
2. 灰度实例拉取 Beta 配置，验证功能正常
        │
3. 扩大灰度范围 / 全量发布（删除 Beta 标签，直接发布正式配置）
        │
4. 所有实例自动拉取新配置
```

# 6. 配置导入导出

## 6.1 单配置导入导出

```bash
# 导出配置（获取原始内容）
curl -X GET 'http://127.0.0.1:8848/nacos/v1/cs/configs?dataId=order-service-dev.yaml&group=DEFAULT_GROUP&show=all'
```

## 6.2 命名空间级别导出

Nacos 控制台支持一键导出/导入整个命名空间下的所有配置：

```
控制台 → 配置管理 → 配置列表 → 导出配置 (导出为 .zip 文件)
控制台 → 配置管理 → 配置列表 → 导入配置 (上传 .zip 文件)
```

导出的文件结构：

```
nacos_config_export_20240101120000.zip
├── DEFAULT_GROUP/
│   ├── order-service-dev.yaml
│   ├── order-service-prod.yaml
│   └── common-config.yaml
├── DATABASE_GROUP/
│   └── database-config.yaml
└── CUSTOM_GROUP/
    └── business-config.yaml
```

# 7. 共享配置与扩展配置

## 7.1 配置分类

| 类型 | 配置方式 | 场景 | 是否支持刷新 |
|------|---------|------|-------------|
| **主配置** | `${spring.application.name}-${profile}.${ext}` | 应用独有配置 | ✅ |
| **共享配置** | `spring.cloud.nacos.config.shared-configs` | 多个应用共享 | ✅ (默认 true) |
| **扩展配置** | `spring.cloud.nacos.config.extension-configs` | 额外定制配置 | ✅ (默认 true) |

## 7.2 共享配置示例

```yaml
# common-config.yaml（多个服务共享）
spring:
  redis:
    host: 127.0.0.1
    port: 6379
  rabbitmq:
    host: 127.0.0.1
    port: 5672
    username: admin
    password: admin

logging:
  level:
    com.example: DEBUG
```

## 7.3 配置源优先级演示

```yaml
# bootstrap.yml
spring:
  cloud:
    nacos:
      config:
        # 扩展配置（优先级 = 共享配置，高于共享配置）
        extension-configs[0]:
          data-id: log-config.yaml
          group: LOG_GROUP
          refresh: true
        extension-configs[1]:
          data-id: monitor-config.yaml
          group: MONITOR_GROUP
          refresh: true
        # 共享配置
        shared-configs[0]:
          data-id: common-config.yaml
          group: DEFAULT_GROUP
          refresh: true
```

# 8. 配置快照与本地缓存

## 8.1 客户端本地缓存机制

Nacos 客户端在首次获取配置后，会将配置内容缓存到本地文件系统：

```
${user.home}/nacos/config/
    └── fixed-127.0.0.1_8848_nacos/
        └── snapshot/
            └── DEFAULT_GROUP/
                ├── order-service-dev.yaml
                ├── common-config.yaml
                └── database-config.yaml
```

## 8.2 缓存容灾

当 Nacos 服务端不可用时，客户端会使用本地缓存的配置，保证应用不会因配置服务宕机而启动失败：

```java
// ClientWorker.java
public String getServerConfig(String dataId, String group, String tenant, long readTimeout) {
    // 1. 尝试从服务端获取
    try {
        String content = agent.httpGet(configPath, headers, params, ...);
        if (content != null) {
            // 写入本地快照
            LocalConfigInfoProcessor.saveSnapshot(envName, dataId, group, tenant, content);
            return content;
        }
    } catch (Exception e) {
        log.warn("Failed to get config from server", e);
    }
    
    // 2. 服务端不可用 → 读取本地快照
    String localConfig = LocalConfigInfoProcessor.getFailover(envName, dataId, group, tenant);
    if (localConfig != null) {
        log.warn("Using local snapshot for config: {}", dataId);
        return localConfig;
    }
    
    // 3. 无快照 → 返回 null (应用使用默认值)
    return null;
}
```

# 9. 配置加密

Nacos 本身不提供内置的配置加密功能，但可以通过以下方式实现：

## 9.1 集成 jasypt

```xml
<dependency>
    <groupId>com.github.ulisesbocchio</groupId>
    <artifactId>jasypt-spring-boot-starter</artifactId>
    <version>3.0.5</version>
</dependency>
```

```yaml
# Nacos 中存储的加密配置
jasypt:
  encryptor:
    password: ${JASYPT_ENCRYPTOR_PASSWORD}  # 通过环境变量注入
    algorithm: PBEWithMD5AndDES

spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb
    username: ENC(3r2i8JkLmNpQ0sT1uV...)
    password: ENC(XyZ9aBcDeFgHiJkLmN...)
```

## 9.2 自定义加密实现

```java
@Component
public class EncryptablePropertyResolver implements EnvironmentPostProcessor {
    
    @Override
    public void postProcessEnvironment(ConfigurableEnvironment environment, 
                                        SpringApplication application) {
        // 遍历所有属性源，解密 ENC() 包裹的值
        for (PropertySource<?> propertySource : environment.getPropertySources()) {
            if (propertySource instanceof EnumerablePropertySource) {
                EnumerablePropertySource<?> source = (EnumerablePropertySource<?>) propertySource;
                for (String key : source.getPropertyNames()) {
                    Object value = source.getProperty(key);
                    if (value instanceof String && ((String) value).startsWith("ENC(")) {
                        String encrypted = ((String) value).substring(4, ((String) value).length() - 1);
                        String decrypted = AESUtils.decrypt(encrypted);
                        System.setProperty(key, decrypted);
                    }
                }
            }
        }
    }
}
```

# 10. 配置管理最佳实践

## 10.1 配置分类原则

| 配置类型 | 建议内容 | 建议 Group |
|---------|---------|-----------|
| 基础设施配置 | 数据库、Redis、MQ 等 | INFRA_GROUP |
| 业务配置 | 业务开关、阈值等 | BUSINESS_GROUP |
| 框架配置 | 日志、监控、线程池等 | FRAMEWORK_GROUP |
| 安全配置 | Token 密钥、加密盐值等 | SECURITY_GROUP |

## 10.2 配置安全原则

1. **敏感信息不上传 Nacos**（或加密后上传）
2. **最小权限原则**：通过 Namespace 控制不同角色的配置访问权限
3. **配置审核**：利用 Nacos 鉴权机制，对配置变更进行审批
4. **版本管理**：保留历史版本，便于回滚

## 10.3 配置治理建议

```yaml
# 推荐的多服务配置结构化方案
spring:
  cloud:
    nacos:
      config:
        # 第一层：全局基础设施配置（所有服务共享）
        shared-configs[0]:
          data-id: global-infra-config.yaml
          group: INFRA_GROUP
        # 第二层：业务域共享配置（如订单域共享配置）
        shared-configs[1]:
          data-id: order-domain-config.yaml
          group: DOMAIN_GROUP
        # 第三层：应用自有配置（自动加载）
        # ${application.name}-${profile}.yaml
```
