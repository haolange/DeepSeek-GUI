---
kind: configuration_system
name: Kun 桌面应用配置系统：用户设置、项目配置与运行时配置的三层架构
category: configuration_system
scope:
    - '**'
source_files:
    - src/main/settings-store.ts
    - src/main/settings-file-paths.ts
    - src/shared/app-settings.ts
    - src/shared/app-settings-types.ts
    - src/shared/app-settings-provider.ts
    - src/shared/app-settings-kun.ts
    - src/shared/app-settings-graph.ts
    - src/shared/app-settings-normalizers.ts
    - src/shared/app-settings-schedule.ts
    - src/shared/app-settings-workflow.ts
    - src/shared/app-settings-claw.ts
    - src/shared/app-settings-write.ts
    - src/shared/app-settings-design.ts
    - src/shared/app-settings-terminal.ts
    - src/shared/app-settings-domain.ts
    - src/shared/model-provider-presets.ts
    - src/shared/secret-redaction.ts
    - src/main/settings-credential-redaction.ts
    - kun/src/config/kun-config.ts
    - kun/src/config/project-config.ts
---

## 1. 整体方案

Kun 采用**三层配置体系**，分别面向用户偏好、工作区项目和本地 Agent 运行时，全部基于 **Zod schema + JSON 文件**实现类型安全的配置加载、校验与迁移。

| 层级 | 配置文件 | 位置 | 作用域 | 主要用途 |
|---|---|---|---|---|
| 用户设置（App Settings） | `kun-settings.json`（兼容 `deepseek-gui-settings.json`） | Electron userData 目录 | 单用户/多 userData | GUI 主题、语言、模型 Provider、通知、快捷键、Write/Claw/Schedule/Workflow/Design/Terminal 等应用级开关 |
| 项目配置（Project Config） | `.kun/project.json` | 工作区根目录下 `.kun/` | 单个 Git 工作区 | MCP 服务器、Skills 启用与根路径、禁用 ID |
| 运行时配置（Kun Config） | `config.json` | 由 `dataDir` 决定（默认在 userData 下） | Kun 本地 HTTP/SSE Agent 运行时 | serve 端点、Provider 路由、上下文压缩、Graph 模式、能力开关、Hooks、质量检查、Lab 实验特性 |

## 2. 核心文件与职责

- `src/main/settings-store.ts`：Electron 主进程中的 `JsonSettingsStore`，负责用户设置的读取、合并、持久化、补丁更新、修订冲突重试、凭据迁移与备份恢复。
- `src/main/settings-file-paths.ts`：定义 `SETTINGS_FILE_NAME = 'kun-settings.json'`、旧文件名 `deepseek-gui-settings.json` 以及跨 userData 目录的兼容读取候选顺序。
- `src/shared/app-settings.ts`：统一导出所有共享设置模块（`app-settings-types`、`app-settings-provider`、`app-settings-kun`、`app-settings-graph`、`app-settings-prompts`、`app-settings-normalizers`、`app-settings-schedule`、`app-settings-workflow`、`app-settings-claw`、`app-settings-write`、`app-settings-design`、`app-settings-terminal`、`app-settings-domain`、`model-provider-presets` 等），每个子模块提供 `defaultXxxSettings()`、`mergeXxxSettings()`、`normalizeXxxSettings()`。
- `kun/src/config/kun-config.ts`：Kun 运行时配置，使用 Zod schema 定义 `KunConfigSchema`，包含 `serve`、`models`、`contextCompaction`、`runtime`、`graph`、`roles`、`capabilities`、`lab`、`hooks`、`quality` 等字段；提供 `readKunConfigFile`、`readOptionalKunConfigFile`、`kunConfigPathForDataDir`、`expandHomePath`。
- `kun/src/config/project-config.ts`：项目级 `.kun/project.json` 的 schema、解析、大小限制（256KB）、MCP server 数量上限（32）、skills roots 上限（32）、相对路径安全校验、原子写入（`.project.json.<pid>.<uuid>.tmp` → rename）。
- `kun/src/config/secret-redaction.ts`：运行时配置中的敏感信息脱敏。
- `src/shared/secret-redaction.ts`：GUI 侧设置中的凭据脱敏。
- `src/main/settings-credential-redaction.ts`：GUI 设置中凭据的脱敏处理。

## 3. 架构与设计约定

### 3.1 用户设置（App Settings）

- **默认值集中**：`defaultSettings()` 在 `settings-store.ts` 中构造完整的 `AppSettingsV1`，并通过各 `defaultXxxSettings()` 组合出 provider、agents.kun、write、claw、schedule、workflow、design、terminal 等子对象。
- **规范化管道**：读取后调用 `normalizeStoredSettings` → `normalizeAppSettings`，对路径做 `expandHomePath`（支持 `~` 和 `~/`）、workspaceRoot/conversationWorkspaceRoot/write workspaceRoots 去重、Claw channel/conversation 的 workspaceRoot 生成默认值。
- **补丁式更新**：`applySettingsPatchToSnapshot` 通过 `mergeXxxSettings` 合并部分更新，避免整份覆盖。
- **持久化策略**：`JsonSettingsStore` 内部缓存 + 队列串行化（`enqueueOperation`），写盘使用 `atomicWriteFile`；可选 `documentBackend` 接口支持外部 CAS 后端并带 revision 冲突重试（最多 2 次）。
- **兼容读取**：`settingsReadCandidates` 按顺序尝试当前 userData、旧文件名、历史 userData 目录名（`deepseek-gui`、`DeepSeek GUI`）下的两种文件名。
- **凭据迁移**：检测遗留明文 `apiKey`，通过 `credentialMigration.prepare` 将凭据移出设置文件，写入 `.pre-extension-credential-migration.json` 备份，失败时回滚；若受保护凭据存储不可用则拒绝写入明文。
- **无效恢复**：JSON 解析失败或 top-level 非对象时，自动备份为 `*.invalid-<timestamp>.json` 并回退到默认设置。

### 3.2 项目配置（Project Config）

- **固定版本**：`KUN_PROJECT_CONFIG_VERSION = 1`，不支持向后兼容多个版本，仅支持向前兼容未知字段（strict schema）。
- **安全约束**：路径必须相对 workspace、不能逃逸工作区、不能是符号链接、文件大小 ≤ 256KB、MCP servers ≤ 32、skills roots ≤ 32、record 键数 ≤ 64。
- **原子写入**：先写临时文件（权限 `0o600`），再 `rename` 到目标，保证并发安全。
- **Digest 计算**：`stableJson` 对对象键排序后哈希，用于变更检测。

### 3.3 运行时配置（Kun Config）

- **严格 Schema**：所有顶层段均使用 `.strict()`，未知字段直接报错。
- **前向兼容**：当 GUI 写入了 TUI 不认识的 capability 字段时，`parseForwardCompatibleKunConfig` 只保留已知的 `serve`、`models`、`contextCompaction`、`runtime`、`roles`、`hooks`、`quality` 段及白名单 capability 子段，其余丢弃。
- **遗留迁移**：`normalizeLegacyProviderKinds` 将旧 `gemini-cli-subscription` kind 映射为 `gemini-cli-api`。
- **路径展开**：`expandHomePath` 支持 `~`、`~/`、`~\` 三种写法。
- **数据目录定位**：`kunConfigPathForDataDir` 根据 `dataDir` 拼接 `config.json` 路径。

## 4. 约定与约束

- **所有配置均为 JSON 文本**，通过 Zod schema 在运行时解析并返回强类型对象。
- **路径一律支持 `~` 展开**，由各自模块的 `expandHomePath` 实现。
- **用户设置文件命名**：新文件 `kun-settings.json`，旧文件 `deepseek-gui-settings.json` 仍被兼容读取。
- **项目配置必须位于工作区内**：任何 relative path 经 `realpath` 解析后必须仍在 workspace root 内，防止逃逸。
- **项目配置目录权限**：`.kun/` 以 `0o700` 创建，项目配置文件以 `0o600` 写入。
- **凭据不得明文落盘**：检测到遗留明文 `apiKey` 时会触发凭据迁移流程，若受保护存储不可用则拒绝保存。
- **设置更新走 patch/update/updateIf API**，禁止直接替换整个快照，以保证 merge 逻辑与规范化一致。
- **运行时配置支持灰度发布**：`graph.rolloutStage` 枚举（experimental/alpha/beta/learning-preview/stable）控制 Graph 模式可见性。
- **配置大小限制**：项目配置最大 256KB，防止恶意大文件攻击。
- **并发安全**：设置写盘使用原子写入；项目配置写盘使用临时文件 + rename；`JsonSettingsStore` 内部操作串行化并支持 revision 冲突重试。

## 5. 关键文件清单

- `src/main/settings-store.ts`
- `src/main/settings-file-paths.ts`
- `src/shared/app-settings.ts`
- `src/shared/app-settings-types.ts`
- `src/shared/app-settings-provider.ts`
- `src/shared/app-settings-kun.ts`
- `src/shared/app-settings-graph.ts`
- `src/shared/app-settings-prompts.ts`
- `src/shared/app-settings-normalizers.ts`
- `src/shared/app-settings-schedule.ts`
- `src/shared/app-settings-workflow.ts`
- `src/shared/app-settings-claw.ts`
- `src/shared/app-settings-write.ts`
- `src/shared/app-settings-design.ts`
- `src/shared/app-settings-terminal.ts`
- `src/shared/app-settings-domain.ts`
- `src/shared/model-provider-presets.ts`
- `src/shared/secret-redaction.ts`
- `src/main/settings-credential-redaction.ts`
- `kun/src/config/kun-config.ts`
- `kun/src/config/project-config.ts`
- `kun/src/config/secret-redaction.ts`
- `examples/extensions/kun-video-editor/src/engine/index.ts`（示例中使用的 project config 参考）
- `docs/KUN_CONFIG.md`（运行时配置文档）