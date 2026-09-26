# 1. 迁移策略总览

迁移建议采用“渐进式”策略：先引入 JUnit Platform，再逐步替换 JUnit 4 测试。

## 1.1 三种策略

- **全量迁移**：一次性替换全部测试
- **渐进迁移**：新测试使用 JUnit 5，旧测试用 Vintage 执行
- **混合共存**：同时保留两套体系

# 2. 关键差异映射

## 2.1 注解与生命周期

- `@BeforeClass` -> `@BeforeAll`
- `@Before` -> `@BeforeEach`
- `@Ignore` -> `@Disabled`

## 2.2 断言与异常

- `@Test(expected=...)` -> `assertThrows`
- `ExpectedException Rule` -> `assertThrows`

## 2.3 Category 与 Tag

- `@Category` -> `@Tag`

## 2.4 Runner / Rule -> Extension

- `@RunWith` -> `@ExtendWith`
- `@Rule` / `@ClassRule` -> Extension + `@RegisterExtension`

# 3. 引入 JUnit Platform 配置

## 3.1 Maven

```xml
<dependency>
  <groupId>org.junit.jupiter</groupId>
  <artifactId>junit-jupiter</artifactId>
  <version>5.10.2</version>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.junit.vintage</groupId>
  <artifactId>junit-vintage-engine</artifactId>
  <version>5.10.2</version>
  <scope>test</scope>
</dependency>
```

## 3.2 Gradle

```groovy
dependencies {
    testImplementation 'org.junit.jupiter:junit-jupiter:5.10.2'
    testRuntimeOnly 'org.junit.vintage:junit-vintage-engine:5.10.2'
}

test {
    useJUnitPlatform()
}
```

# 4. 迁移过程中的常见问题

## 4.1 测试执行不到

- 构建工具未启用 `useJUnitPlatform`
- 依赖未引入 Jupiter 或 Vintage

## 4.2 扩展行为不一致

- Rule 与 Extension 的生命周期差异
- Runner 的组合能力变化

## 4.3 IDE 运行差异

- 需确保 IDE 启用 JUnit 5 Runner
- 旧插件可能不支持 Jupiter
