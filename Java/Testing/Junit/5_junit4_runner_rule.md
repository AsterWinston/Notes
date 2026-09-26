# 1. Runner 机制

JUnit 4 使用 Runner 驱动测试类执行，`@RunWith` 可替换默认 Runner。

## 1.1 默认 Runner

- `BlockJUnit4ClassRunner`：默认执行器。
- 通过反射获取测试方法并执行 Statement。

## 1.2 常见 Runner

- `Parameterized`：参数化测试
- `Suite`：测试套件
- `SpringJUnit4ClassRunner`：Spring Test 集成

```java
@RunWith(Parameterized.class)
public class UserParamTest { }
```

## 1.3 Runner 的局限

- 一个测试类只能有一个 Runner。
- 多种能力叠加时需要自定义 Runner。

# 2. Rule 机制

Rule 是 JUnit 4 的可组合扩展点，基于装饰器模式。

## 2.1 Rule 类型

- `MethodRule`：已废弃
- `TestRule`：推荐使用

## 2.2 `@Rule` 与 `@ClassRule`

- `@Rule`：每个测试方法生效
- `@ClassRule`：类级别共享

## 2.3 常用内置 Rule

- `TemporaryFolder`：临时文件
- `ExpectedException`：异常断言
- `Timeout`：超时控制
- `ExternalResource`：外部资源生命周期
- `RuleChain`：组合多个 Rule

```java
@Rule
public TemporaryFolder folder = new TemporaryFolder();
```

# 3. 自定义 Rule

```java
public class StopwatchRule implements TestRule {
    @Override
    public Statement apply(Statement base, Description description) {
        return new Statement() {
            @Override
            public void evaluate() throws Throwable {
                long start = System.currentTimeMillis();
                try {
                    base.evaluate();
                } finally {
                    long cost = System.currentTimeMillis() - start;
                    System.out.println(description.getDisplayName() + " cost=" + cost);
                }
            }
        };
    }
}
```

# 4. Runner 与 Rule 的选择

- **Runner**：用于改变执行模型（如参数化、套件、Spring 集成）。
- **Rule**：用于增强测试方法的执行过程（超时、资源管理）。

当需要多个功能同时生效时，优先使用 Rule；Runner 只在必要时使用。
