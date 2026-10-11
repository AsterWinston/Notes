开发跑通了,接下来是"**把它变成用户能双击安装的东西**"——`electron-builder` 打包、`asar` 打包源码、代码签名与三平台产物。本篇讲清打包的完整链路、产物形态,以及那些"能跑 dev 却打包失败"的坑。

# 1. electron-builder

## 1.1 打包配置

```json
// package.json
{
  "build": {
    "appId": "com.example.myapp",     // 应用唯一 ID(签名/更新依赖它)
    "productName": "我的应用",
    "files": ["out/**"],              // 打包哪些文件
    "directories": { "output": "release" },
    "win": { "target": ["nsis"] },    // Windows:NSIS 安装包
    "mac": { "target": ["dmg"] },     // macOS:DMG
    "linux": { "target": ["AppImage"] }
  }
}
```

★ **关键认知**：`electron-builder` 是主流打包工具,通过 `package.json` 的 `build` 字段配置——**appId**(唯一标识)、**target**(产物格式:win=NSIS/mac=DMG/linux=AppImage)。⚠ `appId` 很重要:**签名、自动更新**都靠它识别应用;APP 一旦发布,`appId` 别乱改(否则系统认成另一个应用)。

## 1.2 打包产物

```
win:  .exe(可执行) + .exe 安装器(NSIS) / .msi
mac:  .dmg / .zip / .pkg
linux: .AppImage / .deb / .snap
```

★ **关键认知**：产物按平台——**Windows→NSIS 安装包、macOS→DMG、Linux→AppImage**。⚠ 打包是**平台的活**:要在哪个平台出包,通常要在那个平台(或 CI)跑(交叉打包有限制)——这是"全平台发布"要上 CI 的原因(见下)。

# 2. asar 归档

## 2.1 什么是 asar

```
打包后,应用的 JS 源码被压进 app.asar(一个归档文件)
  → 不散落成几千个小文件,加载更快、更整洁
  → 但需读里面的文件时,要特殊处理
```

★ **关键认知**：**asar** 是把应用的源码**归档成单个文件**(`app.asar`),好处是**文件少、加载快、整洁**。⚠ asar 是"只读归档"——**写文件不能用通常的 `fs` 路径**(要 `app.asar` 或 `process.noAsar`),但多数应用只是"读代码打包",不影响;动态写的数据本来就该放 `userData`(第 10 篇),天然回避了这个坑。

## 2.2 读 asar 里的文件

```javascript
// 打包后源码路径是 app.asar/xxx,但 Electron 的 fs 大多自动兼容
// 写入有坑:写进 app.asar 需用 fs 配合 asar 包,或干脆写 userData
```

★ **关键认知**：读 asar 里的文件 `fs` 大多透明支持;但**写**进 asar 是**不支持**的(只读归档)。⚠ 再强调一次:用户数据/配置永远写 `userData`(第 10 篇),别写进安装目录/asar,这个坑就绕开了。

# 3. 代码签名

## 3.1 为什么签名

★ **关键认知**：**代码签名**给应用"验明正身"——**没有签名,Windows SmartScreen/macOS Gatekeeper 会拦截**("这可能是有害软件")。⚠ 签名是**分发的前置条件**:不签名,用户装的时候见红/被拦,体验极差;macOS 更是几乎必须签名+公证(notarization)。

## 3.2 三平台签名

```
Windows: 代码签名证书(.pfx),signtool 签名(或 electron-builder 配置)
macOS: 开发者证书 + 公证(notarization,apple 审核)
Linux: 多为开源分发,签名需求较低
```

★ **关键认知**：签名成本 mac/win 都有——**Windows 买证书、macOS 买 Apple 开发者账号**。⚠ 这是"上架发布"的真成本之一;开源/内部分发可"自签名"(mac 自签有额外步骤),但分发到公网用户,正规签名是必须项。

# 4. 打包失败排查

## 4.1 dev 能跑、打包失败

★ **关键认知**：常见的"dev 正常、打包后白屏/功能失效"根因——**路径问题**(`__dirname` 在打包后变了)、**asar 写入**(往安装目录写文件)、**资源没进 files**(`files` 配置漏了某些静态资源)。⚠ 打包前把"读资源的路径""写文件的路径"都走一遍 `app.getPath('userData')` 与 `__dirname` 的正确用法,能省大量排查。

## 4.2 常见清单

| 坑 | 解法 |
|----|------|
| 打包后白屏 | 检查 loadFile 路径、renderer 资源是否打进 files |
| 写文件报错 | 写入放 userData,别写 asar/安装目录 |
| 图标缺失 | files/构建配置补上图标资源 |
| 体积巨大 | 砍掉不必要依赖、按需模块 |

# 5. 本篇小结与下一步

★ **关键认知**：electron-builder 打包(package.json build 配置),appId 是唯一标识;asar 归档源码(只读,写入走 userData);代码签名是分发前提(mac 需公证);dev 打包差异多因路径/asar/资源。

下一步第 14 篇:自动更新——electron-updater、更新服务器、灰度发布。