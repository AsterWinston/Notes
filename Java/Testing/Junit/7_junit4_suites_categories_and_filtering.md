# 1. 测试套件（Suite）

Suite 用于组织多个测试类的批量执行。

## 1.1 基本用法

```java
@RunWith(Suite.class)
@Suite.SuiteClasses({
    UserServiceTest.class,
    OrderServiceTest.class
})
public class ServiceTestSuite { }
```

## 1.2 适用场景

- 回归测试集合
- 按模块分组执行

# 2. Category 分类

Category 允许给测试打标签并在执行时过滤。

## 2.1 定义与使用

```java
public interface SlowTest { }

@Category(SlowTest.class)
public class LargeDataTest { }
```

## 2.2 与 Suite 结合

```java
@RunWith(Categories.class)
@IncludeCategory(SlowTest.class)
@Suite.SuiteClasses({A.class, B.class})
public class SlowTestSuite { }
```

# 3. 执行过滤

## 3.1 Maven Surefire 过滤

```xml
<configuration>
  <groups>com.example.SlowTest</groups>
  <excludedGroups>com.example.FlakyTest</excludedGroups>
</configuration>
```

## 3.2 Gradle 过滤

```groovy
test {
    useJUnit {
        includeCategories 'com.example.SlowTest'
        excludeCategories 'com.example.FlakyTest'
    }
}
```

# 4. 常见问题

## 4.1 Category 只在 JUnit 4 生效

JUnit 5 使用 `@Tag` 替代 Category，需要迁移策略。

## 4.2 Suite 与 Runner 冲突

Suite 使用 Runner，会占用 `@RunWith`，导致无法与其他 Runner 组合。
