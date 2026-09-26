
# 1. 概述

## 1.1 什么是 Bean Validation

Bean Validation（Jakarta Bean Validation）是 JavaEE 的**声明式数据校验规范**（原 JSR-303 → JSR-349 → JSR-380），通过注解在模型上声明约束，由校验框架自动执行。

- 参考实现（RI）：**Hibernate Validator**
- 其他实现：Apache BVal

## 1.2 为什么需要它

```
传统手动校验：
  if (user.getName() == null || user.getName().isEmpty()) { throw ... }
  if (user.getAge() < 18) { throw ... }
  if (user.getEmail() == null || !email.matches(regex)) { throw ... }

声明式校验：
  @NotBlank @Email private String email;
```

- 校验逻辑集中到注解，代码更简洁、可复用、与业务解耦。
- 同一模型在不同入口（Web 层、服务层、JPA 层）自动校验。

# 2. 内置约束注解

## 2.1 常用约束

| 注解 | 作用 |
|------|------|
| `@NotNull` | 不能为 null |
| `@Null` | 必须为 null |
| `@NotEmpty` | 不能为 null 且不能为空（String/Collection/Map/数组） |
| `@NotBlank` | 不能为 null 且不能为空白字符串（仅 String） |
| `@Size(min, max)` | 长度/元素个数范围 |
| `@Min` / `@Max` | 数值最小值/最大值 |
| `@Positive` / `@PositiveOrZero` | 正数 / 非负 |
| `@Negative` / `@NegativeOrZero` | 负数 / 非正 |
| `@DecimalMin` / `@DecimalMax` | 精确数值范围（BigDecimal） |
| `@Digits(integer, fraction)` | 整数位/小数位限制 |
| `@Pattern(regexp)` | 正则匹配 |
| `@Email` | 邮箱格式 |
| `@Past` / `@PastOrPresent` | 过去时间 |
| `@Future` / `@FutureOrPresent` | 未来时间 |

## 2.2 用法示例

```java
public class UserDTO {

    @NotBlank(message = "用户名不能为空")
    @Size(min = 3, max = 20, message = "用户名长度 3~20")
    private String username;

    @NotBlank(message = "密码不能为空")
    @Pattern(regexp = "^(?=.*[A-Za-z])(?=.*\\d).{8,}$",
            message = "密码需含字母和数字且至少8位")
    private String password;

    @Email(message = "邮箱格式不正确")
    @NotBlank
    private String email;

    @Min(value = 18, message = "年龄不能小于18")
    @Max(value = 65, message = "年龄不能大于65")
    private Integer age;

    @NotNull
    @Past(message = "生日必须是过去时间")
    private LocalDate birthday;
}
```

# 3. 校验流程

## 3.1 编程式校验（Validator）

```java
ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
Validator validator = factory.getValidator();

Set<ConstraintViolation<UserDTO>> violations = validator.validate(userDTO);
if (!violations.isEmpty()) {
    for (ConstraintViolation<UserDTO> v : violations) {
        System.out.println(v.getPropertyPath() + ": " + v.getMessage());
    }
}
```

## 3.2 `ConstraintViolation` 常用方法

| 方法 | 说明 |
|------|------|
| `getMessage()` | 校验失败消息 |
| `getPropertyPath()` | 校验失败的属性路径 |
| `getInvalidValue()` | 非法值 |
| `getConstraintDescriptor()` | 约束元数据 |

## 3.3 分组校验（Groups）

不同场景校验不同规则：

```java
public interface Create { }
public interface Update { }

public class UserDTO {
    @Null(groups = Create.class, message = "新增时不能指定ID")
    @NotNull(groups = Update.class, message = "修改时必须指定ID")
    private Long id;

    @NotBlank(groups = Create.class)
    private String username;
}
```

```java
validator.validate(userDTO, Create.class);   // 新增分组
validator.validate(userDTO, Update.class);   // 修改分组
```

## 3.4 校验顺序与组序列

```java
@GroupSequence({BasicInfo.class, AdvancedInfo.class, Default.class})
public interface OrderedGroups { }
```

# 4. 级联校验（@Valid）

嵌套对象的校验需要使用 `@Valid`：

```java
public class OrderDTO {
    @NotNull
    private Long userId;

    @Valid                      // 级联校验 items 中每个元素的约束
    @NotEmpty
    private List<OrderItemDTO> items;
}

public class OrderItemDTO {
    @NotNull
    private Long productId;

    @Min(1)
    private int quantity;
}
```

- `@Valid` 会递归校验嵌套对象及集合/数组中的元素。
- JPA 实体字段也可使用（Hibernate 会在 `pre-persist`、`pre-update` 时自动校验）。

# 5. 自定义约束

## 5.1 自定义注解

```java
@Target({FIELD, METHOD, PARAMETER, ANNOTATION_TYPE})
@Retention(RUNTIME)
@Constraint(validatedBy = PhoneValidator.class)
@Documented
public @interface Phone {
    String message() default "手机号格式不正确";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

## 5.2 校验器实现

```java
public class PhoneValidator implements ConstraintValidator<Phone, String> {

    private static final Pattern PATTERN =
            Pattern.compile("^1[3-9]\\d{9}$");

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        if (value == null) {
            return true;   // null 交给 @NotNull 处理
        }
        return PATTERN.matcher(value).matches();
    }
}
```

## 5.3 组合约束

```java
@NotBlank
@Email
@Constraint(validatedBy = {})
@Target({FIELD, METHOD, PARAMETER})
@Retention(RUNTIME)
public @interface EmailOrEmpty { }
```

# 6. 手动触发校验（校验方法返回值/参数）

```java
@Validated
public class UserService {

    public void create(@Valid @NotNull UserDTO user) { }

    @NotNull
    public UserDTO get() { return null; }   // 返回值校验
}
```

- `@Validated` 注解在类上，方法参数/返回值使用 `@Valid` 触发校验。
- 校验失败抛 `ConstraintViolationException`。

# 7. 与 Web 框架集成

## 7.1 Spring MVC 集成

```java
@RestController
public class UserController {

    @PostMapping("/users")
    public Result create(@Valid @RequestBody UserDTO user) {
        // 校验失败由 @ExceptionHandler 统一处理
        return Result.ok();
    }
}
```

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Result handle(MethodArgumentNotValidException e) {
        String msg = e.getBindingResult().getFieldErrors().stream()
                .map(f -> f.getDefaultMessage())
                .collect(Collectors.joining(", "));
        return Result.error(msg);
    }
}
```

## 7.2 与 JPA 集成

- Hibernate 在实体持久化前自动触发校验。
- 需要在 `persistence.xml` 中开启（Hibernate 默认开启 bean validation）：

```xml
<property name="hibernate.validator.apply_to_ddl" value="true"/>
<property name="hibernate.validator.autoregister_listeners" value="true"/>
```

## 7.3 与 JSF 集成

- JSF 的 `f:validateBean` 标签可触发 Bean Validation。

# 8. 常见问题与最佳实践

| 问题/实践 | 说明 |
|-----------|------|
| 校验放在 DTO 而非实体 | 避免实体在不同场景约束冲突 |
| 消息用占位符 | `{min}`、`{max}`、`{value}` 动态拼接 |
| null 值处理 | 约束对 null 多数返回 true，用 `@NotNull` 单独控制 |
| 分组管理 | 不同操作（Create/Update）用分组隔离 |
| 性能 | 校验开销小，可放心使用 |
| 国际化 | `messages.properties` 配置不同语言错误消息 |

# 9. 依赖坐标

```xml
<dependency>
    <groupId>jakarta.validation</groupId>
    <artifactId>jakarta.validation-api</artifactId>
    <version>3.0.2</version>
</dependency>
<dependency>
    <groupId>org.hibernate.validator</groupId>
    <artifactId>hibernate-validator</artifactId>
    <version>8.0.1.Final</version>
</dependency>
```

# 10. 与 Spring 校验对照

| Bean Validation | Spring |
|-----------------|--------|
| `@NotBlank` 等约束 | 同（Spring 复用 Bean Validation） |
| `@Valid` | `@Valid` / `@Validated` |
| 分组 | `@Validated(XxxGroup.class)` |
| 异常 | `ConstraintViolationException` | `MethodArgumentNotValidException`、`BindException` |
| 手动校验 | `validator.validate()` | `Validator`（Spring 封装） |
