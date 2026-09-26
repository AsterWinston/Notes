
# 1. 泛化调用概述

泛化调用（Generic Service）允许**不依赖服务接口的 .class 文件**即可调用远程 Dubbo 服务。Consumer 不需要引入 Provider 的 API 模块。

```
传统调用：
  Consumer                                 Provider
  UserService userService (强类型接口)  →  UserServiceImpl

泛化调用：
  Consumer                                 Provider
  GenericService (通用接口)             →  任意 Dubbo 服务
  $invoke(method, paramTypes, args)
```

**使用场景**：

- API 网关/网关平台：动态转发，不需要依赖后端服务 jar
- 测试平台：对任意 Dubbo 服务发起调用
- 服务编排引擎：动态调用，避免硬编码接口依赖

# 2. 泛化调用模式

| 模式 | 说明 | 特点 |
|------|------|------|
| **pojo** | POJO 对象传参和返回 | 需要同时有 API jar，泛化实现自动转换 |
| **map** | Map 表示所有参数和返回值 | 不需要任何 jar，最通用 |
| **protobuf-json** | Protobuf 消息的 JSON 表示 | 用于 Triple 协议 |

# 3. GenericService 接口

```java
public interface GenericService {
    /**
     * 泛化调用
     * @param method     方法名
     * @param parameterTypes  参数类型数组 (如: ["java.lang.Long", "org.example.User"])
     * @param args       参数值数组
     * @return 返回值
     */
    Object $invoke(String method, String[] parameterTypes, Object[] args) 
        throws GenericException;
}
```

# 4. Map 模式（最通用）

## 4.1 引用服务

```java
public class GenericInvokeDemo {
    public static void main(String[] args) {
        // 通过 API 方式引用泛化服务
        ApplicationConfig application = new ApplicationConfig();
        application.setName("generic-consumer");

        RegistryConfig registry = new RegistryConfig();
        registry.setAddress("nacos://127.0.0.1:8848");

        ReferenceConfig<GenericService> reference = new ReferenceConfig<>();
        reference.setApplication(application);
        reference.setRegistry(registry);
        reference.setInterface("org.example.UserService");  // 接口全限定名
        reference.setVersion("1.0.0");
        reference.setGroup("dev");
        reference.setGeneric("true");                       // 关键：开启泛化调用

        GenericService genericService = reference.get();

        // 调用 getUser(Long id)
        Object result = genericService.$invoke(
            "getUser",
            new String[]{"java.lang.Long"},
            new Object[]{1L}
        );
        
        // result 是 Map: {"id":1, "name":"user-1", "age":25}
        System.out.println("Result: " + result);
    }
}
```

## 4.2 传递复杂参数

```java
// 调用 createUser(User user)
// User: {id=Long, name=String, age=Integer}

Map<String, Object> user = new HashMap<>();
user.put("id", 100L);
user.put("name", "test-user");
user.put("age", 30);
user.put("class", "org.example.api.User");  // 指定目标类型（可选）

Object result = genericService.$invoke(
    "createUser",
    new String[]{"org.example.api.User"},
    new Object[]{user}
);

// result 也是 Map
Map<String, Object> createdUser = (Map<String, Object>) result;
System.out.println("Created user id: " + createdUser.get("id"));
```

## 4.3 返回 List/Array

```java
// listUsers() → List<User>
Object result = genericService.$invoke(
    "listUsers",
    new String[]{},
    new Object[]{}
);

// result 是 List<Map>
List<Map<String, Object>> users = (List<Map<String, Object>>) result;
for (Map<String, Object> user : users) {
    System.out.println(user.get("name"));
}
```

# 5. POJO 模式

有 Provider 的 API jar，但希望用泛化方式调用（结果自动转 POJO）：

```java
ReferenceConfig<GenericService> reference = new ReferenceConfig<>();
reference.setInterface("org.example.UserService");
reference.setGeneric("true");

// 调用获取 Map 结果后手动转换或用工具类转换
GenericService genericService = reference.get();
Object result = genericService.$invoke("getUser", 
    new String[]{"java.lang.Long"}, new Object[]{1L});

// 使用 PojoUtils 将 Map 转 POJO
User user = PojoUtils.realize(result, User.class);
```

# 6. Protobuf-JSON 模式（Triple 协议）

```java
ReferenceConfig<GenericService> reference = new ReferenceConfig<>();
reference.setInterface("org.example.UserService");
reference.setProtocol("tri");
reference.setGeneric("protobuf-json");

GenericService genericService = reference.get();

// 参数和返回值都是 JSON 字符串
// 对应 Protobuf Message 的 JSON 表示
String result = (String) genericService.$invoke(
    "getUser",
    new String[]{"long"},
    new Object[]{1L}
);
// result: "{\"id\":\"1\",\"name\":\"user-1\",\"age\":25}"
```

# 7. 泛化调用源码分析

## 7.1 ReferenceConfig 创建泛化代理

```java
// ReferenceConfig 中泛化调用的关键代码
private T createProxy(Map<String, String> map) {
    if (isGeneric) {
        // 强制将接口设置为 GenericService
        refProtocol.refer(GenericService.class, url);
    }
    // ...
}
```

## 7.2 GenericImplFilter（Consumer 端）

```java
@Activate(group = CONSUMER, order = 20000)
public class GenericImplFilter implements Filter {

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) 
            throws RpcException {
        
        String generic = invoker.getUrl().getParameter("generic");
        
        if ("true".equals(generic)) {
            // 将 GenericService.$invoke() 的调用转换为普通的 RPC 调用
            
            // 1. 从 $invoke 参数中提取方法名、参数类型、参数值
            RpcInvocation rpcInvocation = new RpcInvocation(invocation);
            
            String methodName = (String) invocation.getArguments()[0];
            String[] parameterTypes = (String[]) invocation.getArguments()[1];
            Object[] args = (Object[]) invocation.getArguments()[2];
            
            rpcInvocation.setMethodName(methodName);
            rpcInvocation.setParameterTypes(parameterTypes);
            rpcInvocation.setArguments(args);
            
            return invoker.invoke(rpcInvocation);
        }
        
        return invoker.invoke(invocation);
    }
}
```

## 7.3 GenericFilter（Provider 端）

```java
@Activate(group = PROVIDER, order = 30000)
public class GenericFilter implements Filter {

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) 
            throws RpcException {
        
        String generic = invoker.getUrl().getParameter("generic");
        
        if ("true".equals(generic)) {
            // Provider 端处理泛化请求
            
            // 1. 获取目标服务的真实接口
            Class<?> realInterface = invoker.getInterface();
            
            // 2. 获取实际方法
            Method method = realInterface.getMethod(
                invocation.getMethodName(),
                getParameterTypes(invocation.getParameterTypes())
            );
            
            // 3. 将 Map 参数转换为 POJO 参数
            Class<?>[] parameterTypes = method.getParameterTypes();
            Object[] args = new Object[parameterTypes.length];
            for (int i = 0; i < parameterTypes.length; i++) {
                if (invocation.getArguments()[i] instanceof Map) {
                    // Map → POJO 转换
                    args[i] = PojoUtils.realize(
                        invocation.getArguments()[i], parameterTypes[i]);
                } else {
                    args[i] = invocation.getArguments()[i];
                }
            }
            
            invocation.setArguments(args);
            Result result = invoker.invoke(invocation);
            
            // 4. 将 POJO 返回值转为 Map
            if (result.hasException()) {
                return result;
            }
            Object value = result.getValue();
            if (value != null && !(value instanceof Map)) {
                value = PojoUtils.generalize(value);
            }
            return new RpcResult(value);
        }
        
        return invoker.invoke(invocation);
    }
}
```

## 7.4 PojoUtils —— Map/POJO 互相转换

```java
public class PojoUtils {

    // POJO → Map（泛化）
    public static Object generalize(Object pojo) {
        if (pojo instanceof Map) {
            return pojo;
        }
        
        Map<String, Object> map = new HashMap<>();
        for (Field field : pojo.getClass().getDeclaredFields()) {
            field.setAccessible(true);
            Object value = field.get(pojo);
            if (value != null) {
                // 递归转换嵌套 POJO
                map.put(field.getName(), generalize(value));
            }
        }
        return map;
    }

    // Map → POJO（具体化）
    public static Object realize(Object pojo, Class<?> type) {
        if (pojo instanceof Map) {
            Object instance = type.newInstance();
            Map<String, Object> map = (Map<String, Object>) pojo;
            
            for (Field field : type.getDeclaredFields()) {
                Object value = map.get(field.getName());
                if (value != null) {
                    // 递归转换嵌套 Map
                    Object realizedValue = realize(value, field.getType());
                    field.setAccessible(true);
                    field.set(instance, realizedValue);
                }
            }
            return instance;
        }
        return pojo;
    }
}
```

# 8. 泛化服务（Provider 端）

Provider 也可以暴露泛化服务，即不依赖具体接口实现：

```java
@DubboService(interfaceName = "org.example.GenericEchoService")
public class MyGenericService implements GenericService {

    @Override
    public Object $invoke(String method, String[] parameterTypes,
            Object[] args) throws GenericException {
        
        if ("echo".equals(method)) {
            return "Echo: " + args[0];
        }
        if ("ping".equals(method)) {
            return "pong";
        }
        throw new GenericException("Unknown method: " + method);
    }
}
```

Consumer 调用：

```java
ReferenceConfig<GenericService> reference = new ReferenceConfig<>();
reference.setInterface("org.example.GenericEchoService");
reference.setGeneric("true");

GenericService service = reference.get();
Object result = service.$invoke("echo",
    new String[]{"java.lang.String"}, new Object[]{"hello"});
// result: "Echo: hello"
```

# 9. 结合 Spring 的泛化引用

```xml
<!-- 或在 application.yml 中用 Bean 方式 -->
<dubbo:reference id="genericUserService"
    interface="org.example.UserService"
    generic="true"
    version="1.0.0"/>
```

```java
@Autowired
@Qualifier("genericUserService")
private GenericService genericUserService;

public void invoke() {
    Object result = genericUserService.$invoke("getUser",
        new String[]{"java.lang.Long"}, new Object[]{1L});
}
```

# 10. 注意事项

1. **性能**：Map 模式有额外的 Map/POJO 转换开销，性能敏感场景用 POJO 模式或直接引用
2. **类型安全**：参数类型字符串容易出错，建议使用常量管理
3. **序列化兼容**：泛化返回的 Map 可能包含嵌套 Map/List，需递归处理
4. **隐式参数**：通过 `RpcContext` 传递的隐式参数在泛化调用中同样生效
