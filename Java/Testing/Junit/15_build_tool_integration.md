# 1. Maven 集成

JUnit 5 推荐使用 Surefire 3.x + Failsafe 3.x。

## 1.1 Surefire 单元测试

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-surefire-plugin</artifactId>
  <version>3.2.5</version>
  <configuration>
    <useModulePath>false</useModulePath>
  </configuration>
</plugin>
```

## 1.2 Failsafe 集成测试

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-failsafe-plugin</artifactId>
  <version>3.2.5</version>
</plugin>
```

## 1.3 测试类命名约定

- Surefire 默认识别 `*Test`、`*Tests`、`*TestCase`
- Failsafe 默认识别 `*IT`、`*ITCase`

# 2. Gradle 集成

## 2.1 启用 JUnit Platform

```groovy
test {
    useJUnitPlatform()
}
```

## 2.2 选择性执行

```groovy
test {
    useJUnitPlatform {
        includeTags 'fast'
        excludeTags 'slow'
    }
}
```

# 3. 并发与隔离配置

## 3.1 Maven 并发

```xml
<configuration>
  <parallel>classes</parallel>
  <threadCount>4</threadCount>
</configuration>
```

## 3.2 Gradle 并发

```groovy
test {
    maxParallelForks = 4
}
```

# 4. 依赖管理建议

- 统一管理 JUnit 版本，避免多版本冲突。
- 若使用 JUnit 4 与 5 共存，确保 Vintage 与 Jupiter 版本一致。
