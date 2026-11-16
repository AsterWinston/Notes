桌面应用与系统"打成一片"的程度,决定它的成熟度——读写剪贴板、用默认程序打开文件、设置开机自启、监听休眠唤醒。这些零散但高频的系统集成能力,本篇一次讲清。

# 1. clipboard 剪贴板

## 1.1 读写剪贴板

```javascript
const { clipboard } = require('electron')

clipboard.writeText('复制这段文字')           // 写文本
const text = clipboard.readText()             // 读文本

clipboard.writeImage(image)                   // 写图片
clipboard.writeHTML('<b>html</b>')           // 写富文本

clipboard.clear()                             // 清空
```

★ **关键认知**：`clipboard` 读写**系统剪贴板**——`writeText/readText`(文本)、`writeImage`(图片)、`writeHTML`(富文本)。⚠ 剪贴板是**系统级共享**的(你复制的内容别的应用能读到),所以写入敏感内容(密码、密钥)要谨慎,或及时 `clear()`。

## 1.2 监听剪贴板变化

```javascript
let timer
timer = setInterval(() => {
  const now = clipboard.readText()
  if (now !== last) { /* 剪贴板变了 */ }
}, 500)   // 轮询(剪贴板没有可靠的"变化"事件,只能周期读)
```

★ **关键认知**：剪贴板**没有可靠的"变化事件"**,只能**轮询**比对(读一次、和上次比)。⚠ 这是"剪贴板历史/翻译"类工具的通用手法;注意频率与性能平衡,且读取时机(应用在前台时)要控制好。

# 2. shell 调用系统

## 2.1 打开文件/链接/默认程序

```javascript
const { shell } = require('electron')

shell.openExternal('https://example.com')     // 用默认浏览器打开 URL
shell.openPath('/path/to/file.pdf')           // 用默认程序打开文件
shell.showItemInFolder('/path/to/file')       // 在文件管理器中显示
shell.beep()                                  // 系统提示音
```

★ **关键认知**：`shell` 让你**借用系统默认能力**——`openExternal` 用默认浏览器开链接、`openPath` 用默认程序开文件、`showItemInFolder` 在资源管理器定位文件。⚠ **用户点击的链接一定要 `openExternal` 打开**(在系统浏览器里),而非在 Electron 窗口内 `loadURL`(会把你的窗口变成浏览器,还可能有安全风险)。

## 2.2 安全注意

★ **关键认知**：`openExternal` 打开的是**外部可执行环境**——⚖️ **必须校验 URL 是可信协议**(http/https),否则 `openExternal('file://...')` 或恶意协议可能被利用。⚠ 这是文档里明确的安全提示:传给 openExternal 的 URL 要验证 scheme,别直接透传用户输入。

# 3. nativeImage 原生图像

## 3.1 图标与托盘图

```javascript
const { nativeImage } = require('electron')

const icon = nativeImage.createFromPath('./icon.png')
app.dock.setIcon(icon)                // macOS Dock 图标
tray = new Tray(icon)                 // 托盘图标
// 也可从数据生成:createFromDataURL、resize 等
```

★ **关键认知**：`nativeImage` 处理**原生图像**(图标、托盘图、Dock 图)。⚠ 平台对图标格式要求不同——macOS 偏好提供 `@2x` 高分辨率图,Windows 用 `.ico`;托盘图在不同平台的主流尺寸(16/22/32)也不一,需准备多尺寸。

# 4. powerMonitor 与电源

## 4.1 监听休眠/唤醒

```javascript
const { powerMonitor } = require('electron')

powerMonitor.on('suspend', () => { /* 系统睡眠 */ })
powerMonitor.on('resume', () => { /* 系统唤醒 */ })
powerMonitor.on('on-ac', () => { /* 接上电源 */ })
powerMonitor.on('on-battery', () => { /* 拔掉电源 */ })
```

★ **关键认知**：`powerMonitor` 监听**电源状态 / 睡眠唤醒**。⚠ 它的典型用途是"**睡眠期间暂停任务,唤醒后恢复**"(如下载、定时器)——长期运行的后台应用据此优化资源,避免休眠时白跑或醒来时状态错乱。

# 5. 开机自启 autoLaunch

## 5.1 设置自启

```javascript
const { app } = require('electron')

app.setLoginItemSettings({
  openAtLogin: true,      // 开机自启
  openAsHidden: true      // 自启时隐藏(后台应用)
})

const settings = app.getLoginItemSettings()
console.log(settings.openAtLogin)
```

★ **关键认知**：`setLoginItemSettings` 设置**开机自启**(登录时启动)。⚠ 自启要**给用户选择权**(在设置页提供开关),别默默自启;`openAsHidden` 适合"常驻后台"型应用(自启但不弹窗)。macOS 上自启相关有额外权限/配置要注意。

# 6. 系统集成速查

| API | 能力 |
|-----|------|
| clipboard | 读写系统剪贴板 |
| shell | 打开外链/文件/资源管理器 |
| nativeImage | 图标/托盘图 |
| powerMonitor | 睡眠/电源监听 |
| app.setLoginItemSettings | 开机自启 |

# 7. 本篇小结与下一步

★ **关键认知**：clipboard(系统剪贴板,敏感别写)、shell(借默认程序,openExternal 验 scheme)、nativeImage(图标多尺寸)、powerMonitor(睡眠恢复)、autoLaunch(自启给用户选择)。这些零散能力,拼出"成熟桌面应用"的样子。

下一步第 10 篇:文件与存储——fs 文件操作、electron-store、SQLite,把数据落到磁盘。