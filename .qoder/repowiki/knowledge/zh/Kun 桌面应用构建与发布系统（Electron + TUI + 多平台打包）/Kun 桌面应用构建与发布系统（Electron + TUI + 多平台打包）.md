---
kind: build_system
name: Kun 桌面应用构建与发布系统（Electron + TUI + 多平台打包）
category: build_system
scope:
    - '**'
source_files:
    - package.json
    - electron.vite.config.ts
    - electron-builder.config.cjs
    - .github/workflows/release.yml
    - .github/workflows/pr-checks.yml
    - scripts/release-mac.sh
    - scripts/release-win.ps1
    - scripts/lib/release-common.sh
    - scripts/publish-r2.mjs
    - scripts/package-tui.mjs
    - scripts/assemble-tui-release.mjs
    - scripts/compute-ci-release-version.cjs
    - scripts/pack-bundled-extensions.mjs
    - scripts/check-extension-release-gate.mjs
    - scripts/smoke-packaged-extensions.cjs
    - scripts/smoke-packaged-cli.cjs
    - scripts/verify-packaged-macos-native-architecture.cjs
    - scripts/check-package-size.cjs
    - scripts/mac-notarize.cjs
    - scripts/after-pack.cjs
---

## 1. 构建体系概览

本项目是一个基于 npm workspaces 的 Monorepo，根工程 `package.json` 通过 `workspaces: ["packages/*"]` 聚合四个子包（extension-api、extension-react、extension-test、create-kun-extension），并作为 Kun 桌面 Electron 客户端与独立 TUI 运行时的统一构建入口。构建栈由以下组件构成：

- **源码编译**：`electron-vite`（`electron.vite.config.ts`）分别构建 main、preload、renderer 三个产物；`tsconfig.{web,node,build}.json` 区分 Web/Node/构建时类型检查。
- **依赖构建**：`npm run build:kun` 先构建 `@kun/provider-catalog`、`@kun/extension-api`，再执行 `scripts/ensure-kun-install.cjs` 安装本地 Kun 运行时，最后 `npm --prefix kun run build` 编译核心 Agent 运行时。
- **扩展打包**：`scripts/pack-bundled-extensions.mjs` 将 `examples/extensions` 下的扩展预编译为 `.kunx` 包并写入 `resources/bundled-extensions/catalog.json`，随主程序一起分发。
- **桌面端打包**：`electron-builder@26.8.1`（`electron-builder.config.cjs`）负责 macOS dmg/zip、Windows NSIS exe、Linux AppImage/deb 三平台打包，输出到 `dist/`。
- **TUI 独立打包**：`scripts/package-tui.mjs` 将 Kun CLI 以 `pkg` 方式打包为跨平台可执行文件，产出 `Kun-TUI-<version>-<platform>.tar.gz|zip`。
- **CI/CD**：GitHub Actions（`.github/workflows/release.yml`、`pr-checks.yml`）在 PR 合并到 master 时触发全量 release 流水线，在 PR 阶段做三平台构建校验。

## 2. 关键文件与职责

| 文件 | 作用 |
|---|---|
| `package.json` | workspace 定义、所有构建/测试/打包脚本入口（`build`、`dist:*`、`release:*`、`smoke:*`） |
| `electron.vite.config.ts` | electron-vite 多入口配置（main/index.ts、claw-schedule-mcp-node-entry、preload 四入口、renderer index.html/tray-quota.html） |
| `electron-builder.config.cjs` | 打包产物清单、asar 白名单、extraResources（whisper、officecli、bundled-extensions）、签名/公证开关、更新通道、artifactName 模板 |
| `scripts/release-mac.sh` | macOS 发布流程：版本计算 → 签名 → 构建 dmg/zip → 冒烟测试 → GitHub Release 上传 → R2 元数据上传 |
| `scripts/release-win.ps1` | Windows 发布流程：读取 Mac 产出的 `.release-meta.env` → 构建 NSIS → 冒烟测试 → 上传 → R2 promote/publish |
| `scripts/lib/release-common.sh` | 共享工具：semver 校验、channel 归一化、git tag 管理、锁文件、clean dist、签名环境变量注入 |
| `scripts/publish-r2.mjs` | 将各平台产物及 TUI 合约上传至 Cloudflare R2，支持 `upload` / `promote` 双模式 |
| `scripts/compute-ci-release-version.cjs` | CI 中根据上次 tag 自动递增 patch 版本号 |
| `scripts/assemble-tui-release.mjs` | 收集四平台 TUI 产物生成 `release-tui.json` 与 `SHA256SUMS-tui.txt` |
| `.github/workflows/release.yml` | 完整 release 流水线：prepare → build-macos → build-windows → build-linux → build-tui → publish |
| `.github/workflows/pr-checks.yml` | PR 阶段并行构建 Linux/macOS/Windows 安装包并上传 artifact |

## 3. 架构与设计决策

### 3.1 构建产物分层
- **GUI 应用**：`out/main`、`out/preload`、`out/renderer` 经 asar 打包，但 `asarsUnpack` 显式排除 native 模块（better-sqlite3、node-pty、sharp、@computer-use/*、tesseract.js 等）以保证运行时加载。
- **内置运行时**：`kun/dist/**` 被完整打入 asar，使 GUI 启动时可复用已构建的 Agent 运行时。
- **第三方二进制**：`resources/whisper`（跨平台 whisper.cpp 预编译）、`resources/officecli/current`（OfficeCLI 选择器）通过 `extraResources` 原样复制到 `app.asar` 外的 `resources/` 目录。
- **扩展资源**：`resources/bundled-extensions` 仅包含 `catalog.json` 和 `*.kunx` 包，不内联源码。

### 3.2 版本与更新通道
- `package.json.version` 必须为 `x.y.z` 三段 semver，否则 electron-updater 拒绝升级。
- 更新通道通过 `KUN_UPDATE_CHANNEL`（兼容旧 `DEEPSEEK_GUI_UPDATE_CHANNEL`）限定为 `stable` 或 `frontier`，非法值直接抛错。
- 开发版使用独立 appId `com.xingyuzhong.deepseekgui.dv`，正式版用 `com.xingyuzhong.deepseekgui`，避免 Squirrel.Mac 签名锚定冲突。
- 更新 URL 固定为 `${R2_PUBLIC_BASE_URL}/${R2_RELEASE_PREFIX}/channels/${channel}/latest/`，默认 `https://www.kun-agent.com/api/r2/deepseek-gui/channels/stable/latest/`。

### 3.3 签名与公证
- macOS：当存在 `CSC_LINK`/`CSC_KEY_PASSWORD`/`APPLE_API_KEY_ID`/`APPLE_API_ISSUER`/`MAC_SIGN=1` 任一标志时启用 Developer ID 签名 + hardenedRuntime；notarization 通过 `afterSign: ./scripts/mac-notarize.cjs` 自定义处理（支持 `APPLE_API_KEY_BASE64`）。
- Windows/Linux：`CSC_IDENTITY_AUTO_DISCOVERY=false` 关闭自动签名，NSIS 安装器通过 `build/installer.nsh` 定制目录迁移逻辑。
- CI 中 macOS 任务从 secrets 解码 P12/P8 证书后写入临时路径，再设置 `MAC_SIGN=1` 触发签名链。

### 3.4 发布流水线
- **PR 阶段**（`pr-checks.yml`）：并行构建 Linux AppImage/deb、macOS ad-hoc dmg/zip、Windows NSIS，失败则通过 GitHub Script 对 PR 打 `REQUEST_CHANGES`。
- **Release 阶段**（`release.yml`）：顺序执行 prepare → build-macos（带签名公证）→ build-windows → build-linux → build-tui（四平台矩阵）→ publish（验证全部产物 → 创建 draft Release → 上传 assets → 上传 R2 → promote latest → 取消 draft）。
- **本地发布**：`scripts/release-mac.sh` 仅在 macOS 上运行，生成 draft GitHub Release；`scripts/release-win.ps1` 要求先有 Mac 创建的 tag，再补齐 Windows 产物后统一 promote/publish。

### 3.5 质量门禁
- `check:extension-release-gate` 在打包前校验扩展 schema、文档、示例、发布就绪状态。
- 每个平台构建后执行冒烟测试：`smoke:packaged-cli`、`smoke:packaged-extensions`、`smoke:packaged-extension-desktop`、`smoke:extension-native-media`、`smoke:packaged-ocr`。
- 原生依赖架构校验：`verify:packaged-macos-native-architecture.cjs` 确认打包产物中的 .node/.dylib 架构与目标 arch 一致。
- 体积门禁：`check:package-size.cjs --enforce` 限制 macOS dmg/zip 大小。
- 生产审计：`audit:production` 扫描依赖漏洞。

## 4. 约定与约束

- **Node 版本锁定**：`engines.node >= 22.19.0`，CI 统一使用 Node 22。
- **workspace 构建顺序**：必须先 `npm run build:extensions`（provider-catalog → extension-api → extension-react → extension-test → create-kun-extension），再 `npm run build:kun`，最后 `electron-vite build`。
- **环境变量命名**：新变量统一使用 `KUN_*` 前缀，同时兼容旧 `DEEPSEEK_GUI_*` 前缀（如 `KUN_APP_VERSION`/`DEEPSEEK_GUI_APP_VERSION`、`KUN_UPDATE_CHANNEL`/`DEEPSEEK_GUI_UPDATE_CHANNEL`、`KUN_RELEASE_ENV`/`DEEPSEEK_GUI_RELEASE_ENV`）。
- **本地发布环境文件**：优先从 `KUN_RELEASE_ENV` 指定的文件加载，回退到 `scripts/release.local.env`、`release.local.env`，格式为 `KEY=VALUE` 键值对。
- **GitHub Release 产物命名**：GUI 遵循 `Kun-${version}-${os}-${arch}.${ext}`，TUI 遵循 `Kun-TUI-${version}-${platform}.${ext}`，blockmap 与 yml 同步生成。
- **R2 发布策略**：macOS 仅上传单平台元数据，需等待 Windows 完成后统一 `--r2-promote`；promotion 强制要求 mac+win+linux+TUI 四者齐全。
- **安全约束**：`asarUnpack` 列表是白名单机制，新增 native 模块必须显式加入，否则无法在 asar 内加载。
- **Linux 沙箱参数**：AppImage 通过 `executableArgs: ['--disable-setuid-sandbox', '--no-first-run']` 禁用 Chromium 沙箱，deb/AppImage 入口均通过产品 launcher 注入相同参数。
- **缓存与并发**：`ELECTRON_BUILDER_CACHE` 指向 `.cache/electron-builder`；GitHub Release 上传使用 `RELEASE_UPLOAD_CONCURRENCY`（默认 4）并发上传，避免串行瓶颈。
- **锁机制**：`release-common.sh` 通过 `.cache/release.lock` 目录互斥，防止同一机器并发执行多个 release 脚本。
- **分支策略**：release 流水线仅在 `develop` 分支 PR 合并到 `master` 时触发，且 head repo 必须与当前仓库一致，防止外部 fork 滥用。

## 5. 适用性说明

本仓库存在完整的构建与发布系统：npm workspaces + electron-vite 编译、electron-builder 多平台打包、自研 shell/PowerShell 发布脚本、GitHub Actions CI/CD 流水线、R2 制品分发、签名公证流程、以及覆盖 GUI/TUI/扩展/原生依赖的冒烟测试门禁。因此本类别完全适用。