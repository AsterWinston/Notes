# 1.数据校验概述
```
在Web开发中，对用户输入的数据进行合法性校验是必不可少的一环。
数据校验分为前端校验和后端校验：
● 前端校验：在浏览器端通过JavaScript进行校验，提升用户体验，但可以被绕过
● 后端校验：在服务器端进行校验，是保障数据安全和完整性的最后一道防线

SpringMVC支持JSR-303/380（Bean Validation）规范，通过在实体类的属性上添加注解来实现声明式校验。
SpringMVC6默认集成的Bean Validation实现是Hibernate Validator。

核心依赖（SpringMVC 6已传递依赖引入了Hibernate Validator）：
<dependency>
    <groupId>org.hibernate.validator</groupId>
    <artifactId>hibernate-validator</artifactId>
    <version>8.0.1.Final</version>
</dependency>

使用数据校验的标准流程：
1. 在实体类属性上添加校验注解
2. 在Controller方法参数上添加@Valid注解
3. 使用BindingResult接收校验结果
4. 根据校验结果决定后续处理逻辑
```

# 2.常用校验注解
```
JSR-380（Bean Validation 2.0）提供了以下常用校验注解：

| 注解 | 说明 | 适用类型 |
|------|------|----------|
| @NotNull | 不能为null | 任意类型 |
| @NotEmpty | 不能为null且不能为空字符串 | CharSequence、Collection、Map、Array |
| @NotBlank | 不能为null且trim之后长度>0 | 仅String |
| @Size(min, max) | 长度在min~max之间 | CharSequence、Collection、Map、Array |
| @Min(value) | 最小值 | 数值类型 |
| @Max(value) | 最大值 | 数值类型 |
| @Range(min, max) | 范围（Hibernate扩展） | 数值类型 |
| @Email | 邮箱格式 | String |
| @Pattern(regexp) | 正则匹配 | String |
| @Digits(integer, fraction) | 整数和小数位数 | 数值类型 |
| @Positive | 正数 | 数值类型 |
| @PositiveOrZero | 非负数 | 数值类型 |
| @Negative | 负数 | 数值类型 |
| @NegativeOrZero | 非正数 | 数值类型 |
| @Past | 过去的日期 | Date、LocalDate等 |
| @Future | 未来的日期 | Date、LocalDate等 |
| @AssertTrue | 必须为true | boolean |
| @AssertFalse | 必须为false | boolean |

注意：@NotEmpty和@NotBlank只能在String类型上使用
@NotNull同时检查null，是三者中最"弱"的校验
```

# 3.基础使用：@Valid + BindingResult
```
第一步：在POJO类上添加校验注解

package com.powernode.springmvc.bean;

import jakarta.validation.constraints.*;

public class User {
    @NotNull(message = "用户ID不能为null")
    private Long id;

    @NotBlank(message = "用户名不能为空")
    @Size(min = 2, max = 20, message = "用户名长度必须在2~20之间")
    private String name;

    @NotBlank(message = "密码不能为空")
    @Size(min = 6, max = 20, message = "密码长度必须在6~20之间")
    private String password;

    @Email(message = "邮箱格式不正确")
    private String email;

    @Min(value = 0, message = "年龄不能小于0")
    @Max(value = 150, message = "年龄不能大于150")
    private Integer age;

    @Pattern(regexp = "^1[3-9]\\d{9}$", message = "手机号格式不正确")
    private String phone;

    // getter/setter、构造方法省略...
}

第二步：在Controller方法中使用@Valid开启校验

package com.powernode.springmvc.controller;

import com.powernode.springmvc.bean.User;
import jakarta.validation.Valid;
import org.springframework.stereotype.Controller;
import org.springframework.validation.BindingResult;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@Controller
public class UserController {

    @PostMapping("/user/save")
    public String save(@Valid User user, BindingResult bindingResult) {
        // 判断是否有校验错误
        if (bindingResult.hasErrors()) {
            // 获取所有字段校验错误
            List<FieldError> fieldErrors = bindingResult.getFieldErrors();
            for (FieldError fieldError : fieldErrors) {
                // 打印字段名和错误信息
                System.out.println(fieldError.getField() + " : " + fieldError.getDefaultMessage());
                // 输出示例：
                // name : 用户名长度必须在2~20之间
                // email : 邮箱格式不正确
            }
            // 回到表单页面，展示错误信息
            return "user_form";
        }
        // 校验通过，执行业务逻辑
        System.out.println("保存用户：" + user);
        return "ok";
    }
}

第三步：在前端页面显示错误信息

Thymeleaf模板中使用th:errors展示校验错误：
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>用户注册</title>
</head>
<body>
    <h1>用户注册</h1>
    <form th:action="@{/user/save}" method="post">
        用户名：<input type="text" name="name" th:value="${user.name}">
        <span th:if="${#fields.hasErrors('name')}" th:errors="*{name}" style="color: red;"></span><br>

        密码：<input type="password" name="password">
        <span th:if="${#fields.hasErrors('password')}" th:errors="*{password}" style="color: red;"></span><br>

        邮箱：<input type="text" name="email" th:value="${user.email}">
        <span th:if="${#fields.hasErrors('email')}" th:errors="*{email}" style="color: red;"></span><br>

        年龄：<input type="number" name="age" th:value="${user.age}">
        <span th:if="${#fields.hasErrors('age')}" th:errors="*{age}" style="color: red;"></span><br>

        手机号：<input type="text" name="phone" th:value="${user.phone}">
        <span th:if="${#fields.hasErrors('phone')}" th:errors="*{phone}" style="color: red;"></span><br>

        <input type="submit" value="提交">
    </form>
</body>
</html>

注意：
● @Valid和BindingResult必须紧挨着放在Controller方法参数中，中间不能有其他参数
● 如果Controller方法中没有声明BindingResult，校验失败时会直接抛出MethodArgumentNotValidException异常
● @Valid是jakarta.validation包下的（JavaEE规范），不是Spring的
```

# 4.@Validated：Spring增强版校验
```
@Validated是Spring提供的校验注解，相比@Valid，它增加了分组校验功能。

4.1 校验分组的概念：
在实际开发中，同一个POJO在不同场景下需要不同的校验规则。
例如：新增用户时ID可以为null（数据库自增），修改用户时ID不能为null。

定义分组接口：
package com.powernode.springmvc.bean;

// 新增场景的分组
public interface AddGroup {}

// 修改场景的分组
public interface UpdateGroup {}

在POJO中使用groups属性指定校验分组：
public class User {
    @NotNull(message = "用户ID不能为null", groups = UpdateGroup.class)  // 仅修改时校验
    private Long id;

    @NotBlank(message = "用户名不能为空", groups = {AddGroup.class, UpdateGroup.class})
    @Size(min = 2, max = 20, message = "用户名长度2~20", groups = {AddGroup.class, UpdateGroup.class})
    private String name;

    // ... 其他属性略
}

Controller中使用@Validated指定分组：
@PostMapping("/user/save")
public String save(@Validated(AddGroup.class) User user, BindingResult bindingResult) {
    // 只校验属于AddGroup分组的注解
    // ...
}

@PutMapping("/user/update")
public String update(@Validated(UpdateGroup.class) User user, BindingResult bindingResult) {
    // 只校验属于UpdateGroup分组的注解
    // ...
}

注意：
● 没有指定groups属性的校验注解属于Default分组（jakarta.validation.groups.Default）
● @Validated中指定分组后，只会校验该分组的注解，默认分组的注解不会被校验
● 如果要同时校验指定分组和默认分组，需要显式指定：
  @Validated({UpdateGroup.class, Default.class}) User user
```

# 5.自定义校验注解
```
当内置的校验注解不能满足需求时，可以自定义校验注解。
例如：定义一个"不能包含空格"的校验注解。

第一步：定义校验注解
package com.powernode.springmvc.validation;

import jakarta.validation.Constraint;
import jakarta.validation.Payload;
import java.lang.annotation.*;

@Target({ElementType.FIELD, ElementType.METHOD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = NoSpacesValidator.class)  // 指定校验器
public @interface NoSpaces {
    // 默认错误消息
    String message() default "不能包含空格";

    // 分组
    Class<?>[] groups() default {};

    // 负载信息（携带元数据）
    Class<? extends Payload>[] payload() default {};
}

第二步：实现校验器
package com.powernode.springmvc.validation;

import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;

public class NoSpacesValidator implements ConstraintValidator<NoSpaces, String> {

    @Override
    public void initialize(NoSpaces constraintAnnotation) {
        // 初始化方法，可以获取注解中的属性值
    }

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        if (value == null) {
            return true;  // null值由@NotNull负责校验
        }
        // 校验逻辑：不能包含空格
        return !value.contains(" ");
    }
}

第三步：使用自定义注解
public class User {
    @NoSpaces(message = "用户名不能包含空格")
    private String name;
    // ...
}
```

# 6.校验消息的国际化配置
```
校验注解的message属性可以直接写死错误信息，但更好的做法是配置到属性文件中。

6.1 创建校验消息文件（类根路径下）：
ValidationMessages.properties（默认语言）
ValidationMessages_zh_CN.properties（中文，如果默认是中文可不创建）
ValidationMessages_en_US.properties（英文）

内容示例（ValidationMessages.properties）：
user.name.notblank=用户名不能为空
user.name.size=用户名长度必须在{min}~{max}之间
user.email.invalid=邮箱格式不正确
user.age.range=年龄必须在{min}~{max}之间
user.phone.pattern=手机号格式不正确

6.2 在POJO中使用{key}引用消息：
public class User {
    @NotBlank(message = "{user.name.notblank}")
    private String name;

    @Size(min = 2, max = 20, message = "{user.name.size}")
    private String name;

    @Email(message = "{user.email.invalid}")
    private String email;

    @Min(value = 0, message = "{user.age.range}")
    @Max(value = 150, message = "{user.age.range}")
    private Integer age;
}

6.3 自定义消息插值：
SpringMVC会自动将{min}、{max}替换为@Size、@Min等注解中设置的值。
这样错误信息会动态生成，例如："用户名长度必须在2~20之间"。
```

# 7.全局统一处理校验异常
```
如果Controller方法中没有声明BindingResult参数，校验失败时会抛出：
● org.springframework.web.bind.MethodArgumentNotValidException（@Valid校验失败）
● org.springframework.validation.BindException（普通参数绑定失败）

推荐使用@ControllerAdvice统一处理校验异常：

@RestControllerAdvice
public class GlobalValidationHandler {

    /**
     * 处理@Valid校验失败的异常，返回JSON格式的错误信息
     */
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, Object>> handleValidationException(
            MethodArgumentNotValidException e) {
        
        Map<String, Object> result = new HashMap<>();
        result.put("code", 400);
        result.put("message", "参数校验失败");
        
        // 收集所有字段错误
        Map<String, String> fieldErrors = new HashMap<>();
        List<FieldError> errors = e.getBindingResult().getFieldErrors();
        for (FieldError error : errors) {
            fieldErrors.put(error.getField(), error.getDefaultMessage());
        }
        result.put("errors", fieldErrors);
        
        return ResponseEntity.badRequest().body(result);
    }
}

这样，无论是表单提交还是AJAX请求，校验失败都能得到统一的JSON响应格式。
前端可以根据返回的errors字段，将错误信息展示在各表单项旁边。

综合建议：
● 对于传统的表单提交页面，在Controller中使用BindingResult手动处理，返回错误页面
● 对于RESTful API，使用@ControllerAdvice全局异常处理，返回统一的错误JSON
● 校验注解的message建议配置到ValidationMessages.properties中，便于维护和国际化
```
