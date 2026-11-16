桌面应用的数据要**落到磁盘**(不像网页数据在多标签间消失)——配置、用户数据、缓存。本篇讲清 Electron 的三种存储:`fs` 直接操作文件、`electron-store` 存应用配置、`SQLite` 存结构化数据,以及 `userData` 目录这个"该往哪存"的关键。

# 1. userData 目录

## 1.1 数据该存哪

```javascript
const { app } = require('electron')

const dataDir = app.getPath('userData')   // 应用专属数据目录(跨平台自动正确)
// Windows: C:\Users\<user>\AppData\Roaming\<appName>
// macOS:   ~/Library/Application Support/<appName>
// Linux:   ~/.config/<appName>
```

★ **关键认知**：`app.getPath('userData')` 返回**应用专属的数据目录**——**存配置、用户数据、缓存的标准位置**,三平台自动定位正确。⚠ **永远别把用户数据写进应用安装目录**(可能无权限、升级被覆盖、不可写);`userData` 是 Electron 给的"正确位置"。

## 1.2 其他常用路径

```javascript
app.getPath('desktop')     // 桌面
app.getPath('downloads')   // 下载目录
app.getPath('documents')   // 文档
app.getPath('temp')        // 临时
```

★ **关键认知**：`getPath` 可取到各**系统标准目录**(桌面/下载/文档)。⚠ 这些目录名在**不同语言系统的路径不同**,别硬编码 "Desktop",用 `getPath` 让系统告诉你。

# 2. fs 文件操作

## 2.1 主进程里读写文件

```javascript
const fs = require('fs')
const path = require('path')

const filePath = path.join(app.getPath('userData'), 'notes.txt')

// 同步(简单脚本/启动时少量读写)
fs.writeFileSync(filePath, '内容')
const content = fs.readFileSync(filePath, 'utf-8')

// 异步(推荐,避免阻塞主进程)
await fs.promises.writeFile(filePath, '内容')
const content = await fs.promises.readFile(filePath, 'utf-8')
```

★ **关键认知**：主进程可直接用 **Node 的 `fs`** 读写文件。⚠ **优先异步版**(`fs.promises`)——同步 `writeFileSync` 会**阻塞主进程**(主进程还管着窗口响应),大文件/频繁操作用同步会卡 UI。

## 2.2 大文件用流

```javascript
// 读大文件/流式处理时,用 stream 而非一次读入内存
const readStream = fs.createReadStream(largeFile)
readStream.pipe(process.stdout)
```

★ **关键认知**：**大文件**用**流(stream)**一次一块地处理,而非 `readFileSync` 一次性读进内存。⚠ 视频、大日志、备份这类"大块"都该走流,避免内存峰值把应用拖垮。

# 3. electron-store 存配置

## 3.1 键值配置

```javascript
import Store from 'electron-store'

const store = new Store()                 // 基于 userData 下的 config.json
store.set('theme', 'dark')               // 写
const theme = store.get('theme')          // 读
store.delete('theme')                     // 删
```

★ **关键认知**：`electron-store`(第三方库)在 `userData` 下用一个 JSON 文件存**键值配置**——几行代码搞定"记住用户设置"。⚠ 它适合**小体量配置**(主题、窗口尺寸、偏好设置);大数据(文档、列表)别塞配置里,该用 fs/db。

## 3.2 与 userData 的关系

★ **关键认知**：electron-store 本质是"**对 `userData/config.json` 的封装**"——你要手写也行(读 JSON → 改 → 写回),它只是省事。⚠ 理解这层,你就知道"配置存在哪、怎么手动备份/迁移"——就是 userData 里的那个 json 文件。

# 4. SQLite 存结构化数据

## 4.1 用 better-sqlite3

```javascript
import Database from 'better-sqlite3'

const db = new Database(path.join(app.getPath('userData'), 'app.db'))

// 建表 + 增删查改
db.exec('CREATE TABLE IF NOT EXISTS notes (id INTEGER PRIMARY KEY, content TEXT)')
const insert = db.prepare('INSERT INTO notes (content) VALUES (?)')
insert.run('第一条笔记')

const rows = db.prepare('SELECT * FROM notes WHERE id = ?').get(1)
```

★ **关键认知**：**SQLite**(如 better-sqlite3)存**结构化、关系型数据**——比 JSON 文件更适合"频繁查询、多字段、有一定数据量"的场景。⚠ 选型:`electron-store` 存配置、`fs` 存文件、**SQLite 存"要查询的多条数据"**(笔记、订单、聊天记录);`better-sqlite3` 是同步 API 但性能好,需在主进程使用。

## 4.2 选型决策

| 需求 | 用 |
|------|-----|
| 少量配置/偏好 | electron-store |
| 单个文件/文档 | fs |
| 大量结构化/需查询 | SQLite |
| 大量二进制/媒体 | fs(流)或专用存储 |

★ **关键认知**：存储选型看**数据形态**——配置用 store、文件用 fs、结构化查询用 SQLite。⚠ 别"一个 JSON 塞所有数据"(数据一多读写整份、无查询能力),也别"为存两个开关上 SQLite"(杀鸡用牛刀)。

# 5. 本篇小结与下一步

★ **关键认知**：数据存到 `userData`(标准目录);fs 读写文件(优先异步/大文件用流);electron-store 存配置;SQLite 存结构化数据。选型看数据形态,别放安装目录。

下一步进入**阶段三 · 工程化**,第 11 篇:前端框架集成——Vue/React + Electron 的通信封装与最佳实践。