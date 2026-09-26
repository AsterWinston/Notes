
# 1. 并行构建（-T）

```bash
# 每个 CPU 核一个线程
mvn -T 1C clean install

# 固定线程数
mvn -T 4 clean install
```

- `1C` = 1 thread per CPU core。
- 适用场景：多模块项目，模块间无强顺序依赖。
- 注意：部分插件不支持并行（如 `maven-site-plugin`），可能需串行执行。

# 2. Maven Daemon (mvnd)

```bash
# 安装 mvnd 后直接替换 mvn
mvnd clean install
mvnd -T 1C clean verify
```

- 原理：JVM 常驻守护进程，消除每次构建的 JVM 启动开销（~2-5秒）。
- 首次启动与 `mvn` 无差异，后续构建快 30-50%。
- 完全兼容现有 Maven 插件和配置。

# 3. 离线模式

```bash
mvn -o clean install
```

- 跳过网络仓库检查，仅使用本地缓存。
- 适用：依赖已全部缓存的增量构建（如本地开发快速迭代）。
- 注意：首次构建或有新依赖时不能使用。

# 4. 跳过测试

```bash
# 跳过测试执行（仍会编译测试代码）
mvn -DskipTests clean package

# 完全跳过测试（不编译也不执行）
mvn -Dmaven.test.skip=true clean package
```

- CI 的 PR 检查阶段可跳过集成测试，合并后执行全量测试。

# 5. 增量构建

```bash
# 只构建变更的模块及其下游
mvn -pl :changed-module -amd install
```

- 结合 `-pl` / `-amd` / `-rf` 精准控制构建范围。
- 大型项目可将 PR 涉及的模块通过脚本分析后只构建变更部分。

# 6. 构建缓存（Build Cache）

## 6.1 Maven 扩展缓存

- `maven-build-cache-extension`：Maven 官方开发中的构建缓存扩展。
- 原理：缓存每个模块的构建输出，输入不变时复用缓存。

## 6.2 Develocity（原 Gradle Enterprise）

- 统一的构建缓存和构建扫描，支持 Maven 和 Gradle。
- 适用于大型企业 Monorepo。

# 7. 本地仓库优化

- 使用 SSD 存放 `~/.m2/repository`。
- 定期清理未使用的依赖：
```bash
# 查看依赖使用报告
mvn dependency:analyze
```

- CI 中指定仓库路径，避免污染用户级仓库：
```bash
mvn -Dmaven.repo.local=$WORKSPACE/.m2/repository clean install
```

# 8. JVM 调优

```bash
# 增大 Maven 构建 JVM 内存
export MAVEN_OPTS="-Xmx2048m -Xms1024m -XX:+UseG1GC"

# Windows
set MAVEN_OPTS=-Xmx2048m -Xms1024m
```

- 大型项目建议至少 2GB 堆内存。
- 使用 G1GC 减少 GC 停顿。

# 9. 减少日志输出

```bash
# 静默模式（只显示错误）
mvn -q clean install

# 关闭下载进度日志
mvn -ntp clean install

# 批处理模式（CI 推荐）
mvn -B clean verify
```

- `-B`：非交互 + 精简日志。
- `-ntp`：no transfer progress，关闭传输进度条。

# 10. 插件选择优化

- 优先选择支持并行和增量的插件版本。
- 避免构建中执行不必要的目标（如每次构建都生成 javadoc）。
- 在父 POM 中统一锁定插件版本，避免版本漂移导致的额外下载。

# 11. 常见问题排查清单

- 构建慢：用 `-T` 并行 + `mvnd` Daemon + `-o` 离线三板斧排查。
- 依赖下载慢：检查是否配置了私服镜像，检查网络。
- 插件执行慢：检查是否重复执行（如多个模块都执行了相同的代码生成）。
- OOM：增大 `MAVEN_OPTS` 的 `-Xmx` 值。
- 首次构建极慢：正常现象，需下载所有依赖，后续应缓存。
