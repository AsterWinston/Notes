
# 1. 概述

**SAML（Security Assertion Markup Language）** 是 OASIS 制定的基于 XML 的联邦身份认证标准，2005 年发布 2.0 版本，至今仍是企业级 SSO 的事实标准之一。SAML 的核心是通过 **断言（Assertion）** 在 IdP 和 SP 之间传递用户身份信息。

```
┌──────────────────────────────────────────────────────────────────┐
│                     SAML 2.0 核心角色                             │
│                                                                   │
│  ┌──────────────────┐              ┌──────────────────┐          │
│  │   用户 (Subject)  │              │  Identity Provider│          │
│  │                  │    认证      │  (IdP)            │          │
│  │   浏览器/客户端   │─────────────►│  身份认证 +        │          │
│  │                  │              │  签发 Assertion   │          │
│  └────────┬─────────┘              └──────────────────┘          │
│           │                               │                       │
│           │     访问 Service              │ SAML Response         │
│           ▼                               ▼                       │
│  ┌──────────────────────────────────────────────────┐            │
│  │          Service Provider (SP)                   │            │
│  │          接收 Assertion，建立局部会话             │            │
│  └──────────────────────────────────────────────────┘            │
└──────────────────────────────────────────────────────────────────┘
```

# 2. SAML 核心概念

## 2.1 SAML Assertion（断言）

SAML Assertion 是 SAML 的核心，IdP 签发断言来声明关于用户（Subject）的三类信息：

```xml
<!-- 一个完整的 SAML Assertion 示例 -->
<saml:Assertion
    xmlns:saml="urn:oasis:names:tc:SAML:2.0:assertion"
    ID="_abc123def456"
    Version="2.0"
    IssueInstant="2024-01-15T10:30:00Z">

    <!-- 1. 签发者 -->
    <saml:Issuer>https://idp.example.com</saml:Issuer>

    <!-- 2. 签名（防篡改） -->
    <ds:Signature>
        <ds:SignedInfo>...</ds:SignedInfo>
        <ds:SignatureValue>...</ds:SignatureValue>
        <ds:KeyInfo>
            <ds:X509Certificate>MIIDXTCCAkWgAwIBAg...</ds:X509Certificate>
        </ds:KeyInfo>
    </ds:Signature>

    <!-- 3. 主体（用户标识） -->
    <saml:Subject>
        <saml:NameID Format="urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress">
            zhangsan@example.com
        </saml:NameID>
        <saml:SubjectConfirmation Method="urn:oasis:names:tc:SAML:2.0:cm:bearer">
            <saml:SubjectConfirmationData
                NotOnOrAfter="2024-01-15T10:35:00Z"
                Recipient="https://sp.example.com/saml/SSO"
                InResponseTo="_request123"/>
        </saml:SubjectConfirmation>
    </saml:Subject>

    <!-- 4. 条件（有效期、受众限制） -->
    <saml:Conditions
        NotBefore="2024-01-15T10:25:00Z"
        NotOnOrAfter="2024-01-15T10:35:00Z">
        <saml:AudienceRestriction>
            <saml:Audience>https://sp.example.com</saml:Audience>
        </saml:AudienceRestriction>
    </saml:Conditions>

    <!-- 5. 认证声明 -->
    <saml:AuthnStatement AuthnInstant="2024-01-15T10:30:00Z"
                         SessionIndex="_session123">
        <saml:AuthnContext>
            <saml:AuthnContextClassRef>
                urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport
            </saml:AuthnContextClassRef>
        </saml:AuthnContext>
    </saml:AuthnStatement>

    <!-- 6. 属性声明（用户属性） -->
    <saml:AttributeStatement>
        <saml:Attribute Name="email" NameFormat="urn:oasis:...basic">
            <saml:AttributeValue>zhangsan@example.com</saml:AttributeValue>
        </saml:Attribute>
        <saml:Attribute Name="displayName" NameFormat="urn:oasis:...basic">
            <saml:AttributeValue>张三</saml:AttributeValue>
        </saml:Attribute>
        <saml:Attribute Name="department" NameFormat="urn:oasis:...basic">
            <saml:AttributeValue>技术部</saml:AttributeValue>
        </saml:Attribute>
        <saml:Attribute Name="role" NameFormat="urn:oasis:...basic">
            <saml:AttributeValue>ADMIN</saml:AttributeValue>
            <saml:AttributeValue>USER</saml:AttributeValue>
        </saml:Attribute>
    </saml:AttributeStatement>
</saml:Assertion>
```

## 2.2 NameID 格式

NameID 是 SAML 中标识用户的核心字段，支持多种格式：

| 格式 | URI | 说明 |
|------|-----|------|
| **未指定** | `urn:oasis:names:tc:SAML:1.1:nameid-format:unspecified` | 解析方式由 SP 和 IdP 约定 |
| **邮箱** | `urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress` | `user@domain.com` |
| **X.509** | `urn:oasis:names:tc:SAML:1.1:nameid-format:X509SubjectName` | 证书 DN |
| **Windows Domain** | `urn:oasis:names:tc:SAML:1.1:nameid-format:WindowsDomainQualifiedName` | `DOMAIN\username` |
| **Kerberos** | `urn:oasis:names:tc:SAML:2.0:nameid-format:kerberos` | `user@REALM` |
| **实体 ID** | `urn:oasis:names:tc:SAML:2.0:nameid-format:entity` | 另一个 SAML Entity |
| **持久化** | `urn:oasis:names:tc:SAML:2.0:nameid-format:persistent` | 不透明的持久标识符 |
| **临时** | `urn:oasis:names:tc:SAML:2.0:nameid-format:transient` | 临时一次性标识 |

**实践建议**：`emailAddress` 或 `persistent` 最常用。`persistent` 保护用户隐私，每个 SP 看到不同的标识符。

## 2.3 认证上下文类

```xml
<!-- 认证方式声明 -->
<saml:AuthnContextClassRef>
    <!-- 密码保护传输（HTTPS + 密码） -->
    urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport

    <!-- 多因素认证 -->
    urn:oasis:names:tc:SAML:2.0:ac:classes:TimeSyncToken

    <!-- X.509 证书 -->
    urn:oasis:names:tc:SAML:2.0:ac:classes:TLSClient

    <!-- Kerberos -->
    urn:oasis:names:tc:SAML:2.0:ac:classes:Kerberos

    <!-- 之前的会话（SSO 再次使用） -->
    urn:oasis:names:tc:SAML:2.0:ac:classes:PreviousSession
</saml:AuthnContextClassRef>
```

# 3. SAML 2.0 两种绑定流程

## 3.1 SP-Initiated SSO（最常用）

```
用户              SP (Service)          IdP (Identity Provider)
 │                    │                       │
 │ 1. 访问受保护资源  │                       │
 │───────────────────►│                       │
 │                    │ 2. 创建 AuthnRequest  │
 │                    │    (ID, ACS URL等)    │
 │                    │                       │
 │ 3. SP重定向到IdP   │                       │
 │◄───────────────────│                       │
 │ HTTP 302 Location: https://idp.example.com/SAML2/SSO?SAMLRequest=base64encoded&RelayState=originalUrl
 │                    │                       │
 │ 4. GET IdP SSO URL │                       │
 │──────────────────────────────────────────►│
 │                    │                       │ 5. 解析 AuthnRequest
 │                    │                       │ 6. 验证请求合法性
 │                    │                       │ 7. 检查是否有 IdP Session
 │                    │                       │    ├─ 有：直接签发 Assertion
 │                    │                       │    └─ 无：展示登录页
 │                    │                       │
 │ 8. 用户登录（如需要）                       │
 │──────────────────────────────────────────►│
 │                    │                       │ 9. 验证凭据
 │                    │                       │10. 创建 IdP Session
 │                    │                       │11. 构造 SAML Response
 │                    │                       │    (含 Assertion + 签名)
 │                    │                       │
 │12. IdP返回HTML Form（自动提交）            │
 │◄──────────────────────────────────────────│
 │ <form action="https://sp.example.com/SAML2/SSO" method="POST">
 │   <input name="SAMLResponse" value="base64encoded"/>
 │   <input name="RelayState" value="originalUrl"/>
 │ </form>
 │ <script>document.forms[0].submit()</script>
 │                    │                       │
 │13. POST到SP的ACS                            │
 │───────────────────►│                       │
 │                    │14. 验证 SAML Response │
 │                    │    ├─ 验签（X.509）   │
 │                    │    ├─ 检查有效期      │
 │                    │    ├─ 检查 Audience   │
 │                    │    ├─ 检查 InResponseTo│
 │                    │    └─ 提取用户信息    │
 │                    │15. 创建SP局部会话     │
 │                    │16. 重定向到原始URL    │
 │◄───────────────────│                       │
 │17. 正常访问资源                             │
```

## 3.2 IdP-Initiated SSO

用户先登录 IdP 门户，再点击系统中的链接启动 SSO：

```
用户先登录 IdP 门户 → 点击"访问 App A" → IdP 直接签发 Assertion → POST 到 SP
```

**优点**：用户无需先访问 SP，从 IdP 门户一站式导航。
**缺点**：缺少 InResponseTo 验证（因为没有 AuthnRequest），安全性略低。

# 4. SAML 绑定（Binding）详解

SAML 协议消息可以通过多种传输方式在 SP 和 IdP 之间传递：

## 4.1 HTTP Redirect Binding

```
GET /SAML2/SSO?SAMLRequest=fZJNT%2BMwEIb…
               &RelayState=https%3A%2F%2Fsp.example.com%2Fapp
               &SigAlg=http%3A%2F%2Fwww.w3.org%2F2001%2F04%2Fxmldsig-more%23rsa-sha256
               &Signature=abc123…
```

**特点**：
- 适用于 AuthnRequest 等小消息
- SAML 消息：XML → Deflate 压缩 → Base64 编码 → URL Encode
- 支持签名（SigAlg + Signature 参数）
- 受 URL 长度限制（通常 8KB 左右）

```java
// SAML 消息编码流程
// 1. 构造 XML
String samlXml = buildAuthnRequest();
// 2. Deflate 压缩
ByteArrayOutputStream baos = new ByteArrayOutputStream();
DeflaterOutputStream dos = new DeflaterOutputStream(baos, new Deflater(Deflater.DEFLATED, true));
dos.write(samlXml.getBytes(StandardCharsets.UTF_8));
dos.close();
// 3. Base64 编码
byte[] compressed = baos.toByteArray();
String encoded = Base64.getEncoder().encodeToString(compressed);
// 4. URL Encode
String urlEncoded = URLEncoder.encode(encoded, "UTF-8");
// 5. 拼接 URL
String redirectUrl = idpSsoUrl + "?SAMLRequest=" + urlEncoded;
```

## 4.2 HTTP POST Binding

```html
<!-- IdP 返回给浏览器的 HTML 页面 -->
<html>
<body onload="document.forms[0].submit()">
    <form method="POST" action="https://sp.example.com/SAML2/SSO">
        <input type="hidden" name="SAMLResponse"
               value="PHNhbWxwOlJlc3BvbnNlIHhtbG5z...（Base64编码的SAML XML）"/>
        <input type="hidden" name="RelayState"
               value="https://sp.example.com/original/page"/>
    </form>
</body>
</html>
```

**特点**：
- 适用于 Assertion 等大消息
- 无 URL 长度限制
- SAML Response 通常经过 Base64 编码（不一定压缩）
- 浏览器自动提交表单，用户无感

## 4.3 HTTP Artifact Binding

```
流程：
1. IdP 生成一个短的 Artifact（引用），通过 302 重定向传回 SP
2. SP 收到 Artifact 后，通过后端 SOAP/HTTP 请求向 IdP 换取完整的 Assertion

优点：
- URL 短，避免 GET 请求过长
- Assertion 不经过浏览器，更安全
- 无需 HTML 表单自动提交

缺点：
- 需要后端直连（SOAP 接口）
- 多一次网络往返
- 实现更复杂
```

## 4.4 绑定方式选择建议

| 场景 | AuthnRequest 方向 | Response 方向 |
|------|------------------|--------------|
| **一般 Web 应用** | HTTP Redirect | HTTP POST |
| **Assertion 很大** | HTTP Redirect | HTTP Artifact |
| **高安全要求** | HTTP Redirect | HTTP Artifact |

# 5. SAML 元数据（Metadata）

## 5.1 Metadata 作用

SAML Metadata 是描述 SP 和 IdP 能力和端点的 XML 文档，双方交换 Metadata 后即可建立信任关系，无需手动配置所有参数。

```xml
<!-- IdP Metadata 示例 -->
<EntityDescriptor
    xmlns="urn:oasis:names:tc:SAML:2.0:metadata"
    entityID="https://idp.example.com">

    <!-- IdP 描述符 -->
    <IDPSSODescriptor
        protocolSupportEnumeration="urn:oasis:names:tc:SAML:2.0:protocol">

        <!-- 签名证书（用于 SP 验证 IdP 签名） -->
        <KeyDescriptor use="signing">
            <ds:KeyInfo>
                <ds:X509Data>
                    <ds:X509Certificate>MIIDXTCCAkW...</ds:X509Certificate>
                </ds:X509Data>
            </ds:KeyInfo>
        </KeyDescriptor>

        <!-- 加密证书（用于 SP 加密数据发给 IdP） -->
        <KeyDescriptor use="encryption">
            <ds:KeyInfo>
                <ds:X509Data>
                    <ds:X509Certificate>MIIDXTCCAkW...</ds:X509Certificate>
                </ds:X509Data>
            </ds:KeyInfo>
        </KeyDescriptor>

        <!-- 单点登出服务端点 -->
        <SingleLogoutService
            Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect"
            Location="https://idp.example.com/SAML2/SLO"/>
        <SingleLogoutService
            Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST"
            Location="https://idp.example.com/SAML2/SLO"/>

        <!-- NameID 格式 -->
        <NameIDFormat>
            urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress
        </NameIDFormat>

        <!-- SSO 端点 -->
        <SingleSignOnService
            Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect"
            Location="https://idp.example.com/SAML2/SSO"/>
        <SingleSignOnService
            Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST"
            Location="https://idp.example.com/SAML2/SSO"/>
    </IDPSSODescriptor>
</EntityDescriptor>
```

# 6. SAML 安全性

## 6.1 防重放攻击

```xml
<!-- AuthnRequest -->
<samlp:AuthnRequest ID="_abc123" ...>
    <!-- 生成随机 Request ID -->
</samlp:AuthnRequest>

<!-- SAML Response（Assertion 内） -->
<saml:SubjectConfirmationData
    InResponseTo="_abc123"
    NotOnOrAfter="2024-01-15T10:35:00Z"
    Recipient="https://sp.example.com/SAML2/SSO">

<!-- SP 验证： -->
<!-- 1. InResponseTo 必须匹配原始 AuthnRequest ID -->
<!-- 2. 当前时间 < NotOnOrAfter -->
<!-- 3. Recipient 必须等于当前 SP 的 ACS URL -->
```

## 6.2 XML 签名验证

```java
// SAML Response 验签核心逻辑
public boolean validateSignature(Response samlResponse) {
    // 1. 获取断言中的签名
    Signature signature = samlResponse.getSignature();
    if (signature == null) {
        signature = samlResponse.getAssertions().get(0).getSignature();
    }

    // 2. 从 Metadata 中获取 IdP 的公钥
    X509Certificate idpCert = metadataResolver
        .getSigningCertificate(samlResponse.getIssuer().getValue());

    // 3. 验证签名
    try {
        SignatureValidator validator = new SignatureValidator(idpCert);
        validator.validate(signature);
        return true;
    } catch (ValidationException e) {
        return false;
    }
}
```

## 6.3 XML 签名包装攻击（XML Signature Wrapping）

```
攻击原理：
原始 Assertion：
<Response>
  <Assertion ID="legit1">用户A</Assertion>
  <Signature>签名覆盖 Response</Signature>
</Response>

攻击后（但签名仍然有效，因为签名覆盖的是 Response 根元素）：
<Response>
  <Assertion ID="legit1">用户A</Assertion>  ← 合法
  <Assertion ID="evil2">用户B(Admin)</Assertion>  ← 攻击者注入
  <Signature>签名覆盖 Response</Signature>
</Response>

如果 SP 取第一个 Assertion → 正常
如果 SP 取最后一个 Assertion → 被攻击！
```

**防御措施**：
1. 始终验证签名覆盖的元素正是被使用的 Assertion
2. 使用 `InResponseTo` + `Recipient` + `AudienceRestriction` 限制 Assertion 使用范围
3. 检查 Assertion 数量是否符合预期

# 7. SAML 2.0 单点登出（SLO）

SAML SLO 支持两种模式：

```xml
<!-- IdP Metadata 中的 SLO 端点 -->
<SingleLogoutService
    Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect"
    Location="https://idp.example.com/SAML2/SLO"/>
```

## 7.1 SP-Initiated SLO

```
用户点击 SP 中的登出按钮
  → SP 发送 LogoutRequest 到 IdP
  → IdP 销毁全局会话
  → IdP 向所有已认证的 SP 发送 LogoutRequest
  → 各 SP 销毁局部会话
  → IdP 返回 LogoutResponse 给发起登出的 SP
```

## 7.2 IdP-Initiated SLO

```
用户在 IdP 点击全局登出
  → IdP 向所有已认证的 SP 发送 LogoutRequest
  → 各 SP 销毁局部会话
  → IdP 销毁全局会话
```

# 8. SAML vs CAS vs OIDC

| 维度 | SAML 2.0 | CAS | OIDC |
|------|----------|-----|------|
| **数据格式** | XML（重量级） | XML/JSON（轻量） | JSON/JWT（轻量） |
| **标准组织** | OASIS | 开源社区（Apereo） | OpenID Foundation |
| **移动端支持** | 弱（XML 解析成本高） | 弱 | 强（原生支持） |
| **SLO** | ✅ 完善 | ✅ 完善 | ✅ RP-Initiated |
| **跨组织** | ✅ 联邦身份 | ❌ 不适合 | ✅ 适合 |
| **元数据** | ✅ 标准化 | ❌ | ✅ Discovery |
| **签名加密** | XML Sig/Enc | ❌ 依赖 HTTPS | JWS/JWE |
| **配置复杂度** | 高 | 低 | 中 |

# 9. 何时选择 SAML

| 场景 | 推荐 |
|------|------|
| 需要跨组织、跨安全域的联合认证 | ✅ SAML |
| 企业客户已有 AD/LDAP 基础设施 | ✅ SAML |
| 需要与 Office 365、Salesforce 等 SaaS 集成 | ✅ SAML |
| 需要严格的 XML 签名/加密机制 | ✅ SAML |
| 纯 Java Web 企业内部系统 | 考虑 CAS（更简单） |
| 新系统，需要移动端支持 | 考虑 OIDC |
| 微服务架构 | 考虑 JWT + OAuth2 |
