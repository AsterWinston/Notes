# 1. 并行构建（-T）

Maven 的 Reactor 在默认情况下是**串行**调度模块的：即使模块之间没有任何依赖关系，也一个接一个地构建。`-T`（threads）参数让 Reactor 把**相互无依赖的模块**交给多个线程并行构建，它是多模块项目最直接的提速开关——本质是用 CPU 换墙钟时间，对单模块项目则完全无效，因为并行粒度是"模块"，不是插件或源文件。

```bash
# 每个 CPU 核一个线程
mvn -T 1C clean install

# 固定线程数
mvn -T 4 clean install
```

| 写法 | 含义 | 建议 |
|------|------|------|
| `-T 1C` | 线程数 = CPU 核数 × 1 | 通用起点 |
| `-T 2C` | 线程数 = CPU 核数 × 2 | IO 密集（依赖下载为主）时可尝试 |
| `-T 4` | 固定 4 线程 | 核数多但内存有限时按**内存**定线程 |
| `-T 0.5C` | 每 2 核 1 线程 | 大模块单构建吃内存时的保守值 |

**注意**：并行化不改变依赖解析瓶颈。如果构建慢的根源在第 2 节的"下载与元数据检查"，`-T` 救不了；先定位瓶颈再决定是否并行。

**设计考量**：

- **插件与扩展的线程安全**：部分插件不支持并行（如 `maven-site-plugin`、一些旧版代码生成插件），构建时会报 "not marked as thread-safe"，需要升级插件版本，或把该目标挪出并行构建（放到独立的非并行 profile / 单独一次 `mvn` 调用）。
- **Maven 3.9 的扩展线程安全声明**：3.9.x 开始，核心扩展（extension）也需要声明线程安全，旧扩展在 `-T` 下可能报错，属于升级兼容问题而非构建问题。
- **内存跟着线程数涨**：每个模块的编译都占用堆内存，8 线程并行时 `MAVEN_OPTS` 的 `-Xmx` 不够就会 GC 风暴甚至 OOM，线程数要按"核数与内存取小"来确定。

# 2. 依赖解析慢的根因

日常抱怨"构建慢"，大多数时候慢的不是编译，而是**依赖解析与下载**。常见根因可以归纳为四类，每一类都有对应的诊断手段：

**SNAPSHOT 元数据检查**是最大的隐形开销。快照仓库默认 `updatePolicy=daily`，每天第一次构建会对每个快照依赖（含其父 POM 链）发起 metadata 检查；聚合项目里只要有一个上游依赖快照版本，解析请求就会成倍放大。用 `-U` 强制更新时，这个检查会对**所有**快照构件立即执行一次。

**依赖树与 POM 链规模**。传递依赖深、父 POM 层级多，意味着要下载大量小 XML 文件。Maven 解析时对同一坐标会去重，但 POM 链与插件依赖仍然构成几十上百次网络往返——单次只有几 KB，架不住次数多。

**仓库与镜像配置**。POM 里声明了多个 `<repository>` 而 `settings.xml` 没有统一镜像收敛时，解析缺失构件会**按顺序逐个仓库尝试**，每个未命中都是一次往返；仓库顺序摆放不当会显著放大耗时。

**失败缓存与代理**。下载失败的构件会在本地仓库写入 `*.lastUpdated` 标记，默认策略下不会立即重试，但换一个模块、换一次命令行就可能重新尝试，表现为"反复超时、构建时快时慢"。

```bash
# 观察单个构件的解析路径与耗时
mvn -X dependency:resolve 2>&1 | Select-String "Downloading|Downloaded"

# 强制更新快照（诊断用，不要放进常规构建）
mvn -U clean verify
```

> **行业实践**：生产 CI 禁用 `-U`，快照同步交给每日一次的基线任务；先判断"慢在下载还是编译"，再决定用药——镜像与缓存治下载，`-T` 与 mvnd 治编译。

# 3. 本地仓库复用与 CI 缓存

本地仓库（`~/.m2/repository`）本身就是 Maven 最大的"构建缓存"：构件解析一旦命中本地，就完全不产生网络流量，这也是"第二次构建总比第一次快"的根本原因。把它放在 SSD 上收益明显；团队统一仓库路径还能减少重复下载，但要注意**并发写坏**的风险。CI 的核心命题是：每次构建都是全新执行器，如果本地仓库从零开始，等于每次都要重演"首次构建极慢"，所以必须把 `~/.m2/repository` 纳入流水线缓存。

GitHub Actions：

```yaml
# .github/workflows/build.yml
- name: Cache Maven repository
  uses: actions/cache@v4
  with:
    path: ~/.m2/repository
    key: ${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}
    restore-keys: ${{ runner.os }}-m2-

- name: Build
  run: mvn -B -ntp -T 1C clean verify
```

GitLab CI：

```yaml
# .gitlab-ci.yml
variables:
  MAVEN_OPTS: "-Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository"
cache:
  key: "m2-$CI_COMMIT_REF_SLUG"
  paths:
    - .m2/repository/
script:
  - mvn -B -ntp -T 1C clean verify
```

> **注意**：只缓存 repository 目录，**不要**把 `target` 缓存回去——Maven 默认不做构建缓存，脏 target 会被"莫名复用"，轻则构建行为不一致，重则把旧产物带上线。Maven 也没有天然的 lockfile，缓存 key 通常绑定所有 POM 的内容哈希（`hashFiles('**/pom.xml')`）；命中旧 key 后 `-U` 之外的问题都不大，快照会随缓存老化，必要时用 `--delete-obsolete-maven-cache` 类脚本或换 key 重建。

# 4. 离线模式

```bash
mvn -o clean install
```

`-o` 让 Maven **完全不访问远程仓库**，仅用本地仓库解析，省掉所有元数据检查与网络往返，对依赖已齐备的增量构建提速非常明显。适用：依赖已全部缓存的增量构建（如本地开发快速迭代）。

限制也很清楚：前提是仓库"全且对"——插件、父 POM、extension 都得在本地有缓存，缺任何一个直接构建失败；首次构建或新增依赖时不能使用。更稳的日常做法其实是不带 `-o`，而是用私服镜像加合理更新策略把网络请求收敛到内网。

> **注意**：本地用 `-o` 迭代、提交前**不带 `-o` 完整构建一次**，是避免"本地能过 CI 不能过"这类经典事故的低成本习惯。

# 5. 通过 Profile 跳过测试与文档

```bash
# 跳过测试执行（仍会编译测试代码）
mvn -DskipTests clean package

# 完全跳过测试（不编译也不执行）
mvn -Dmaven.test.skip=true clean package
```

两者的差别常被忽略：`-DskipTests` 只跳过 surefire **执行**，测试代码仍参与编译，能守住"测试代码至少是语法正确"的底线；`-Dmaven.test.skip=true` 连测试编译一起跳过，快得多，但测试代码是否还能编译完全无人兜底，只适合临时验证。

比命令行参数更可治理的做法是把这些开关做成父 POM 里的 profile：

```xml
<!-- 父 POM：本地快速打包档 -->
<profiles>
	<profile>
		<id>fast</id>
		<properties>
			<skipTests>true</skipTests>
			<maven.javadoc.skip>true</maven.javadoc.skip>
			<maven.source.skip>true</maven.source.skip>
		</properties>
	</profile>
</profiles>
```

```bash
mvn -Pfast -T 1C clean install
```

CI 的 PR 检查阶段可跳过集成测试与 javadoc/source 生成（它们动辄贡献总时长的 10-20%），合并后再执行全量 `verify`。

> **注意**：跳过测试/文档只属于**过程加速**，不能出现在主干发布构建里；发布链路必须完整执行测试、javadoc、签名等目标，否则等于把质量与合规交给运气。

# 6. 增量构建与 mvnd

## 6.1 按变更范围构建

```bash
# 只构建变更的模块及其下游
mvn -pl :changed-module -amd install
```

结合 `-pl`（projects list）/ `-am`（also make，上游依赖）/ `-amd`（also dependents，下游依赖）/ `-rf`（resume from）精准控制构建范围。大型项目可以把 PR 涉及的模块通过脚本分析后只构建变更部分：

```bash
# 示例：根据 git diff 推导变更模块并构建其自身与下游
CHANGED=$(git diff --name-only HEAD~1 | grep '/pom.xml$' | xargs -n1 dirname | paste -sd, -)
mvn -B -ntp -T 1C -pl "$CHANGED" -amd install
```

脚本要点：变更模块要包含"模块自身 + 下游"（`-amd`），除非你能证明接口与 DTO 没有变化；公共模块（父 POM、common 包）变更时应放弃裁剪、全量构建。

## 6.2 增量编译的局限

`maven-compiler-plugin` 默认按"模块粒度"判断 stale——只要有源文件比 class 新，往往触发**整个模块重编**；`maven-shade` 之类的插件更是每次全量重写产物。所以 Maven 的"增量"远不如 Gradle 精细，本地极速迭代更多要靠 mvnd、构建缓存或热部署工具（JRebel / Spring Boot DevTools）来补，而不是反复 `clean`——`clean` 会删掉仅有的增量痕迹，高频构建时只在日常打包与发布时才 clean。

## 6.3 Maven Daemon (mvnd)

```bash
# 安装 mvnd 后直接替换 mvn
mvnd clean install
mvnd -T 1C clean verify
```

- 原理：JVM 常驻守护进程，消除每次构建的 JVM 启动与类加载开销（约 2-5 秒），同时保留 JIT 预热收益。
- 首次启动与 `mvn` 无差异，后续构建快 30-50%；与 `-T` 叠加效果更佳（mvnd 的默认构建线程已是 `1C`）。
- 完全兼容现有 Maven 插件和配置，可与 Maven Wrapper 组合使用；Maven 4 的 daemon 化正是同一思路的延续。

# 7. 构建缓存（Build Cache）

## 7.1 Maven 扩展缓存

- `maven-build-cache-extension`：Maven 官方孵化中的构建缓存扩展，思路与 Gradle Build Cache 一致——哈希每个插件目标的输入（源文件、POM、依赖解析结果），输入未变时直接复用上次输出，连编译都跳过。
- 风险在于**缓存投毒**：插件必须"确定性"（同输入同输出），否则缓存会固化错误产物；因此该扩展默认只覆盖核心插件，且要求插件显式声明可缓存，启用前要逐一评估。

## 7.2 Develocity（原 Gradle Enterprise）

- 统一的构建缓存和构建扫描，支持 Maven 和 Gradle；构建扫描能给出"哪个插件目标耗了多长、依赖解析花了多少"的火焰图级定位，是大型组织治"慢"的前后门。
- 适用于大型企业 Monorepo，商业授权，中小团队可先用 `-X` 日志与上面第 2 节的手段做轻量定位。

# 8. 本地仓库与 JVM 调优

- 使用 SSD 存放 `~/.m2/repository`。
- 定期清理未使用的依赖、收敛依赖树（见第 9 节）：
```bash
# 查看依赖使用报告
mvn dependency:analyze
```

- CI 中指定仓库路径，避免污染用户级仓库：
```bash
mvn -Dmaven.repo.local=$WORKSPACE/.m2/repository clean install
```

```bash
# 增大 Maven 构建 JVM 内存
export MAVEN_OPTS="-Xmx2048m -Xms1024m -XX:+UseG1GC"

# Windows
set MAVEN_OPTS=-Xmx2048m -Xms1024m
```

- 大型项目建议至少 2GB 堆内存；`-T` 并行时堆需求随线程数近似倍增，OOM 的常见表现是 GC 频繁、构建异常退出或插件报 "GC overhead limit exceeded"。
- 使用 G1GC 减少 GC 停顿。

# 9. 依赖树裁剪与插件瘦身

依赖树越胖，解析越慢、打包越慢、classpath 扫描越慢（Spring Boot 启动时的 auto-configuration 条件评估同样受其拖累）。裁剪三步：

```bash
# 1. 找出冲突与被忽略的版本
mvn dependency:tree -Dverbose

# 2. 找出"用了没声明 / 声明了没用"的依赖
mvn dependency:analyze

# 3. 排除未使用与重复引入的传递依赖
```

```xml
<dependency>
	<groupId>some.overweight.lib</groupId>
	<artifactId>heavy-lib</artifactId>
	<exclusions>
		<exclusion>
			<groupId>unused.transitive</groupId>
			<artifactId>*</artifactId>
		</exclusion>
	</exclusions>
</dependency>
```

插件侧同样要做瘦身：

- 优先选择支持并行和增量的插件版本。
- 避免构建中执行不必要的目标（如每次构建都生成 javadoc、source、翻遍全仓库的 enforcer 规则）。
- 在父 POM 中统一锁定插件版本，避免 "version not specified" 警告背后每次构建重新解析插件最新 RELEASE 的额外下载与版本漂移。

# 10. 日志与命令行参数

```bash
# 静默模式（只显示错误）
mvn -q clean install

# 关闭下载进度日志
mvn -ntp clean install

# 批处理模式（CI 推荐）
mvn -B clean verify
```

- `-B`：非交互 + 精简日志，去掉进度条与 ANSI 控制符，CI 日志体积能小一半以上。
- `-ntp`：no transfer progress，关闭逐个文件的传输进度条。
- `-q` 只留错误，排查问题时反而会缺信息，日常 CI 用 `-B -ntp` 组合比 `-q` 更合适。

# 11. 量化收益案例

模拟案例口径：120 模块、Spring Boot 3 单体仓库，8C16G CI 执行器，基线为串行 `mvn clean verify`。

| 优化手段（逐步叠加） | 构建耗时 | 变化 |
|---------------------|---------|------|
| ① 基线：串行构建，无镜像无缓存 | 14 分 | — |
| ② `-T 1C` 并行 | 6 分 | -57% |
| ③ 换 mvnd | 4.5 分 | 累计 -68% |
| ④ 私服镜像 + `-B -ntp` | 4 分 | 累计 -71% |
| ⑤ CI 缓存 `~/.m2/repository` | 3.5 分 | 累计 -75% |
| ⑥ `-pl/-amd` 范围裁剪 + 依赖裁剪 | 3 分 | 累计 -79% |

> **注意**：这是模拟口径，不同项目的瓶颈分布差异极大——依赖下载占大头的，收益集中在④⑤；编译占大头的，收益集中在②③。上表的价值在方法：先定位瓶颈，再叠加手段，每一步都量化验证，而不是把"加速咒语"一次性全念一遍。

# 12. 常见问题排查清单

- 构建慢：用 `-T` 并行 + `mvnd` Daemon + `-o` 离线三板斧排查，并用第 2 节的方法先定位"下载慢还是编译慢"。
- 依赖下载慢：检查是否配置了私服镜像，检查网络；快照多则检查 `updatePolicy` 与 `-U`。
- 插件执行慢：检查是否重复执行（如多个模块都执行了相同的代码生成），用 `mvn -X` 看目标级耗时。
- OOM：增大 `MAVEN_OPTS` 的 `-Xmx` 值；并行时先降线程数再谈扩容。
- 首次构建极慢：正常现象，需下载所有依赖，后续应缓存。
- `-T` 报错 not thread-safe：升级插件/扩展，或把相关目标挪出并行构建。
- 本地能过 CI 不能过：本地 `mvn install` 污染了工作流，去掉 `-o` 重新全量验证，并清理 CI 缓存 key。

# 13. 小结

Maven 提速是一套组合拳，优先级可以背下来：**先治网络层**（私服镜像、CI 缓存 `~/.m2/repository`、治理 SNAPSHOT 更新策略），**再治调度层**（`-T 1C` 并行、mvnd 常驻守护），**最后治范围**（`-pl/-amd` 变更裁剪、依赖树裁剪、profile 跳过非必要目标）。构建缓存扩展与 Develocity 属于前两者做完之后的进阶选项，引入前先评估缓存投毒与授权成本。

所有加速手段都必须通过 profile 或 CI 参数**显式隔离**：跳过测试、离线模式、范围裁剪只服务于迭代过程，主干与发布构建保持完整语义，这是"快"不变成"事故"的前提。衡量标准也只有一个——优化前后各留一次构建数据，用第 11 节的表格式持续复盘，而不是凭感觉说"好像快了"。
