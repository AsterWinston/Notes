
# 1. 概述

## 1.1 什么是 JavaMail

JavaMail（Jakarta Mail）是 JavaEE 的**邮件收发标准 API**，统一支持 SMTP（发送）、POP3/IMAP（接收）等邮件协议。

```
┌────────────┐   SMTP    ┌────────────┐   SMTP    ┌────────────┐
│ Java 应用   │ ────────► │ 发送服务器   │ ────────► │ 收件人      │
│ (发送)      │          │ (SMTP)      │           │ 邮箱       │
└────────────┘          └────────────┘           └─────┬──────┘
┌────────────┐   POP3/IMAP ┌────────────┐              │
│ Java 应用   │ ◄───────── │ 接收服务器   │ ◄───────────┘
│ (接收)      │            │ (POP3/IMAP) │
└────────────┘            └────────────┘
```

## 1.2 涉及协议

| 协议 | 端口 | 用途 |
|------|------|------|
| SMTP | 25（465 SSL / 587 TLS） | 发送邮件 |
| POP3 | 110（995 SSL） | 接收邮件（下载后本地管理） |
| IMAP | 143（993 SSL） | 接收邮件（邮件保留在服务器，支持多文件夹/同步） |

## 1.3 参考实现

- RI：Jakarta Mail（`com.sun.mail:jakarta.mail`），提供 SMTP/POP3/IMAP Provider。
- 第三方：Apache Commons Email（对 JavaMail 的封装）。

# 2. 核心 API

| 对象 | 作用 |
|------|------|
| `Session` | 邮件会话（持有配置 Properties 与认证信息） |
| `Transport` | 发送邮件 |
| `Store` | 连接邮件接收服务器 |
| `Folder` | 邮件文件夹（收件箱等） |
| `Message` | 邮件消息（`MimeMessage` 是常用实现） |
| `Address` | 地址（`InternetAddress`） |
| `Multipart` / `BodyPart` | 邮件主体（正文 + 附件） |

# 3. 发送简单邮件

## 3.1 基础配置

```java
Properties props = new Properties();
props.put("mail.smtp.host", "smtp.example.com");
props.put("mail.smtp.port", "587");
props.put("mail.smtp.auth", "true");
props.put("mail.smtp.starttls.enable", "true");

Session session = Session.getInstance(props, new Authenticator() {
    @Override
    protected PasswordAuthentication getPasswordAuthentication() {
        return new PasswordAuthentication("sender@example.com", "password");
    }
});
```

## 3.2 发送文本邮件

```java
MimeMessage message = new MimeMessage(session);
message.setFrom(new InternetAddress("sender@example.com"));
message.setRecipient(Message.RecipientType.TO,
        new InternetAddress("receiver@example.com"));
message.setSubject("主题");
message.setSentDate(new Date());
message.setText("正文内容");

Transport.send(message);
```

- `setRecipient` 类型：`TO`（收件人）、`CC`（抄送）、`BCC`（密送）。

# 4. 发送带附件/HTML 的邮件

```java
MimeMessage message = new MimeMessage(session);
message.setFrom(new InternetAddress("sender@example.com"));
message.setRecipients(Message.RecipientType.TO,
        InternetAddress.parse("a@x.com,b@x.com"));
message.setSubject("HTML + 附件");

// 1. HTML 正文
MimeBodyPart htmlPart = new MimeBodyPart();
htmlPart.setContent("<h1>你好</h1><p>邮件正文</p>", "text/html;charset=UTF-8");

// 2. 附件
MimeBodyPart attachPart = new MimeBodyPart();
try (InputStream in = new FileInputStream("report.pdf")) {
    attachPart.setDataHandler(new DataHandler(new ByteArrayDataSource(
            in.readAllBytes(), "application/pdf")));
    attachPart.setFileName("report.pdf");
}

// 3. 组装
Multipart multipart = new MimeMultipart();
multipart.addBodyPart(htmlPart);
multipart.addBodyPart(attachPart);
message.setContent(multipart);

Transport.send(message);
```

- 编码注意：中文标题需 `MimeUtility.encodeText("中文标题")`，正文用 UTF-8。
- 附件大文件可用流式 `setDataHandler` 避免内存溢出。

# 5. 接收邮件

## 5.1 IMAP 读取收件箱

```java
Properties props = new Properties();
props.put("mail.store.protocol", "imaps");
Session session = Session.getInstance(props);

Store store = session.getStore("imaps");
store.connect("imap.example.com", "user@example.com", "password");

Folder inbox = store.getFolder("INBOX");
inbox.open(Folder.READ_ONLY);

Message[] messages = inbox.getMessages();
for (Message message : messages) {
    System.out.println("主题：" + message.getSubject());
    System.out.println("发件人：" + message.getFrom()[0]);
    System.out.println("正文：" + getText(message));
}

inbox.close(false);
store.close();
```

## 5.2 提取正文（处理多部分）

```java
public static String getText(Part part) throws Exception {
    if (part.isMimeType("text/plain")) {
        return (String) part.getContent();
    }
    if (part.isMimeType("multipart/*")) {
        Multipart mp = (Multipart) part.getContent();
        for (int i = 0; i < mp.getCount(); i++) {
            String text = getText(mp.getBodyPart(i));
            if (text != null) return text;
        }
    }
    return null;
}
```

# 6. JavaEE 容器中的 JavaMail

## 6.1 容器配置邮件会话

应用服务器可以配置邮件会话资源，应用通过 JNDI 获取：

**web.xml 声明：**

```xml
<resource-ref>
    <res-ref-name>mail/MySession</res-ref-name>
    <res-type>jakarta.mail.Session</res-type>
    <res-auth>Container</res-auth>
</resource-ref>
```

**代码注入：**

```java
@Resource(mappedName = "java:comp/env/mail/MySession")
private Session mailSession;

public void sendNotice(String to) throws MessagingException {
    MimeMessage message = new MimeMessage(mailSession);
    message.setFrom(new InternetAddress(mailSession.getProperty("mail.from")));
    message.setRecipients(Message.RecipientType.TO, InternetAddress.parse(to));
    message.setSubject("通知");
    message.setText("内容");
    Transport.send(message);
}
```

# 7. 常见问题排查清单

| 问题 | 排查方向 |
|------|----------|
| `AuthenticationFailedException` | 检查账号/授权码（部分服务商需开启 SMTP 并设置客户端授权码） |
| 连接超时 | 检查 SMTP 端口（25/465/587）、SSL/TLS 配置、网络白名单 |
| 中文乱码 | 主题用 `MimeUtility.encodeText`，正文指定 UTF-8 |
| 被判定为垃圾邮件 | 配置 SPF/DKIM、使用正规域名、注意发送频率 |
| 附件过大被拒 | 分片或改用对象存储链接 |
| 发送无报错但收不到 | 检查收件箱/垃圾箱、SPF、退信（bounce） |

# 8. 与 Spring Mail 对照

| JavaMail | Spring Mail |
|----------|-------------|
| `Session` + `Transport.send` | `JavaMailSender` / `SimpleMailMessage` |
| `MimeMessage` 手工组装 | `MimeMessageHelper` 简化组装 |
| 手动配置 Properties | `spring.mail.*` 配置 |
| 无模板支持 | 可集成 Thymeleaf 模板渲染邮件 |

> 现代 Spring Boot 项目推荐直接用 `spring-boot-starter-mail` + `JavaMailSender`，JavaMail 是其底层实现。
