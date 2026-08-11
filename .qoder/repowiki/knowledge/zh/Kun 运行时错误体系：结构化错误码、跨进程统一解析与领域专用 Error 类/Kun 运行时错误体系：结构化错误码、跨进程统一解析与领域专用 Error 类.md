---
kind: error_handling
name: Kun 运行时错误体系：结构化错误码、跨进程统一解析与领域专用 Error 类
category: error_handling
scope:
    - '**'
source_files:
    - kun/src/contracts/errors.ts
    - src/shared/runtime-error.ts
    - packages/extension-api/src/errors.ts
    - src/main/index.ts
    - src/renderer/src/agent/kun-runtime.ts
    - kun/src/tui/client.ts
    - kun/src/adapters/tool/mcp-types.ts
    - kun/src/adapters/browser-use/browser-controller.ts
    - kun/src/graph/graph-reducer.ts
    - examples/extensions/kun-video-editor/src/engine/errors.ts
---

## 1. 整体方案

本仓库采用**「结构化错误码 + 共享解析器 + 领域专用 Error 子类」**的分层错误处理体系，覆盖三个边界：

- **Kun HTTP/SSE 运行时契约**（`kun/src/contracts/errors.ts`）：用 Zod 枚举 `KunErrorCode` 定义所有后端端点返回的 `code`，并约束 `{ code, message, details? }` 三元体。
- **主进程 / 渲染器 / TUI 共享运行时错误归一化**（`src/shared/runtime-error.ts`）：提供 `parseRuntimeErrorBody`、`runtimeErrorToError`、`isKnownKunErrorCode`、`isLegacyMainGuardCode` 等工具，把后端 JSON、旧版 `error: string` 以及纯字符串 body 统一成 `RuntimeError { code, message, details? }`，再包装为 JS `Error` 抛出。
- **扩展 API 错误**（`packages/extension-api/src/errors.ts`）：独立定义 `ExtensionErrorCode` 枚举和 `ExtensionApiError` 类，带 `operation`、`extensionId`、`retryable`、`details`、`documentation` 字段，并通过 `from(value)` 静态方法做安全降级。

此外，各子模块还按领域自定义 `class XxxError extends Error`（如 `BrowserControllerError`、`McpAuthorizationRequiredError`、`GraphReducerError`、`VideoEngineError` 等），用于在进程内传播更精确的错误语义；这些类型不跨 IPC 边界，仅作为进程内异常类型使用。

## 2. 关键文件与位置

| 职责 | 文件路径 | 说明 |
|---|---|---|
| 运行时错误契约 | `kun/src/contracts/errors.ts` | Zod 定义的 `KunErrorCode`、`RuntimeErrorSeverity`、`KunErrorBody` |
| 跨进程错误归一化 | `src/shared/runtime-error.ts` | `parseRuntimeErrorBody`、`runtimeErrorToError`、已知代码白名单 |
| 扩展 API 错误 | `packages/extension-api/src/errors.ts` | `ExtensionErrorCode`、`ExtensionApiError`、`DiagnosticSchema` |
| Electron 主进程入口 | `src/main/index.ts` | `runtimeRequest`/`runtimeRequestOnLease` 统一 catch → `parseRuntimeErrorBody` → `runtimeFailure` |
| 渲染器运行时客户端 | `src/renderer/src/agent/kun-runtime.ts` | 每个 HTTP 调用都 `throw runtimeErrorToError(readRuntimeError(...))` |
| 领域专用 Error 类 | `kun/src/adapters/**`、`kun/src/graph/**`、`examples/extensions/**` | 进程内细粒度异常 |

## 3. 架构与约定

### 3.1 运行时错误码分层

- **Kun 运行时错误码**（`KunErrorCode`）：`validation_error`、`unauthorized`、`forbidden`、`not_found`、`conflict`、`rate_limited`、`turn_in_progress`、`turn_not_running`、`approval_not_pending`、`capability_unavailable`、`provider_unavailable`、`policy_blocked`、`model_modality_unsupported`、`attachment_validation_failed`、`internal_error`、`not_implemented`、`aborted`。
- **遗留主进程守卫码**（`LegacyMainGuardCode`）：`runtime_auth_required`、`runtime_request_failed`、`fetch_failed`、`runtime_offline`、`runtime_port_conflict`、`runtime_unhealthy`、`runtime_request_user_input_unsupported`、`missing_api_key`。
- 两者合并为 `RuntimeErrorCode`，未知值一律回退到 `unknown`。

### 3.2 解析与转换流程

```text
HTTP/SSE body (JSON 或字符串)
  → parseRuntimeErrorBody(body, fallback)
    → 尝试 JSON.parse；失败则取 trim 后的 body
    → 从 record.code 或 record.error 提取 code
    → 从 record.message / record.error.string / nested.message 取 message
    → 保留 record.details（可选）
  → runtimeErrorToError(RuntimeError)
    → 若 code === 'unknown' 且无 details，直接 new Error(message)
    → 否则将 { code, message, details } 序列化为 JSON 字符串放入 Error.message
```

### 3.3 调用方约定

- **Electron 主进程**（`src/main/index.ts`）：`runtimeRequest`/`runtimeRequestOnLease` 捕获所有 throw，先 `logError`，再用 `parseRuntimeErrorBody` 判断是否已携带结构化 code；若是则走 `runtimeFailure(parsed.code, ...)`，否则标记为 `fetch_failed`。`runtimeJsonError` 通过 `runtimeErrorToError` 构造可被上层捕获的 Error。
- **渲染器**（`src/renderer/src/agent/kun-runtime.ts`）：每个对 Kun 运行时的 HTTP 调用都 `throw runtimeErrorToError(readRuntimeError(response.body, 'failed to ...'))`，使 UI 层可通过 `instanceof Error` 捕获并按 `code` 分支处理。
- **TUI**：通过 `client.ts` 中的 Zod schema 校验响应，未匹配时由 zod 抛错；SSE 事件通过 `IncrementalSseParser`/`parseRuntimeEventFrame` 解析，错误以 `lastError` 字段暴露给 CLI 输出。
- **扩展宿主**：`ExtensionApiError.from(value)` 会安全降级非标准对象为 `INTERNAL_ERROR`，保证扩展不会因宿主异常崩溃。

### 3.4 进程内领域错误

各子系统自行定义 `class XxxError extends Error` 并在同进程内 throw/catch，例如：
- `adapters/model/gemini-cli-api-model-client.ts` 的 `GeminiCliApiHttpError`
- `adapters/tool/mcp-types.ts` 的 `McpAuthorizationRequiredError`
- `graph/graph-reducer.ts` 的 `GraphReducerError`
- `examples/extensions/kun-video-editor/src/engine/errors.ts` 的 `VideoEngineError`

这些类型不序列化到 IPC，仅用于进程内控制流。

## 4. 约定与约束

| 约定 | 来源/证据 |
|---|---|
| 所有 Kun HTTP/SSE 端点必须返回 `{ code, message, details? }` 结构 | `kun/src/contracts/errors.ts` 中 Zod `KunErrorBody` 注释“mirrors what Kun diagnostics can render” |
| 未知错误码一律归并为 `unknown`，禁止透传任意字符串 | `src/shared/runtime-error.ts` 中 `normalizeCode` 使用白名单 Set |
| 主进程对外部请求失败统一归类为 `fetch_failed` | `src/main/index.ts` 中 `runtimeRequest`/`runtimeRequestOnLease` catch 分支 |
| 渲染器侧每个运行时调用都必须经 `runtimeErrorToError` 包装后抛出 | `src/renderer/src/agent/kun-runtime.ts` 中每条调用均如此 |
| 扩展 API 错误必须实现 `ExtensionApiError` 并通过 `from()` 构造 | `packages/extension-api/src/errors.ts` 中 `ExtensionErrorSchema` 与 `from` 静态方法 |
| 进程内细分错误使用 `class XxxError extends Error` 而非字符串 | 多处适配器/图执行器/视频引擎中的自定义 Error 类 |
| 错误消息长度受约束（扩展 API 限制 message ≤ 4096、code ≤ 128） | `packages/extension-api/src/errors.ts` 中 Zod `.min/.max` |
| 诊断信息使用统一 `DiagnosticSchema`（severity: info/warning/error） | `packages/extension-api/src/errors.ts` |

该体系确保：**跨进程边界的错误始终是可枚举、可向后兼容的结构化码；进程内的复杂状态用领域 Error 类表达；未知或损坏数据永远回退到安全的 `unknown`/`INTERNAL_ERROR`。**