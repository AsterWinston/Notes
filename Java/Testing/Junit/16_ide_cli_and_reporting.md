# 1. IDE 运行与调试

主流 IDE（IntelliJ IDEA、Eclipse、VS Code）均支持 JUnit 5。

## 1.1 运行方式

- 运行单个测试方法
- 运行测试类
- 运行测试包/模块

## 1.2 调试建议

- 设置断点观察断言失败原因
- 对不可重复问题使用日志或 `@RepeatedTest`

# 2. CLI 执行

JUnit Platform 提供 ConsoleLauncher：

```bash
java -jar junit-platform-console-standalone-1.10.2.jar \
  --class-path target/test-classes:target/classes \
  --scan-class-path
```

## 2.1 过滤与标签

```bash
--include-tag fast --exclude-tag slow
```

# 3. 报告与输出

## 3.1 JUnit XML

构建工具默认生成 JUnit XML，可被 CI 系统解析。

## 3.2 失败诊断信息

- 保留堆栈信息
- 输出关键上下文（输入参数、配置）

# 4. 代码覆盖率

常见工具：JaCoCo、Cobertura。

```xml
<plugin>
  <groupId>org.jacoco</groupId>
  <artifactId>jacoco-maven-plugin</artifactId>
  <version>0.8.11</version>
</plugin>
```

**建议**：覆盖率作为趋势指标，而不是唯一指标。
