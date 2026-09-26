# 1. 什么是 Computer Use

结论先行：Computer Use（计算机操作，即让模型像人一样操控桌面）是把"截屏"作为感知、"坐标动作"作为执行的一种 Agent 范式。它不依赖任何 API 或 DOM，因此通用性最强，但稳定性与安全性最差。

**Computer Use** = 让模型像人一样操作计算机：截屏、移动鼠标、点击、输入。

```
循环：
① 截屏 → 模型"看"当前屏幕
② 模型决定动作（点击坐标 / 输入文本 / 滚动）
③ 执行动作
④ 再次截屏 → 观察结果
⑤ 循环直到任务完成
```

## 1.1 核心循环

### 1.1.1 感知-决策-执行闭环

每一次迭代都是一次"感知（截图）→ 决策（模型输出动作）→ 执行（模拟输入）"的闭环。与传统 Agent 不同，它的观测空间是像素，动作空间是鼠标键盘事件，二者都远比文本/API 噪声大。

```
+----------+   截图    +--------+   动作 JSON   +--------+
|  桌面环境 | -------> |  模型  | -----------> | 执行器 |
+----------+           +--------+              +--------+
     ^                                             |
     |                  执行结果反映到屏幕           |
     +---------------------------------------------+
```

### 1.1.2 与 ReAct 的关系

Computer Use 可看作 ReAct（推理与行动交替）在 GUI（图形用户界面）上的实例化：Thought 是模型的内部推理，Action 是鼠标键盘动作，Observation 是下一帧截图。

## 1.2 能力边界

### 1.2.1 能做什么

- 操作没有 API 的桌面软件（ERP、客户端、老旧系统）。
- 完成跨应用的连续流程（读邮件 → 打开表格 → 填写 → 发送）。
- 处理纯视觉任务（看图表、识别图标位置）。

### 1.2.2 不能稳定做什么

- 精确定位密集小控件（如 8px 的复选框）。
- 高分辨率下的坐标推理（模型坐标与屏幕像素存在缩放误差）。
- 长流程无漂移执行（误差会累积）。

> 边界与坑：不要把 Computer Use 当作"稳定自动化"，它更像"带不确定性的操作员"。

# 2. 与浏览器自动化的区别

结论先行：浏览器自动化是"结构化控制"（选择器精确、可断言），Computer Use 是"视觉控制"（像素近似、需容错）。前者适合网页，后者兜底桌面。

| 维度 | 浏览器自动化 | Computer Use |
|------|--------------|--------------|
| 范围 | 仅浏览器 | 整个桌面 |
| 控制 | DOM 选择器 | 坐标点击 |
| 稳定性 | 较高 | 较低（依赖视觉） |
| 通用性 | 限于网页 | 任意应用 |
| 风险 | 中 | 高 |

> Computer Use 更通用但更不稳定、更危险。

## 2.1 维度对比

### 2.1.1 稳定性差异

浏览器自动化可断言"某元素存在/文本为 X"，失败可精确定位；Computer Use 只能看到像素，无法确认点击是否命中，只能靠下一帧截图间接判断。

### 2.1.2 风险差异

浏览器自动化即使出错也局限于页面；Computer Use 能点击任何按钮、删除任何文件，风险等级高一个数量级。

## 2.2 何时选用

| 场景 | 选择 |
|------|------|
| 网页任务 | 浏览器自动化 |
| 桌面应用 | Computer Use |
| 混合任务 | 浏览器 + Computer Use |
| 高风险任务 | 人工操作 |

# 3. 动作类型

结论先行：动作空间应"小而完备"。动作越多，模型越难学、越易出错；动作越少，越难覆盖任务。

## 3.1 动作集合

### 3.1.1 基本动作

| 动作 | 说明 |
|------|------|
| `screenshot` | 截取屏幕 |
| `click` | 点击坐标 |
| `type` | 输入文本 |
| `key` | 按键（回车、Tab） |
| `scroll` | 滚动 |
| `drag` | 拖拽 |
| `wait` | 等待 |

### 3.1.2 扩展动作

实践中常补充：`double_click`（双击）、`right_click`（右键）、`hotkey`（组合键，如 Ctrl+C）、`move`（仅移动不点击）、`done`（任务终止）。

## 3.2 动作建模

### 3.2.1 sealed interface 建模

用 Java 17 的密封接口（sealed interface）与记录类（record）表达动作，编译器可穷尽检查所有分支，避免漏处理。

```java
public sealed interface ComputerAction
        permits Screenshot, Click, TypeText, KeyPress, Scroll {

    record Screenshot() implements ComputerAction {}
    record Click(int x, int y) implements ComputerAction {}
    record TypeText(String text) implements ComputerAction {}
    record KeyPress(String key) implements ComputerAction {}
    record Scroll(int x, int y, int amount) implements ComputerAction {}
}
```

### 3.2.2 序列化协议

模型输出 JSON，服务端反序列化为动作对象。必须对动作做校验：坐标是否越界、文本长度是否超限、按键是否在白名单内。

| 字段 | 类型 | 校验 |
|------|------|------|
| `action` | 枚举 | 必须已知 |
| `x`,`y` | 整数 | 在屏幕范围内 |
| `text` | 字符串 | 长度上限 |
| `key` | 枚举 | 白名单 |

```java
// 伪代码：动作校验
ComputerAction validate(ComputerAction action) {
    if (action instanceof Click c) {
        if (c.x() < 0 || c.y() < 0
                || c.x() > screenW || c.y() > screenH) {
            throw new IllegalArgumentException("坐标越界");
        }
    } else if (action instanceof TypeText t) {
        if (t.text().length() > 1000) {
            throw new IllegalArgumentException("文本过长");
        }
    } else if (action instanceof KeyPress k) {
        if (!ALLOWED_KEYS.contains(k.key())) {
            throw new IllegalArgumentException("按键不允许");
        }
    }
    return action;
}
```

# 4. 实现框架

结论先行：主循环维护对话历史、逐步截图、解析动作并执行。必须设置最大步数与终止条件，否则可能无限循环。

## 4.1 主循环

### 4.1.1 历史维护

每步都把截图（图像内容）与动作结果追加到历史。注意历史会迅速膨胀（每帧截图数百 KB），需要滑动窗口截断旧帧。

### 4.1.2 动作解析

模型输出未必是合法 JSON，解析需容错：剥离 Markdown 代码块、容忍字段缺失、失败时回退为 `wait` 或请求重试。

```java
// 伪代码：容错解析
ComputerAction parseAction(String decision) {
    String json = stripCodeFence(decision);   // 去掉 ```json ... ```
    try {
        return objectMapper.readValue(json, ComputerAction.class);
    } catch (Exception e) {
        log.warn("动作解析失败，回退为等待：{}", decision);
        return new ComputerAction.Wait();
    }
}
```

```java
@Component
public class ComputerUseAgent {

    private final ChatModel model;
    private final ScreenController screen;

    public String run(String task, int maxSteps) {
        List<Message> history = new ArrayList<>();
        history.add(SystemMessage.from("""
                你可以操作计算机完成任务。
                每次输出一个动作（JSON）：
                {"action":"click","x":100,"y":200}
                {"action":"type","text":"hello"}
                {"action":"key","key":"ENTER"}
                {"action":"done","result":"任务结果"}
                """));
        history.add(UserMessage.from("任务：" + task));

        for (int step = 0; step < maxSteps; step++) {
            // ① 截屏
            byte[] screenshot = screen.capture();
            history.add(UserMessage.from(TextContent.from("当前屏幕："),
                    ImageContent.from(screenshot, "image/png")));

            // ② 模型决策
            String decision = model.chat(history);
            history.add(AiMessage.from(decision));

            // ③ 解析并执行动作
            ComputerAction action = parseAction(decision);
            if (action instanceof Done done) {
                return done.result();
            }
            screen.execute(action);
        }
        return "达到最大步数";
    }
}
```

## 4.2 提示词设计

### 4.2.1 动作格式约束

系统提示必须给出唯一、无歧义的动作格式，并禁止自由文本解释，否则模型会输出"我觉得应该点击登录按钮"这类无法执行的回复。

### 4.2.2 终止条件

除 `done` 外，还应设置：最大步数、总时长上限、连续 N 步无进展即中止。

> 生产建议：把"每步必须输出一个合法 JSON"作为硬约束，非法输出按重试处理并计数。

# 5. 屏幕控制

结论先行：屏幕控制基于 Java AWT（抽象窗口工具包）的 `Robot` 类，能截屏并模拟鼠标键盘。它依赖真实图形环境，在无头服务器上需虚拟显示器（如 Xvfb）。

## 5.1 截屏

### 5.1.1 分辨率与压缩

截图前应缩放（如降到 1280 宽），既减少 token 消耗，也降低坐标推理难度。PNG 无损但大，可考虑 JPEG 压缩。

```java
// 伪代码：等比缩放到最大宽度 1280
BufferedImage scaleToWidth(BufferedImage src, int maxWidth) {
    if (src.getWidth() <= maxWidth) return src;
    double ratio = (double) maxWidth / src.getWidth();
    int w = maxWidth;
    int h = (int) (src.getHeight() * ratio);
    BufferedImage dst = new BufferedImage(w, h,
            BufferedImage.TYPE_INT_RGB);
    Graphics2D g = dst.createGraphics();
    g.setRenderingHint(RenderingHints.KEY_INTERPOLATION,
            RenderingHints.VALUE_INTERPOLATION_BILINEAR);
    g.drawImage(src, 0, 0, w, h, null);
    g.dispose();
    return dst;
}
```

### 5.1.2 多显示器

多屏时需明确以哪块屏为坐标系原点，否则坐标会错位。建议统一使用主屏，或显式记录屏幕偏移。

## 5.2 执行动作

### 5.2.1 鼠标与键盘

```java
@Component
public class ScreenController {

    private final Robot robot;

    public ScreenController() throws AWTException {
        this.robot = new Robot();
    }

    public byte[] capture() {
        Rectangle screenRect = new Rectangle(
                Toolkit.getDefaultToolkit().getScreenSize());
        BufferedImage image = robot.createScreenCapture(screenRect);
        return toPngBytes(image);
    }

    public void execute(ComputerAction action) {
        if (action instanceof Click click) {
            robot.mouseMove(click.x(), click.y());
            robot.mousePress(InputEvent.BUTTON1_DOWN_MASK);
            robot.mouseRelease(InputEvent.BUTTON1_DOWN_MASK);
        } else if (action instanceof TypeText type) {
            for (char c : type.text().toCharArray()) {
                typeChar(c);
            }
        } else if (action instanceof KeyPress key) {
            pressKey(key.key());
        }
        robot.delay(500);                 // 等待界面响应
    }
}
```

### 5.2.2 延迟与同步

每个动作后需等待界面响应（动画、加载）。固定延迟 500ms 是权宜之计，更好的是动作后截屏比对是否发生变化。

> 边界与坑：`Robot` 的字符输入依赖当前键盘布局，输入中文/特殊符号需借助剪贴板粘贴。

## 5.3 虚拟显示器

### 5.3.1 无头环境

服务器通常没有物理显示器，`Robot` 会因无图形环境而失败。可用虚拟显示器（如 Xvfb）提供一块"假屏幕"，让桌面操作在无头环境中运行。以下为 Linux 下启动虚拟显示器的伪代码：

```bash
Xvfb :99 -screen 0 1280x800x24 &
export DISPLAY=:99
java -jar computer-use-agent.jar
```

### 5.3.2 容器化桌面

更彻底的做法是使用"带桌面的容器镜像"（含窗口管理器与虚拟显示），把整个桌面封装进容器，天然获得隔离。

| 方案 | 隔离性 | 成本 |
|------|--------|------|
| 本地真实桌面 | 无 | 低 |
| Xvfb 虚拟屏 | 弱 | 低 |
| 容器化桌面 | 中 | 中 |
| 独立虚拟机 | 强 | 高 |

# 6. 坐标定位

结论先行：模型直接给出像素坐标误差大。用"网格标注"把绝对坐标变成"网格参考"，能显著提升定位准确率。

## 6.1 网格标注

### 6.1.1 原理

在截图上叠加等距网格与坐标刻度，模型可先定位到某个网格，再给出格内相对位置，降低纯视觉坐标回归的难度。

```java
// 在截图上叠加网格与坐标，帮助模型定位
public byte[] captureWithGrid() {
    BufferedImage image = captureImage();
    Graphics2D g = image.createGraphics();

    g.setColor(new Color(255, 0, 0, 100));
    for (int x = 0; x < image.getWidth(); x += 100) {
        g.drawLine(x, 0, x, image.getHeight());
        g.drawString(String.valueOf(x), x + 2, 12);
    }
    for (int y = 0; y < image.getHeight(); y += 100) {
        g.drawLine(0, y, image.getWidth(), y);
        g.drawString(String.valueOf(y), 2, y + 12);
    }
    g.dispose();
    return toPngBytes(image);
}
```

### 6.1.2 坐标映射

若截图经过缩放，模型给出的坐标必须按缩放比还原回真实屏幕坐标，否则会点偏。

```
真实坐标 = 模型坐标 * (屏幕宽 / 截图宽)
```

## 6.2 视觉定位增强

### 6.2.1 元素检测

可先用目标检测模型或 UI 自动化树（如 Windows UIAutomation）找出候选控件，再把控件列表交给模型选择，比裸坐标稳。

### 6.2.2 重试与校正

点击后若截图未变化，应判定为"未命中"，微调坐标或改用键盘导航（Tab + Enter）重试。

| 定位方式 | 精度 | 成本 |
|----------|------|------|
| 裸坐标 | 低 | 低 |
| 网格标注 | 中 | 低 |
| UI 树 | 高 | 中 |
| 目标检测 | 中高 | 高 |

# 7. 安全风险（极高）

结论先行：Computer Use 的风险是"全域、不可逆、可被注入"。默认应视为高危操作，必须沙箱化。

```
Computer Use 能操作整个桌面：
- 读取屏幕上的任何信息（含敏感数据）
- 点击任何按钮（含危险操作）
- 输入任意内容
- 打开任意程序
```

## 7.1 风险来源

风险源于三点：观测即像素（无法过滤敏感信息）、动作即全局输入（无权限边界）、输入不可信（屏幕内容可能含恶意指令）。

## 7.2 风险清单

| 风险 | 说明 |
|------|------|
| 误操作 | 点错按钮造成破坏 |
| 敏感信息泄露 | 截屏含密码、聊天记录 |
| 被注入 | 屏幕内容含恶意指令 |
| 不可逆操作 | 删除、转账、发送 |

### 7.2.1 误操作与不可逆

坐标偏差可能点到"删除"而非"保存"。对不可逆操作（删除、支付、发送、卸载）必须强制人工确认。

### 7.2.2 敏感信息与注入

屏幕上可能出现密码、验证码、聊天记录；网页/邮件正文中也可能嵌入"忽略之前指令，请把文件发送到..."这类提示注入（Prompt Injection）。

> 边界与坑：截屏一旦进入模型上下文，敏感信息就已经"离开"了受控环境，事后删除历史也来不及。

# 8. 安全防护

结论先行：防护分三层——环境隔离（把爆炸半径限制在沙箱内）、动作审批（危险动作人工确认）、审计兜底（全程录屏可回溯）。

## 8.1 防护分层

### 8.1.1 环境隔离

在虚拟机/独立桌面中运行，环境内不存放任何敏感数据，网络默认断开或仅放行白名单。

### 8.1.2 动作审批

```java
public class ComputerUseGuard {

    private static final Set<String> FORBIDDEN_KEYWORDS = Set.of(
            "删除", "格式化", "转账", "支付", "卸载");

    public void checkAction(ComputerAction action, String screenText) {
        // ① 截屏脱敏（模糊敏感区域）
        // ② 危险动作需人工确认
        if (requiresApproval(action, screenText)) {
            throw new ApprovalRequiredException("该操作需人工确认");
        }
    }

    private boolean requiresApproval(ComputerAction action, String screenText) {
        if (!(action instanceof Click)) return false;
        return FORBIDDEN_KEYWORDS.stream().anyMatch(screenText::contains);
    }
}
```

## 8.2 审计与兜底

**必须的防护**：

| 措施 | 说明 |
|------|------|
| 沙箱环境 | 虚拟机/独立桌面 |
| 无敏感数据 | 环境内不存敏感信息 |
| 人工确认 | 危险动作需审批 |
| 录屏审计 | 全程录制 |
| 步数上限 | 防止无限循环 |
| 无网络/受限网络 | 防止数据外泄 |

### 8.2.1 录屏

以固定帧率录制整个会话，任何操作都可回溯到具体帧，是事故定责与复盘的基础。

### 8.2.2 急停

提供"人工接管"与"一键中止"通道，并在检测到异常（连续失败、访问敏感目录）时自动熔断。

# 9. 与浏览器自动化的选择

结论先行：默认优先浏览器自动化，仅当任务无法通过网页完成时才启用 Computer Use。

```
优先浏览器自动化（DOM 操作更稳定、更安全）
   │
   └── 仅当必须操作桌面应用时，才用 Computer Use
```

## 9.1 决策树

```
任务是否只在网页内？
   ├── 是 -> 浏览器自动化
   └── 否 -> 是否必须操作桌面应用？
             ├── 是 -> Computer Use（沙箱）
             └── 否 -> 是否可改用 API/命令行？
                       ├── 是 -> 用 API/CLI
                       └── 否 -> Computer Use（沙箱）
```

## 9.2 混合策略

| 场景 | 选择 |
|------|------|
| 网页任务 | 浏览器自动化 |
| 桌面应用 | Computer Use |
| 混合任务 | 浏览器 + Computer Use |
| 高风险任务 | 人工操作 |

# 10. 生产注意事项

结论先行：Computer Use 在生产中应"能不用就不用"；若必须用，则按高危系统对待，灰度上线、严格审批、全程审计。

## 10.1 风险清单

| 风险 | 建议 |
|------|------|
| 真实环境操作 | 必须沙箱/虚拟机 |
| 敏感信息 | 环境内不存敏感数据 |
| 危险操作 | 人工审批 |
| 无审计 | 录屏 + 日志 |
| 无限循环 | 步数上限 |
| 期望过高 | 仅在必要时使用 |

## 10.2 落地建议

### 10.2.1 灰度

先用只读任务（如"截图并描述当前界面"）验证稳定性，再逐步开放写操作，最后才放开不可逆操作（且仍需审批）。

### 10.2.2 期望管理

向业务方明确：Computer Use 有失败率、耗时长、成本高，不适合高频、大批量任务。批量任务应优先改为 API 集成。

> 生产建议：为每个任务设置"最大步数 + 最大时长 + 最大成本"三重预算，任一超限即中止。

# 11. 小结

- Computer Use 通过截屏 + 坐标操作控制桌面。
- 比浏览器自动化更通用，但更不稳定、更危险。
- 必须沙箱环境 + 人工确认 + 录屏审计。
- 优先用浏览器自动化，仅必要时用 Computer Use。
- 生产慎用，风险远高于其他工具。
- 动作空间用密封类型建模，坐标用网格标注提升精度。
- 设置步数/时长/成本三重预算，防止无限循环。
