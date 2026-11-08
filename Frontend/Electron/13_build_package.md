开发跑通了,接下来是"**把它变成用户能双击安装的东西**"——`electron-builder` 打包、`asar` 打包源码、代码签名与三平台产物、CI 多平台矩阵。本篇讲清打包的完整链路、产物形态,以及那些"能跑 dev 却打包失败"的坑。

# 1. electron-builder

## 1.1 打包配置

```json
// package.json
{
  "build": {
    "appId": "com.example.myapp",     // 应用唯一 ID(签名/更新依赖它)
    "productName": "我的应用",
    "files": ["out/**", "!out/**/*.map"],   // 打包哪些文件(! 前缀排除)
    "asar": true,                     // 源码归档(默认开)
    "directories": { "output": "release", "buildResources": "build" },
    "win": { "target": [{ "target": "nsis", "arch": ["x64"] }] },
    "nsis": { "oneClick": false, "allowToChangeInstallationDirectory": true },
    "mac": { "target": ["dmg", "zip"], "category": "public.app-category.developer-tools" },
    "linux": { "target": ["AppImage", "deb"] },
    "publish": [{ "provider": "github" }]    // 给自动更新用(第 14 篇)
  }
}
```

★ **关键认知**：`electron-builder` 是主流打包工具,通过 `package.json` 的 `build` 字段配置——**appId**(唯一标识)、**target**(产物格式:win=NSIS/mac=DMG/linux=AppImage)。⚠ `appId` 很重要:**签名、自动更新**都靠它识别应用;APP 一旦发布,`appId` 别乱改(否则系统认成另一个应用)。

## 1.2 打包产物

```text
win:  .exe(可执行) + .exe 安装器(NSIS) / .msi
mac:  .dmg / .zip / .pkg
linux: .AppImage / .deb / .snap
```

★ **关键认知**：产物按平台——**Windows→NSIS 安装包、macOS→DMG、Linux→AppImage**。⚠ 打包是**平台的活**:要在哪个平台出包,通常要在那个平台(或 CI)跑(交叉打包有限制)——这是"全平台发布"要上 CI 的原因(见下)。另注意 **mac 只能由 mac 打**(许可限制),win 包在 mac/linux 上交叉打需要 wine 环境,实践中几乎都交给 CI。

## 1.3 配置的分层结构

```text
build 配置是"通用 + 平台"两层合并:
  顶层(appId/productName/files/asar)     → 所有平台共用
  win/mac/linux 段                        → 只覆盖对应平台
  nsis/dmg/AppImage/deb 段                → 再细一层:控制"安装器形态"
electron-builder.yml 可完全替代 JSON 写法(还能写注释,大项目更清爽)
```

★ **关键认知**：同一个 key 在顶层与平台段都出现时**平台段优先**;`files` 是**白名单+黑名单**混合(`!` 排除),默认还会带上 `package.json` 与 `node_modules` 的**生产依赖**——`devDependencies` 不会进包,所以**运行期依赖别放 dev**。⚠ `nsis.oneClick: false` 才有"选目录/卸载器"的传统安装流程;`mac.category` 缺失会被 App Store 校验拒(公证不受影响但上架要)。

# 2. asar 归档

## 2.1 什么是 asar

```text
打包后,应用的 JS 源码被压进 app.asar(一个归档文件)
  → 不散落成几千个小文件,加载更快、更整洁
  → 但需读里面的文件时,要特殊处理
```

★ **关键认知**：**asar** 是把应用的源码**归档成单个文件**(`app.asar`),好处是**文件少、加载快、整洁**。⚠ asar 是"只读归档"——**写文件不能用通常的 `fs` 路径**(要 `app.asar` 或 `process.noAsar`),但多数应用只是"读代码打包",不影响;动态写的数据本来就该放 `userData`(第 10 篇),天然回避了这个坑。

## 2.2 读 asar 里的文件

```javascript
// 打包后源码路径是 app.asar/xxx,但 Electron 的 fs 大多自动兼容
// 写入有坑:写进 app.asar 需用 fs 配合 asar 包,或干脆写 userData

// asar 对体积也友好:几万个 JS 小文件 → 一个归档,安装时不再逐文件 IO
// 查看/解包(调试"资源漏没漏"时很好用):
//   npx asar list release/win-unpacked/resources/app.asar
//   npx asar extract app.asar ./check
```

★ **关键认知**：读 asar 里的文件 `fs` 大多透明支持;但**写**进 asar 是**不支持**的(只读归档)。⚠ 再强调一次:用户数据/配置永远写 `userData`(第 10 篇),别写进安装目录/asar,这个坑就绕开了。

★ **关键认知**：asar 原理上**不是压缩,是拼接归档**(类似 tar)——它省的是"成千上万小文件的元数据与 IO 次数",不省体积太多;真正减体积要靠砍依赖。⚠ 两类资源**必须排除出 asar**:原生模块 `.node`(要真实文件路径给 dlopen,`asarUnpack` 处理,见第 18 篇)、以及运行时要被外部工具读的文件;`"asar": false` 只在你排查"是不是 asar 引起的"时临时用。

# 3. 代码签名

## 3.1 为什么签名

★ **关键认知**：**代码签名**给应用"验明正身"——**没有签名,Windows SmartScreen/macOS Gatekeeper 会拦截**("这可能是有害软件")。⚠ 签名是**分发的前置条件**:不签名,用户装的时候见红/被拦,体验极差;macOS 更是几乎必须签名+公证(notarization)。

## 3.2 三平台签名

```text
Windows: 代码签名证书(.pfx),signtool 签名(或 electron-builder 配置)
macOS: 开发者证书 + 公证(notarization,apple 审核)
Linux: 多为开源分发,签名需求较低
```

★ **关键认知**：签名成本 mac/win 都有——**Windows 买证书、macOS 买 Apple 开发者账号**。⚠ 这是"上架发布"的真成本之一;开源/内部分发可"自签名"(mac 自签有额外步骤),但分发到公网用户,正规签名是必须项。

## 3.3 Windows EV 与 macOS notarytool

```yaml
# electron-builder.yml(节选)
win:
  certificateFile: ./win.pfx
  certificatePassword: $env(WIN_CSC_KEY_PASSWORD)   # CI 里放 secrets,别进仓库
mac:
  hardenedRuntime: true          # 公证的前提(带 entitlements)
  entitlements: build/entitlements.mac.plist
  notarize: true                 # builder 24+ 默认走 notarytool
```

```bash
# mac 公证(手动链路;electron-builder 已自动化,理解原理有用):
xcrun notarytool submit myapp.zip --apple-id me@x.com --password @app-password --team-id XXXX --keychain-profile notary
xcrun notarytool history --keychain-profile notary
xcrun stapler staple "MyApp.app"     # 把公证票据"钉"回 app,离线也能过 Gatekeeper
```

★ **关键认知**：mac 签名 ≠ 放行——macOS 12+ 的 Gatekeeper 还要求**公证(notarization)**:把产物上传 Apple 扫描、`stapler` 钉回票据;`hardenedRuntime` + entitlements 是公证前提(jit、网络、子进程权限逐项声明)。⚠ Windows 侧的现实:**普通 OV 证书装机头几个月照样弹 SmartScreen**(信誉靠下载量积累),只有 **EV 证书**基本免弹,这也是企业采购常选 EV 的原因;CI 里证书口令走 secrets,密码文件形式(`certificateStore`/pfx)绝不进 git。

# 4. CI 多平台矩阵

```yaml
# .github/workflows/build.yml(节选)
jobs:
  build:
    strategy:
      matrix:
        os: [windows-latest, macos-latest, ubuntu-latest]   # 每个 runner 打自己的包
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: npm ci
      - run: npm run build && npx electron-builder --publish never
        env:
          CSC_LINK: ${{ secrets.WIN_CSC_LINK }}          # 签名材料只存在于 CI secrets
          APPLE_ID: ${{ secrets.APPLE_ID }}
      - uses: actions/upload-artifact@v4
        with: { name: dist-${{ matrix.os }}, path: release/ }
```

★ **关键认知**：多平台产物的标准解是**CI 矩阵**——GitHub Actions 用 `matrix.os` 把 win/mac/linux 各跑各的 runner,产物 `upload-artifact` 汇总,本地机器完全不碰交叉编译。⚠ 三个实战坑:**mac runner 价格贵/排队慢**(免费额度按分钟计权,mac 是 10x),**签名 secrets 要按平台分流**(win 的 pfx 与 mac 的 API key 不同套),**Linux 打 AppImage 要在老 glibc 环境**(官方推荐 docker 镜像 `electronuserland/builder`,否则新 Ubuntu 的包在老 Debian 上跑不起来)。`--publish never` 防 CI 误发版,正式版单独跑 tag 触发。

# 5. 打包失败排查

## 5.1 dev 能跑、打包失败

★ **关键认知**：常见的"dev 正常、打包后白屏/功能失效"根因——**路径问题**(`__dirname` 在打包后变了)、**asar 写入**(往安装目录写文件)、**资源没进 files**(`files` 配置漏了某些静态资源)。⚠ 打包前把"读资源的路径""写文件的路径"都走一遍 `app.getPath('userData')` 与 `__dirname` 的正确用法,能省大量排查。

## 5.2 常见清单

| 坑 | 解法 |
|----|------|
| 打包后白屏 | 检查 loadFile 路径、renderer 资源是否打进 files |
| 写文件报错 | 写入放 userData,别写 asar/安装目录 |
| 图标缺失 | files/构建配置补上图标资源 |
| 体积巨大 | 砍掉不必要依赖、按需模块 |
| 原生模块崩 | `.node` 用 asarUnpack 排除(第 18 篇) |
| mac 装完打不开 | 缺签名/公证,补 notarytool 链路 |

★ **关键认知**：体积优化的三板斧——**生产依赖瘦身**(dependencies 里只留运行时要的)、**files 排除 sourcemap/测试**、前端产物本身压缩;Electron 基线 ~80-100MB 是框架钱,省不了,能省的是你的 `app.asar` 那部分。发布产物里的 `latest.yml`(win/mac 元数据)是自动更新的入口文件,别删、别改名,与第 14 篇衔接。

# 6. 本篇小结与下一步

★ **关键认知**：electron-builder 打包采用"通用+平台"两层配置,appId 是唯一标识且发布后不可变;asar 是拼接归档(只读、不省体积,原生模块要 unpack);签名是分发前提——mac 要 hardenedRuntime+notarization(不止签名),Windows 想免 SmartScreen 基本得 EV;多平台产物交给 CI 矩阵(mac 贵、Linux 用老 glibc 镜像);dev 打包差异多因路径/asar/资源,产物里的 latest.yml 衔接自动更新。

下一步第 14 篇:自动更新——electron-updater、更新服务器、灰度发布。
