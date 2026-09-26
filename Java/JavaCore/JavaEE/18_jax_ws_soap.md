
# 1. 概述

## 1.1 什么是 JAX-WS

JAX-WS（Java API for XML Web Services）是 JavaEE 的 **SOAP Web 服务规范**（JSR-224），用于开发基于 SOAP 协议的 Web 服务。参考实现：Metro（RI）、Apache CXF 等。

```
┌──────────────┐   SOAP/XML    ┌──────────────┐
│  服务消费者   │ ────────────► │   Web 服务    │
│  (Client)    │ ◄──────────── │   (Service)  │
└──────────────┘               └──────────────┘
```

## 1.2 SOAP 与 REST 对比

| 对比 | SOAP（JAX-WS） | REST（JAX-RS） |
|------|----------------|----------------|
| 协议 | SOAP（XML 信封） | HTTP 方法 + 资源 |
| 消息格式 | XML | JSON/XML/文本 |
| WSDL 描述 | 有 | 无（可用 OpenAPI） |
| 契约严格度 | 强（Schema） | 弱 |
| 典型场景 | 金融、电信、企业集成 | Web/移动端/微服务 |
| 传输 | HTTP/SMTP/JMS 等 | HTTP |
| 现状 | 存量系统 | 主流 |

## 1.3 SOAP 消息结构

```xml
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
    <soap:Header>
        <!-- 认证/事务头（可选） -->
    </soap:Header>
    <soap:Body>
        <!-- 业务数据 -->
    </soap:Body>
</soap:Envelope>
```

# 2. 自底向上开发（契约优先）

## 2.1 服务接口

```java
@WebService
public interface CalculatorService {

    @WebMethod
    int add(@WebParam(name = "a") int a, @WebParam(name = "b") int b);
}
```

## 2.2 服务实现

```java
@WebService(endpointInterface = "com.example.CalculatorService")
public class CalculatorServiceImpl implements CalculatorService {

    @Override
    public int add(int a, int b) {
        return a + b;
    }
}
```

## 2.3 发布服务（JavaSE 方式）

```java
public class Server {
    public static void main(String[] args) {
        Endpoint.publish("http://localhost:8080/calc",
                new CalculatorServiceImpl());
        System.out.println("服务已发布");
    }
}
```

- 访问 `http://localhost:8080/calc?wsdl` 可查看 WSDL。
- JavaEE 环境部署时用注解注册，无需 `Endpoint.publish`。

# 3. 常用注解

| 注解 | 作用 |
|------|------|
| `@WebService` | 类/接口标记为 Web 服务 |
| `@WebMethod` | 标记为服务方法 |
| `@WebParam` | 方法参数描述 |
| `@WebResult` | 返回值描述 |
| `@XmlType` | 复杂类型 |
| `@WebServiceClient` | 客户端生成标记（wsimport 生成） |
| `@SOAPBinding` | 绑定方式（document/rpc、literal/encoded） |

```java
@WebService
public interface OrderWs {

    @WebMethod(operationName = "createOrder")
    @WebResult(name = "orderId")
    Long createOrder(
            @WebParam(name = "order") Order order,
            @WebParam(name = "token", header = true) String token);
}
```

# 4. 客户端开发

## 4.1 工具生成客户端（wsimport）

```bash
wsimport -keep -p com.example.client http://localhost:8080/calc?wsdl
```

生成客户端后调用：

```java
CalculatorServiceImplService service = new CalculatorServiceImplService();
CalculatorService port = service.getCalculatorServiceImplPort();
int result = port.add(1, 2);
```

## 4.2 动态调用（不使用生成代码）

```java
QName serviceName = new QName("http://example.com/", "CalculatorService");
QName portName = new QName("http://example.com/", "CalculatorServiceImplPort");

Service service = Service.create(serviceName);
service.addPort(portName, SOAPBinding.SOAP11HTTP_BINDING,
        "http://localhost:8080/calc");

Dispatch<SOAPMessage> dispatch = service.createDispatch(
        portName, SOAPMessage.class, Service.Mode.MESSAGE);
```

# 5. 处理 SOAP Header（认证等）

通过 `SOAPHandler` 拦截器处理 Header：

```java
public class AuthHandler implements SOAPHandler<SOAPMessageContext> {

    @Override
    public boolean handleMessage(SOAPMessageContext context) {
        Boolean outbound = (Boolean) context.get(
                MessageContext.MESSAGE_OUTBOUND_PROPERTY);
        if (outbound) {
            // 客户端：添加认证 Token
            SOAPMessage message = context.getMessage();
            // 操作 Header...
        }
        return true;   // 返回 false 中断链路
    }
}
```

在客户端注册：

```java
port = service.getPort(CalculatorService.class);
BindingProvider bp = (BindingProvider) port;
bp.getBinding().getHandlerChain().add(new AuthHandler());
```

# 6. JAX-WS 与安全

| 方案 | 说明 |
|------|------|
| WS-Security | SOAP 头携带安全令牌/签名/加密 |
| HTTPS 传输 | 传输层加密 |
| WS-Policy | 声明式策略 |
| 应用层令牌 | Header 中传 token（简单场景） |

# 7. 常见问题排查清单

| 问题 | 排查方向 |
|------|----------|
| 服务调用失败 | 查看 WSDL、endpoint 地址、网络/防火墙 |
| Schema 兼容问题 | 契约优先（XSD）保证前后端一致 |
| 中文乱码 | SOAP 消息 XML 声明 UTF-8，设置 MessageFactory 编码 |
| 性能差 | SOAP 开销大，考虑 REST 替代 |
| 命名空间不匹配 | 确认 targetNamespace 一致 |

# 8. JAX-WS vs Spring 对照

| JAX-WS | Spring 生态 |
|--------|-------------|
| `@WebService` + WSDL | Spring Web Services（`spring-ws`） |
| Metro / CXF | Apache CXF 集成 Spring |
| SOAP 协议 | 现代项目通常用 REST（`@RestController`） |
| wsimport | `wsdl2java`（CXF） |

> 现代开发中 SOAP 使用较少，主要出现在与外部老系统（金融、运营商）对接的场景；理解其 SOAP 消息结构、WSDL 与 Handler 机制即可。
