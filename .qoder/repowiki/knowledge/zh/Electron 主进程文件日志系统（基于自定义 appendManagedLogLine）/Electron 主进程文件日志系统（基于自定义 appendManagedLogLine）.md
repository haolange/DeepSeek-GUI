---
kind: logging_system
name: Electron 主进程文件日志系统（基于自定义 appendManagedLogLine）
category: logging_system
scope:
    - '**'
source_files:
    - src/main/logger.ts
    - src/main/logger.test.ts
    - src/main/index.ts
    - src/main/kun-process.ts
    - src/main/runtime/kun-runtime-subagent-config.ts
    - src/shared/app-settings-types.ts
---

## 1. 使用的系统与框架

本仓库**没有引入第三方日志库**（如 winston、pino、bunyan、debug 等）。Electron 主进程使用自实现的轻量级文件日志模块 `src/main/logger.ts`，通过 Node.js `fs/promises` 直接写入按日期分片的 `.log` 文本文件。渲染器与 TUI（`kun/`）则直接使用原生 `console.warn` / `console.error` 输出，未接入统一日志框架。

## 2. 核心文件

- `src/main/logger.ts`：日志子系统唯一实现，导出 `configureLogger`、`logError`、`logWarn`、`logInfo`、`appendManagedLogLine`、`pruneOnStartup`。
- `src/shared/app-settings-types.ts`：定义默认保留天数常量 `DEFAULT_LOG_RETENTION_DAYS = 3`。
- `src/main/index.ts`：应用启动入口，负责解析日志目录、调用 `configureLogger`、监听设置变更热更新日志配置、在启动时调用 `pruneOnStartup`。
- `src/main/kun-process.ts`、`src/main/runtime/kun-runtime-subagent-config.ts`：通过 `appendManagedLogLine('kun', ...)` 将子进程/子 agent 的原始 stdout/stderr 透传到管理日志文件。
- `src/main/logger.test.ts`：Vitest 测试，验证结构化 detail 字段被正确序列化到日志文件中。

## 3. 架构与约定

### 3.1 日志级别
仅支持三个级别：`'error' | 'warn' | 'info'`（类型 `LogLevel`），分别对应 `logError`、`logWarn`、`logInfo`。无 debug 级别。

### 3.2 日志格式
每行固定格式：
```
[ISO时间戳] [LEVEL] [category] message — detail: <safeStringify(detail)>
```
其中 `category` 由调用方传入（如 `'sse'`、`'startup-settings'`、`'weixin-bridge'`、`'logger'`），用于区分来源模块；`detail` 可选，会被 `safeStringify` 安全序列化为 JSON 字符串并截断至 2000 字符。

### 3.3 文件组织
- 文件名形如 `<prefix>-YYYY-MM-DD.log`，前缀限定为 `ManagedLogFilePrefix = 'deepseek-gui' | 'kun'`。
- 所有写操作先 `mkdir({ recursive: true })`，再 `appendFile`，确保目录存在且追加写入。
- 每次写入后触发 `pruneOldLogs()`，扫描目录中匹配 `^\(deepseek-gui|kun\)-.*\.log$` 的文件，删除 mtime 早于 `retentionDays × 24h` 的旧文件。
- 所有 I/O 异常均被 try/catch 吞掉，保证“never crash the app because of logging”。

### 3.4 生命周期集成
- 启动阶段（`src/main/index.ts`）：解析 `resolveLogDirectory(app)` → `configureLogger({ dir, enabled: initial.log.enabled, retentionDays: initial.log.retentionDays })` → 稍后调用 `pruneOnStartup()`。
- 运行时热更新：当 `applySettingsPatch` 检测到 `previous.log.enabled` 或 `previous.log.retentionDays` 变化时，立即 `configureLogger({ enabled, retentionDays })` 重新注入配置。
- 子进程日志桥接：`kun-process.ts` 捕获 Kun 子进程的 stdout/stderr，经 `formatKunLogLine` 格式化后通过 `appendManagedLogLine('kun', ...)` 写入同一日志目录。

### 3.5 可观测性边界
- 日志开关与保留期来自应用设置 `settings.log.enabled` / `settings.log.retentionDays`，可通过 IPC 动态修改。
- 日志目录路径由 Electron `app.getPath('logs')` 或自定义逻辑解析得到（`resolveLogDirectory`）。
- 启动 trace（`traceStartup`）仍走 `console.info`，独立于文件日志。

## 4. 约定与约束

- **必须通过 `logError` / `logWarn` / `logInfo` 写入受管日志**：这些函数封装了时间戳、级别、分类和 detail 序列化，避免散落式 `console.log` 进入持久化日志。
- **category 参数必填**：每个日志调用必须提供字符串 category，作为日志行的 `[category]` 字段，便于后续过滤与检索。
- **detail 最大 2000 字符**：`safeStringify` 强制截断，防止超大 payload 撑爆日志文件。
- **日志文件命名前缀受限**：仅允许 `deepseek-gui` 与 `kun` 两个前缀，新增前缀需修改 `MANAGED_LOG_FILE_PREFIXES`。
- **保留策略不可绕过**：`pruneOldLogs` 在每次写入后执行，无法禁用；可通过 `enabled: false` 完全关闭日志写入。
- **TUI / 扩展代码不依赖此 logger**：`kun/src/...` 与示例扩展普遍使用 `console.warn` / `console.error` 直接输出，不在该文件日志体系内。
- **测试覆盖**：`src/main/logger.test.ts` 验证了 `logInfo` 能将结构化对象（含嵌套字段）序列化进日志文件，并通过 `vi.waitFor` 异步断言内容。

## 5. 适用性说明

该日志系统仅服务于 Electron 主进程（`src/main`）及其管理的子进程（Kun runtime、WeChat bridge 等）。渲染器进程、TUI 引擎、扩展宿主均未接入此模块，而是各自使用 `console.*` 输出。因此它是一个**主进程专用的、极简的、基于文件系统追加写入的日志子系统**，而非跨进程的统一日志框架。