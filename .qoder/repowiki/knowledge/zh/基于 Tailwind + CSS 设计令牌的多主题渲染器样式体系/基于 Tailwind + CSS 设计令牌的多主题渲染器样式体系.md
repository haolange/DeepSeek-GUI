---
kind: frontend_style
name: 基于 Tailwind + CSS 设计令牌的多主题渲染器样式体系
category: frontend_style
scope:
    - '**'
source_files:
    - tailwind.config.js
    - postcss.config.js
    - src/renderer/src/index.css
    - src/renderer/src/styles/base-shell.css
    - src/renderer/src/styles/graph-workbench.css
    - src/renderer/src/styles/write-editor.css
    - src/renderer/src/styles/write-rich-editor.css
    - src/renderer/src/styles/markdown-code.css
    - src/renderer/src/styles/settings-layout.css
    - src/renderer/src/styles/workflow-canvas.css
    - src/renderer/src/styles/provider-quota-panel.css
    - src/renderer/src/styles/tray-provider-quota.css
---

## 1. 系统/方法概述

Kun 桌面应用的 UI 样式由 **Tailwind CSS**（PostCSS 构建）驱动，配合一套以 `--ds-*` 前缀命名的 **CSS 自定义属性（设计令牌）**，实现浅色、深色、iKun 专属配色与 Retroma 羊皮纸主题的运行时切换。样式入口位于 `src/renderer/src/index.css`，通过 `@tailwind base/components/utilities` 注入基础层；所有业务组件的类名均使用 Tailwind 原子类组合，并通过 `@apply` 在 `components` 层定义少量全局控件（如 `.primary-button`、`.secondary-button`、`.settings-input`）。

主题切换机制：根元素上设置 `data-theme="dark"` 或 `data-ikun-mode="on"` / `data-retroma-mode="on"`，Tailwind 配置中 `darkMode: ['selector', '[data-theme="dark"]']` 使暗色模式按选择器触发而非媒体查询。

## 2. 关键文件

- `tailwind.config.js` — 扩展 Tailwind 颜色空间，将 `accent`、`control`、`background`、`foreground`、`border`、`muted`、`sidebar`、`primary`、`ds.*` 等映射到 `var(--ds-*)` 令牌；声明 `boxShadow` 与 `borderRadius` 扩展。
- `postcss.config.js` — 启用 `tailwindcss` 与 `autoprefixer`。
- `src/renderer/src/index.css` — 应用 Tailwind 指令，定义通用按钮与输入控件的 `@layer components`。
- `src/renderer/src/styles/base-shell.css` — 核心设计令牌定义：`--ds-bg-*`、`--ds-surface-*`、`--ds-border-*`、`--ds-text-*`、`--ds-accent*`、`--ds-shadow-*`、`--ds-radius-*`、`--ds-scrollbar-*` 等，并包含 light/dark/iKun/Retroma 多套主题覆盖块；同时提供窗口拖拽区域、Windows/Linux 标题栏、滚动条、composer shell 等全局样式。
- `src/renderer/src/styles/*.css` — 按功能域划分的样式模块：`graph-workbench.css`、`write-editor.css`、`write-rich-editor.css`、`markdown-code.css`、`neutral-polish.css`、`provider-quota-panel.css`、`tray-provider-quota.css`、`workflow-canvas.css`、`settings-layout.css`、`surfaces-write.css`。
- `examples/extensions/*/src/webview/styles.css` — 扩展 Webview 内也遵循同一 `data-theme` 主题约定（light/high-contrast），体现主题策略在扩展侧的复用。

## 3. 架构与约定

- **令牌优先**：所有颜色、圆角、阴影、间距语义都通过 `--ds-*` 变量暴露，组件仅引用这些变量，不直接写死色值。Tailwind 配置把 `bg-ds-*`、`text-ds-*`、`border-ds-*` 等实用类绑定到对应变量，从而让原子类成为主题安全的 API。
- **主题开关分层**：`base-shell.css` 中 `:root` 定义默认浅色系；`[data-theme='dark']` 覆盖为暖炭暗色；`[data-ikun-mode='on']` 与 `[data-ikun-mode='on'][data-theme='dark']` 分别覆盖浅色与深色下的 iKun 专属黄橙配色；`[data-retroma-mode='on']:not([data-theme='dark'])` 限定 Retroma 羊皮纸主题仅在浅色生效。这种“基线 + 主题覆盖”的方式允许任意组合。
- **平台适配**：`[data-platform='darwin']`、`win32/linux` 下调整 `--ds-window-controls-safe-*`、`--ds-windows-titlebar-height` 等令牌，统一处理 macOS traffic-light 与 Windows/Linux 自绘标题栏差异。
- **响应式/容器查询**：使用 Tailwind 内置断点及 `@container`（如 `workspace-mode-tabs` 的 `inline-size` 容器）做局部响应式布局。
- **可访问性**：焦点环统一通过 `color-mix(in srgb, var(--ds-focus-ring) 18%, transparent)` 生成；`focus-visible` 替代 `:focus`；禁用文本选择时区分可编辑区域（`input/textarea/[contenteditable]` 恢复 `user-select: text`）。
- **Electron 集成**：通过 `-webkit-app-region: drag/no-drag` 控制窗口拖拽区域；`color-scheme` 随主题切换；`zoom: var(--ds-ui-scale)` 支持全局缩放。

## 4. 约定与约束

- 新增视觉 token 必须通过 `--ds-*` 变量声明，并在 Tailwind `theme.extend.colors.ds` 中注册别名，禁止在组件中直接使用原始色值。
- 主题切换仅通过修改根节点 `data-*` 属性完成，不得依赖 `prefers-color-scheme`。
- 组件样式优先使用 Tailwind 原子类；需要复用的控件形态才放入 `index.css` 的 `@layer components`（如 `.primary-button`、`.secondary-button`、`.settings-input`）。
- 暗色模式相关规则统一放在 `[data-theme='dark']` 选择器下，避免污染亮色上下文。
- 扩展 Webview 需沿用相同的 `data-theme` 约定，以保证 Kun 主进程与扩展视图的主题一致性。
- 平台差异通过 `data-platform` 与 `--ds-*` 令牌表达，不在组件中硬编码平台判断。
- 滚动条、选区、字体族等全局视觉细节集中在 `base-shell.css`，组件不应重复定义。