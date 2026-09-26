
# 1. 聚合与父工程

- 聚合（Aggregator）：负责管理模块列表，`packaging` 必须是 `pom`。
- 父工程（Parent）：负责继承与统一配置。
- 通常一个 POM 同时承担聚合与父工程角色。

| 特性 | 聚合（Aggregator） | 父工程（Parent） |
|------|-------------------|------------------|
| 关键字 | `<modules>` | `<parent>` |
| `packaging` | 必须 `pom` | 必须 `pom` |
| 方向 | 父 -> 子（知道子模块） | 子 -> 父（引用父 POM） |
| 作用 | 一起构建多个模块 | 继承统一配置 |
| 互相感知 | 父知道所有子模块 | 子只知道父，不知兄弟模块 |
| 可单独存在 | 是（纯聚合 POM） | 是（纯父 POM 无子模块） |

```xml
<project>
	<groupId>com.example</groupId>
	<artifactId>parent</artifactId>
	<version>1.0.0</version>
	<packaging>pom</packaging>

	<modules>
		<module>module-a</module>
		<module>module-b</module>
	</modules>
</project>
```

# 2. Reactor 构建顺序

- Maven 会根据模块依赖关系确定构建顺序。
- 如未显式依赖，顺序由 `modules` 列表决定。

常用命令：
```bash
mvn -pl module-a -am install
```

- `-pl` 只构建指定模块
- `-am` 同时构建其依赖模块

## 2.1 Reactor 参数组合实战

| 参数组合 | 含义 | 示例 |
|----------|------|------|
| `-pl module-a` | 只构建 module-a | `mvn -pl :service-impl install` |
| `-pl module-a -am` | 构建 module-a 及其上游依赖 | `mvn -pl :web -am install` |
| `-pl module-a -amd` | 构建 module-a 及其下游依赖 | `mvn -pl :common -amd install` |
| `-rf module-a` | 从 module-a 开始继续构建（恢复用） | `mvn -rf :service-impl install` |
| `-N` | 只构建当前模块（非递归） | `mvn -N install` |

- 常见组合：
```bash
# 只构建 web 模块及其依赖链（跳过无关模块）
mvn -pl :web -am clean install

# 从失败模块恢复（跳过前面已成功的模块）
mvn -rf :service-impl install

# 只执行当前目录的 POM（不递归子模块）
mvn -N clean package
```

# 3. 模块依赖设计

- 避免循环依赖。
- 公共代码下沉到 `common` 模块。
- API 与实现分离（如 `service-api` / `service-impl`）。
- 模块依赖建议单向流动（core -> service -> web）。

## 3.1 企业分层模块模式

常见分层模式：
```
parent (pom, 聚合 + 父工程)
├── common          (公共工具、基础实体、异常)
├── domain          (领域模型、DDD 聚合根)
├── repository       (数据访问、MyBatis/JPA)
├── service-api     (服务接口、DTO、RPC 定义)
├── service-impl    (业务实现)
└── web             (Controller、拦截器、过滤器)
```

依赖方向：
```
web -> service-api -> service-impl -> domain
                     service-impl -> repository -> domain
                                               repository -> common

单向流动，严禁反向依赖。
```

## 3.2 API/Impl 分离模式

- `xxx-api`：只含接口、DTO、枚举——轻量，供外部模块引用。
- `xxx-impl`：实现类，编译期不暴露给其他模块（运行时通过 Spring DI 注入）。
- 好处：
	- 减少编译传递（api 模块变更不强制 impl 的消费者重新编译）。
	- RPC 场景隔开客户端与服务端依赖。

## 3.3 循环依赖检测与解决

**检测方法：**
```bash
# 通过依赖树排查
mvn dependency:tree

# Maven Reactor 也会在构建时检测循环依赖并报错
```

常见报错：
```
[ERROR] The projects in the reactor contain a cyclic reference: ...
```

**解决方案：**
1. **提取公共模块**：将共同依赖的类型/工具提取到 `common` 或新建 `base` 模块。
2. **引入中间接口模块**：A 和 B 互相依赖 → 提取共同接口到 C，A 和 B 都依赖 C。
3. **调整分层**：检查是否不合理的分层导致循环（如 domain 依赖了 service）。
4. **使用 Optional 依赖**：仅在极端情况下，通过 `<optional>true</optional>` 弱化依赖方向（不推荐）。

# 4. 版本与属性策略

- 父工程统一版本，子模块不再重复声明版本。
- 使用属性集中管理：

```xml
<properties>
	<java.version>21</java.version>
	<spring.version>6.1.12</spring.version>
</properties>
```

- Maven CI 友好版本写法（适合流水线自动发版）：
```xml
<version>${revision}${changelist}</version>

<properties>
	<revision>1.0.0</revision>
	<changelist>-SNAPSHOT</changelist>
</properties>
```

# 5. 共享插件管理

- 用 `pluginManagement` 统一插件版本与配置。
- 子模块在 `plugins` 中引用即可。

```xml
<build>
	<pluginManagement>
		<plugins>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-compiler-plugin</artifactId>
				<version>3.11.0</version>
			</plugin>
		</plugins>
	</pluginManagement>
</build>
```

# 6. 常用 Reactor 参数

- 只构建当前模块：
```bash
mvn -pl :module-a -am install
```

- 从失败模块继续：
```bash
mvn -rf :module-a install
```

- 多线程构建：
```bash
mvn -T 1C install
```

# 7. 常见问题

- 依赖版本不一致：统一在父 POM 中锁定。
- 循环依赖：拆分公共模块或调整依赖方向。

# 8. 常见问题排查清单

- 模块未构建：检查父 POM 的 `modules` 是否包含模块名。
- 构建顺序异常：确认模块依赖关系是否声明。
- 只想构建单个模块：使用 `-pl` 与 `-am`。
- 构建失败后重试：使用 `-rf` 从失败模块继续。
- 子模块找不到父 POM：检查 `relativePath` 或父 POM 是否已安装。

# 9. 实战案例：多模块工程结构

目标：构建一个典型的企业分层结构。

模块示例：
```
company-parent
├── common
├── service-api
├── service-impl
└── web
```

父 POM 管理依赖版本与插件版本；子模块只负责业务代码。
