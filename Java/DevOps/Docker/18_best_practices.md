
# 1. 镜像构建（Build）

1. 多阶段构建：构建工具不进入运行镜像。
2. 基础镜像固定版本（必要时用 digest 锁定关键环境）。
3. 合理分层：先复制依赖描述文件再下载依赖，提升缓存命中。
4. `.dockerignore`：排除 `.git`、构建产物、密钥、无关文件。
5. 不在镜像里写密钥：避免 `ENV/ARG` 固化敏感信息。

参考：`8_dockerfile_best.md`、`15_cicd.md`。

# 2. 容器运行（Run）

1. 明确入口：ENTRYPOINT/CMD 清晰，不要在 entrypoint 里写复杂 shell 逻辑。
2. PID 1 行为：正确处理信号；必要时使用 `--init`。
3. 资源限制：至少设置内存上限与 CPU 上限；防止邻居噪音。
4. 重启策略：用 `restart: unless-stopped`（或根据场景选择），避免异常退出无人接管。
5. 健康检查：为关键服务提供轻量 healthcheck。

参考：`5_container.md`、`12_resource_limits.md`、`16_logging_monitoring.md`。

# 3. 数据与存储（Data）

1. 状态数据用命名 volume（数据库、队列等）。
2. 配置与证书用 bind mount 且只读。
3. 备份恢复有脚本与演练：不要等故障才第一次恢复。
4. 谨慎使用 `docker compose down -v`：它会删除 volume。

参考：`6_volume.md`。

# 4. 网络（Network）

1. 最小暴露：只暴露入口服务端口，内部依赖不做 `ports` 映射。
2. 服务间互访用内部网络与服务名。
3. 明确边界：区分 internal 网络与对外网络（如需要）。

参考：`7_network.md`。

# 5. 安全（Security）

1. 禁止 `--privileged`（除非你能写出理由和补偿措施）。
2. 以非 root 运行，capabilities 最小化（`cap-drop=ALL` 再按需添加）。
3. 根文件系统只读（可行则启用），可写目录显式挂载。
4. 不挂载 docker.sock 到业务容器。
5. 镜像扫描与基础镜像升级纳入常规流程。

参考：`13_security.md`。

# 6. 日志与可观测（Ops）

1. 日志输出到 stdout/stderr。
2. 配置日志轮转（json-file 必配）。
3. 发生故障优先用：`docker logs`、`docker inspect`、`docker stats`、`docker events`。

参考：`16_logging_monitoring.md`、`19_troubleshooting.md`。

# 7. Compose 使用习惯（单机场景）

1. `depends_on` 只保证启动顺序；配合 `healthcheck` 才能更接近“依赖就绪”。
2. 多环境用 override 文件或 profiles，不要复制多份完全相同的 compose。
3. 用 `docker compose config` 检查最终渲染结果（变量替换/合并覆盖）。

参考：`9_compose.md`、`10_compose_microservices.md`。

# 8. 最小落地模板（可复制）

上线前最小自检：

1. 镜像：固定版本 + 扫描通过 + 不含密钥。
2. 运行：非 root + 资源限制 + 重启策略 + healthcheck。
3. 数据：volume 持久化 + 备份恢复方案。
4. 网络：只暴露必要端口。
5. 日志：轮转配置已启用。
