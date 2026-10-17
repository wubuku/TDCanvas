---
title: TDCanvas 技术架构说明
status: draft
scope: research
---

# TDCanvas 技术架构说明

> **目的**：回答 TDCanvas 由哪些运行时、模块和数据边界组成，以及一次画布操作如何经过前端、服务适配层和桌面原生层。
> **状态**：调研草稿；未进入发布文档导航。
> **证据范围**：当前仓库的 `web/`、`web/src-tauri/`、`modules/comfyui-local/`、`canvas-agent/`、`plugins/`、`docs/`、现有开发文档和 ComfyUI PRD。

## 结论

当前项目**没有一份完整的、跨模块的技术架构说明**。已有文档分别覆盖了局部主题：

- `docs/content/docs/development/canvas-data-structure.zh-CN.mdx`：画布项目、节点和本地媒体数据结构。
- `docs/content/docs/development/local-development.zh-CN.mdx`：本地开发与桌面启动。
- `docs/content/docs/development/local-codex-canvas.zh-CN.mdx`：Canvas Agent、MCP 和 Codex/Claude 连接。
- `docs/product/comfyui-local-mode-prd.md` 与 `modules/comfyui-local/README.md`：ComfyUI 本地模块的目标和实现边界。
- `plugins/canvas/README.md`：画布节点插件 SDK 和发布方式。

这些文档没有共同说明以下关系：Web 前端与 Tauri 桌面壳的边界、状态和媒体的持久化分层、画布插件如何进入节点注册表、ComfyUI 模块如何接入宿主、Agent 如何通过本地进程回写画布，以及各部分分别如何构建。因此本稿补足“项目级”架构视图，但仍保留在 `docs/drafts/`，避免在未 review 前修改 upstream-facing 文档。

当前实现可以概括为：

1. `web/` 是主要应用前端，使用 React + TypeScript + Vite；浏览器运行时用于开发和预览，正式用户入口是 Tauri 2 桌面客户端。
2. Tauri 负责窗口生命周期、文件系统、HTTP、下载目录、媒体缓存和 ComfyUI 本地进程/HTTP 桥接；前端通过 `services/platform/` 和 Tauri 插件调用原生能力。
3. 画布是一个由 Zustand 状态、节点注册表、画布操作协议和插件运行时组成的宿主；内置节点与外部插件节点共享 `CanvasNodeDefinition`/`CanvasNodeContext` 约定。
4. ComfyUI 是可选的本地外部运行时，不是 TDCanvas 的内置推理引擎。TDCanvas 保存工作流定义和画布宏节点，通过 Tauri 原生插件管理或连接本机 ComfyUI，再调用其本地 HTTP API。
5. Canvas Agent 是独立的 Node.js 本地进程，向网页提供 HTTP 接口并提供 MCP Server；它再连接 Codex 或 Claude Code，将经过确认的画布操作交回网页。
6. 项目文档站、Canvas Agent、ComfyUI Local Module 和主 Web/Tauri 应用各自有独立的构建入口，不是同一个运行时或同一个依赖图。

## 1. 总体组件视图

```text
用户
 │
 ├─ TDCanvas Desktop（Tauri 2）
 │    ├─ WebView：web/ 的 React/Vite 应用
 │    │    ├─ 页面、布局、画布组件
 │    │    ├─ Zustand stores + React Query
 │    │    ├─ 画布节点注册表 / 插件运行时
 │    │    ├─ API、媒体、WebDAV、Agent 服务适配器
 │    │    └─ ComfyUI Local 宿主集成
 │    └─ Rust 原生层：web/src-tauri/
 │         ├─ Tauri 官方插件（HTTP、FS、Dialog、Process、Updater）
 │         ├─ TDCanvas 媒体缓存与桌面命令
 │         └─ tdcanvas-comfyui-local 插件
 │
 ├─ 浏览器开发/预览运行时（Vite）
 │    └─ 同一套 Web UI；不具备需要 Tauri 的桌面能力
 │
 ├─ 可选的本地 TDCanvas Agent（Node.js）
 │    ├─ HTTP/SSE：网页侧边栏和画布连接
 │    ├─ MCP：Codex CLI / Agent 访问画布工具
 │    └─ Codex app-server / Claude Code adapter
 │
 ├─ 可选的本机 ComfyUI 进程
 │    └─ 由 Tauri 插件启动或连接，通过 127.0.0.1 端口提供 ComfyUI API
 │
 └─ 外部服务
      ├─ AI 土豆或其他 OpenAI/Gemini/Ark 兼容 API
      ├─ WebDAV 服务器
      └─ 远程画布插件静态资源/官方插件注册表
```

这是一张运行时关系图，不表示所有目录都会被同一个构建产物打包。`docs/` 是独立的 Next.js/Fumadocs 文档站；`canvas-agent/` 是独立 Node.js 包；`modules/comfyui-local/` 的 TypeScript 核心和 Rust Tauri 插件分别参与不同构建阶段。

## 2. 仓库目录与职责边界

| 目录 | 当前职责 | 运行/构建关系 |
| --- | --- | --- |
| `web/` | 主应用 Web 前端、页面、画布、服务适配器和 Zustand 状态 | `npm run dev`/`npm run build`；Tauri 构建把 `web/dist` 作为前端资源 |
| `web/src-tauri/` | Tauri 2 桌面壳、Rust 命令、媒体缓存、窗口启动流程 | `tauri dev/build` 直接编译；依赖 `modules/comfyui-local/tauri-plugin` |
| `web/src/integrations/comfyui-local/` | ComfyUI 模块在宿主中的接入点：native client、工作流库、宏节点、执行和结果节点 | 主 Web 构建的一部分；通过 `index.ts` 接入 `modules/comfyui-local` |
| `modules/comfyui-local/contracts/` | ComfyUI 工作流、输入输出、环境和执行结果的跨层类型协议 | 无 React/Tauri 业务逻辑，供 TS 层共享类型 |
| `modules/comfyui-local/core/` | API Format JSON 解析、节点检查、输入输出暴露、工作流物化 | 可独立 TypeScript 构建和测试 |
| `modules/comfyui-local/frontend/` | 将 Tauri 命令封装成 `createComfyNativeClient` | 只依赖 contracts 的调用类型和 `invoke` 函数 |
| `modules/comfyui-local/tauri-plugin/` | 环境检测、进程生命周期、本地 HTTP、上传、队列、轮询和输出处理 | Rust crate，由桌面 Tauri 工程链接 |
| `canvas-agent/` | 本地 Agent 的 HTTP/SSE、MCP、画布会话和 Codex/Claude 适配 | 独立 Node.js 包；不随 Web bundle 自动打包 |
| `plugins/canvas/` | 画布插件 SDK、模板、插件示例和官方插件发布约定 | 插件独立构建为 ESM，再由 Web 加载器安装/启用 |
| `plugins/tdcanvas/` | 给 Codex 使用的 TDCanvas 插件和 Skill | Agent/开发工具侧资源，不是画布节点运行时 |
| `docs/` | Next.js + Fumadocs 文档站及 MDX 内容 | 独立 `bun run build`/`npm run build` |
| `openspec/` | 变更提案、设计和规格文件 | 设计/协作资料，不是运行时模块 |
| `docs/drafts/` | research fork 的调研证据、构建记录和 review 草稿 | 不进入发布文档导航 |

### 2.1 ComfyUI 宿主接入的实际边界

产品设计文档建议由 `web/src/integrations/comfyui-local/index.ts` 作为唯一宿主接入点。当前代码确实集中在这个入口导出 contracts/core，并创建 `comfyNativeClient`；但它使用相对路径直接导入 `modules/comfyui-local/*/src`，而不是消费已经发布的 npm 包或 `dist` 产物。这意味着：

- 逻辑边界已经被拆开，便于研究和单元测试；
- 当前构建边界仍是“源码直接纳入 Web 构建”，不是完全独立发布的模块边界；
- 如果将来要把模块单独发布或由多个宿主复用，需要额外决定包产物、版本和构建顺序。

这是当前实现与 `docs/product/comfyui-local-mode-prd.md` 中“独立模块”描述之间的一个待 review 差异，本稿不在此处修改实现。

## 3. Web 前端运行时

### 3.1 启动与路由

`web/src/main.tsx` 在 React 根节点渲染前完成三项全局初始化：分析、原生右键菜单保护和 ComfyUI 工作流节点注册，然后挂载 `AppProviders` 与路由。

`web/src/router.tsx` 根据运行时选择路由实现：

- Tauri 桌面使用 `createHashRouter`，让打包后的静态资源可以在 WebView 内直接刷新；
- 浏览器开发/预览使用 `createBrowserRouter`。

当前主要路由是画布首页/项目、素材、提示词、ComfyUI 本地和配置页；所有页面都包在 `UserLayout` 中，并共享顶部导航和 Agent 面板。

### 3.2 Provider 与跨页面基础设施

`web/src/components/layout/app-providers.tsx` 提供：

- Ant Design `ConfigProvider`、Pro Components 和中英文 locale；
- React Query 的全局 `QueryClient`；
- 桌面启动就绪上报、窗口标题栏和 `ClientRootInit`；
- 主题同步：WebView 的 `color-scheme` 与桌面窗口主题同步。

因此页面组件通常不直接负责运行时判断，而是通过 stores、服务适配器和现有 hooks 取得能力。`web/src/services/platform/desktop-runtime.ts` 对 `isTauriRuntime`、`invokeDesktop`、Tauri HTTP、桌面文件 URL 和本地文件读取做了统一封装。

### 3.3 画布宿主

画布项目的核心结构是 `CanvasProject`：项目元数据、`nodes`、`connections`、Agent 对话会话、背景和 viewport。节点的具体业务状态放在 `CanvasNodeData.metadata`，动态端口和资源引用由画布类型与插件协议解释。

画布运行时由几层组成：

1. `web/src/stores/canvas/use-canvas-store.ts` 保存项目列表，并通过 Zustand `persist` 持久化。
2. `web/src/lib/canvas/node-registry.ts` 保存节点定义和插件归属，提供注册、卸载、查找和默认规格。
3. `web/src/types/canvas-plugin.ts` 定义宿主向节点注入的 `CanvasNodeContext`、操作列表、端口和资源协议。
4. `web/src/lib/canvas/plugin-node-context.ts` 把项目、节点、主题、连线查询、`applyOps`、AI 能力和插件存储组合成节点上下文。
5. 画布项目页维护交互期的节点/连线/viewport/history 状态，再把项目变更写回 `useCanvasStore`。

这种分层让插件节点可以使用统一的画布操作，而不用直接依赖某个页面组件的内部 state。插件仍然在页面上下文中执行，不是沙箱进程。

## 4. 状态、存储与数据所有权

### 4.1 状态分层

| 数据 | 当前所有者 | 持久化方式 | 说明 |
| --- | --- | --- | --- |
| 画布项目 | `useCanvasStore` | `localForage` 的 `tdcanvas/app_state`，保存节流约 400ms | 项目 JSON 不应长期内嵌大媒体 |
| 我的素材 | `useAssetStore` | 同一 `localForage` 应用状态库 | 重新 hydration 时解析媒体 `storageKey` |
| 图片 Blob | `image-storage.ts` | 独立 `tdcanvas/image_files` 实例 | 节点只保留 URL、`storageKey` 和元信息 |
| 视频/音频/文件 Blob | `file-storage.ts` | 独立媒体存储实例 | 通过 `storageKey` 解析展示 URL |
| AI 配置/API Key | `useConfigStore` | Zustand 默认 Web `localStorage` 持久化 | 当前 `partialize` 将配置规范化为官方 Aitudou-only shape，并保留 API Key；读取时再次规范化 |
| 插件记录 | `usePluginStore` | `localForage` | 记录 URL、缓存源码、启用状态和版本 |
| ComfyUI 工作流库 | `workflow-library.ts` | 独立 `tdcanvas-comfyui-local/workflow_library` | 不直接放进主画布 store |
| WebDAV 配置 | `useConfigStore.webdav` | 与配置 store 一起由 Zustand 管理 | 同步数据本身通过 `app-sync.ts` 组织为 domain manifest |
| 桌面媒体缓存 | Rust `MediaCacheState` | Tauri `app_local_data_dir()/media-cache` | 使用 asset protocol 暴露稳定桌面地址 |
| ComfyUI 环境配置/进程所有权 | Rust ComfyUI plugin | `app_local_data_dir()/comfyui-local/environments.json` 与 `process-ownership.json` | 由原生进程管理器维护 |

### 4.2 画布与媒体的关系

画布 JSON 与媒体二进制是两套存储：

```text
CanvasProject / Asset
  └─ metadata.content 或 data.url（当前展示地址）
  └─ storageKey（稳定的本地引用）
       ├─ 浏览器：IndexedDB/localForage Blob
       └─ 桌面：Tauri AppLocalData 媒体缓存或 ComfyUI 结果文件
```

`image-storage.ts`、`file-storage.ts` 和 `local-media-cache.ts` 负责在展示 URL、Blob、桌面文件 URL 和远程 URL 之间转换；删除项目或素材后会根据仍被引用的 `storageKey` 清理未使用媒体。WebDAV 同步由 `app-sync.ts` 按 `canvas` 与 `assets` 两个 domain 合并 JSON，并携带文件清单，不把浏览器/桌面路径当成跨设备稳定 ID。

## 5. 外部 API 与服务适配层

`web/src/services/api/` 是前端业务请求的统一入口，包含：

- `aitudou.ts`、`image.ts`、`video.ts`、`audio.ts`：图片、视频、音频和通用任务调用；
- `aitudou-protocol.ts`、`aitudou-contract.ts`、`aitudou-task-journal.ts`：Aitudou 响应协议、任务恢复和结果归一化；
- `model-plugin.ts`：模型自定义脚本/协议扩展；
- `canvas-agent.ts`：对本地 Canvas Agent 的 HTTP 接口封装；
- `prompts.ts`、`prompt-source-runtime.ts`：提示词来源和运行时加载。

`useConfigStore` 用 channel 的 `baseUrl`、`apiKey`、`apiFormat` 和模型能力选择请求配置；`buildApiUrl` 负责把不同渠道的 base URL 规范化为请求地址。桌面端需要 Tauri 网络权限时，通过 `services/platform/desktop-runtime.ts` 的 `platformFetch` 或 Tauri 原生命令完成，而不是依赖 Vite 开发服务器代理作为生产能力。

这里的“服务层”不是后端服务：大部分请求由 WebView 直接发往用户配置的 API；Tauri 主要提供受控 HTTP、文件和本地进程能力。AI API Key 保存在当前客户端配置中，不能把它理解成由 TDCanvas 云端托管。

## 6. Tauri 桌面层

### 6.1 主应用原生层

`web/src-tauri/src/lib.rs` 注册 Tauri 官方插件和 TDCanvas 自有能力：

- HTTP、Dialog、FS、Process、Updater（按构建 feature 启用）；
- 启动状态和 splashscreen/main window 显示时序；
- 打开下载目录、AI 土豆注册页、ComfyUI WebUI；
- 允许用户选择下载目录；
- 远程媒体下载到 `AppLocalData/media-cache` 并以 asset protocol 返回给 WebView。

`web/src-tauri/tauri.conf.json` 将 `web/dist` 配置为 `frontendDist`，开发时用 `http://127.0.0.1:3000`，并只把 `media-cache` 与 `comfyui-local/results` 纳入 asset protocol scope。桌面启动时，前端在画布 store hydration 后调用 `frontend_ready`，Rust 侧再关闭 splashscreen 并显示主窗口。

### 6.2 平台边界

- Web 浏览器可以运行画布和远程 API 功能，但不能调用 Tauri 命令、桌面文件 URL 或本机 ComfyUI 进程管理。
- 正式桌面包由 Tauri 按宿主平台构建；macOS `.app/.dmg` 和 Windows 安装包不应在另一平台假设可完整生成。
- 可写业务文件应进入系统 AppLocalData/Application Support，不写入安装目录或源码仓库。

## 7. ComfyUI Local Module

### 7.1 分层

```text
web/src/integrations/comfyui-local/
  ├─ workflow-library.ts       # 浏览器本地工作流库
  ├─ workflow-import-wizard.tsx# 导入、依赖、输入输出暴露
  ├─ canvas-node.tsx           # ComfyUI 宏节点和动态端口
  ├─ execution.ts              # 画布资源收集、提交、轮询、结果回填
  └─ index.ts                  # 宿主接入点

modules/comfyui-local/
  ├─ contracts/                # 跨层协议类型
  ├─ core/                     # API JSON 检查与工作流物化
  ├─ frontend/                 # Tauri invoke client
  └─ tauri-plugin/             # Rust 环境/进程/HTTP/文件实现
```

### 7.2 执行链路

```text
导入 API Format JSON
  → parseComfyApiWorkflow / inspectComfyWorkflow
  → 结合 object_info 解析输入、输出和缺失节点
  → 保存 ComfyWorkflowDefinition 到本地工作流库
  → 创建 comfyui-local:workflow 宏节点
  → 画布连接提示词/图片/音频/视频等资源
  → materializeComfyWorkflow 生成本次提交 JSON
  → Tauri invoke(queue_workflow)
  → ComfyUI /prompt
  → Tauri 插件轮询 /history/{prompt_id}
  → 解析输出并写入结果节点/本地媒体地址
```

Tauri 插件的命令面包含环境选择、环境启动/连接/停止、状态和日志、系统信息、`object_info`、输入上传、工作流入队、执行等待和中断。它把 ComfyUI 当作本机的外部进程/API，不把 ComfyUI 内部每个运行时节点都映射为 TDCanvas 节点；画布只暴露用户选定的输入输出。

### 7.3 生命周期与队列

- 已保存的环境 profile 由 Rust registry 管理；启动的子进程由 `ComfyProcessManager` 记录并在退出/插件 drop 时回收。
- 可连接一个已经运行的本机 ComfyUI；断开 TDCanvas 不应默认终止不归 TDCanvas 所有的外部进程。
- 任务执行状态由前端 `ComfyExecutionManager` 以 `projectId + nodeId` 维度跟踪；原生层以 `prompt_id` 轮询历史并决定删除等待项还是中断当前任务。
- 结果节点是画布节点，不是 ComfyUI 输出目录的直接视图；结果路径、视频容器和跨画布任务恢复仍需按 `docs/content/docs/progress/pending-test.mdx` 的验收项复核。

## 8. Canvas Agent 与外部自动化

`canvas-agent/` 是独立的本地 Node.js 进程，不是 Tauri 的 Rust 子模块，也不是 Web bundle 的一部分。它包含：

- `src/server/http.ts`：网页连接、token、SSE/HTTP 请求和结果回传；
- `src/server/mcp.ts`：向 Codex/其他 MCP 客户端暴露工具；
- `src/canvas/session.ts`、`operations.ts`、`tools.ts`：画布快照、操作校验和会话状态；
- `src/agent/codex.ts`、`codex-client.ts`、`claude.ts`：Agent 进程适配和事件归一化。

典型链路是：

```text
Codex/Claude
  → MCP tool call
  → canvas-agent
  → HTTP/SSE 到 TDCanvas WebView
  → CanvasSession / applyOps
  → 画布页面状态与 useCanvasStore
```

写操作仍由网页侧确认；Agent 协议需要同时按 `threadId`、`turnId` 和 `itemId` 归属消息，不能只按时间顺序合并实时事件。这是当前项目注意事项中明确的可靠性边界。

## 9. 画布插件系统

画布插件是运行在 WebView 页面内的 ESM 代码：

1. `plugins/canvas/sdk/` 提供类型、宿主 React、JSX runtime 和构建助手。
2. 插件导出 `CanvasPlugin`，其中包含节点定义、可选 CSS 和 setup/cleanup。
3. `web/src/lib/canvas/plugin-loader.ts` 从 `web/public/plugins/index.json`、官方注册表或用户 URL 获取源码；源码会被评估并注册节点定义。
4. `node-registry.ts` 按 `type → pluginId` 维护节点定义；禁用插件时注销其节点。
5. `plugin-runtime.ts` 向远程插件提供宿主 React、事件总线和 CSS 注入能力。

插件代码可直接访问页面上下文和浏览器本地数据，可能接触 AI API Key；因此插件安装不是权限隔离机制，用户只能安装可信来源的插件。这个风险已经在 `plugins/canvas/README.md` 中说明。

ComfyUI 宏节点目前走宿主集成路径，不是通过远程插件注册表下载；它使用同一套节点注册表和 `CanvasNodeContext`，但其本地进程能力来自 Tauri 插件。

## 10. 构建与交付拓扑

| 目标 | 工作目录 | 主要命令/入口 | 产物或运行结果 |
| --- | --- | --- | --- |
| Web 开发 | `web/` | `npm run dev` | Vite 开发服务器 |
| Web 静态构建 | `web/` | `npm run build` | `web/dist` |
| 桌面开发 | `web/` | `npm run desktop:dev` | Tauri 窗口 + Vite dev server |
| 桌面本机测试包 | `web/` | `npm run desktop:build` | Tauri 本地 bundle |
| 正式桌面打包 | `web/` | `npm run desktop:package` | Windows NSIS 或 macOS `.app/.dmg`；正式更新包需要签名配置 |
| ComfyUI TS 模块 | `modules/comfyui-local/` | `npm run verify` | `dist`、TypeScript 测试结果 |
| Canvas Agent | `canvas-agent/` | `npm run build` | `dist/index.js` 和 `tdcanvas-agent` 命令 |
| 文档站 | `docs/` | `bun run build` 或 `npm run build` | Next.js standalone 文档站 |
| 画布插件 | `plugins/canvas/<name>/` | 插件自己的 `npm run build` | `dist/<name>.js`，可同步到 `web/public/plugins/` |

桌面构建的关键配置在 `web/src-tauri/tauri.conf.json` 与 `web/scripts/package-desktop.mjs`：Tauri 构建会先执行 Vite build；正式 updater 包还需要 endpoint、公钥和签名私钥。不要把 docs build、Canvas Agent build 或插件 build 当作桌面包的隐式步骤，它们是独立交付物。

## 11. 当前边界、差异与待确认项

以下内容是从代码和文档对照得出的 review 项，不代表本次已经修复：

1. **缺少项目级架构文档**：本稿是补充草稿，尚未进入 `docs/content/docs/` 或导航。
2. **ComfyUI 模块的“独立”程度**：设计文档按 contracts/core/frontend/tauri-plugin 划分模块，但 Web 当前直接引用模块源码；需要决定未来是否改为消费构建产物。
3. **配置协议与持久化范围不完全一致**：API service 和 `AiConfig` 类型仍保留 OpenAI/Gemini/Ark/Aitudou 等协议概念，但 `useConfigStore` 的更新、`partialize` 和 `merge` 会把持久化配置规范化为官方 Aitudou channel；不能仅凭类型定义宣称所有渠道都已作为同等生产能力验证。
4. **进度文档可能落后于实现**：`todo.mdx` 仍把部分 ComfyUI 阶段列为待办，而模块 README 和当前代码已经包含环境、导入和执行闭环；应由产品/开发 review 后再调整状态文档。
5. **媒体结果路径仍需验收**：Tauri asset scope、ComfyUI 结果目录和媒体缓存已经存在，但不同输出类型、视频容器、刷新和跨画布恢复仍以 `pending-test.mdx` 为准，不在架构说明中宣称全部通过。
6. **浏览器与桌面能力不同**：浏览器运行时能加载 UI，但 ComfyUI 环境管理、桌面媒体 URL、下载目录和部分网络权限只有 Tauri 可用；文档和功能入口不应把两者描述成完全等价。
7. **安全边界是本地信任边界**：API Key、WebDAV 凭证、插件源码和 Agent token 都在本机范围内使用；远程插件与本地 Agent 不应被视为无权限的沙箱。

## 12. 验证记录

本次仅进行了静态调研：

- 检查了现有文档目录、`docs/drafts/`、主应用入口、路由、Provider、stores、画布注册表、服务层、Tauri 配置、ComfyUI 模块、Canvas Agent 和插件 SDK。
- 用仓库相对路径核对了本文引用的主要文件。
- 未运行应用构建、模块测试、桌面打包或端到端验收；本文没有把未执行的流程描述为已验证。
- 按 `project-docs` 技能要求检查了 `todo.mdx` 和 `pending-test.mdx`；本次是研究文档补充，没有修改进度文档。

## 13. 推广建议

如果后续需要把架构说明纳入正式文档，建议只做最小增量：

1. 将本稿压缩为“运行时、模块边界、数据持久化、构建入口”四个小节，目标位置为 `docs/content/docs/development/architecture.zh-CN.mdx`。
2. 保留 ComfyUI、Agent、插件和数据结构的细节在各自专题文档中，正式架构页只链接它们，不复制全文。
3. 在推广前先确认 ComfyUI 模块是否继续直接引用源码，并同步更新 `todo.mdx`/`pending-test.mdx` 中已经变化的状态。
4. 不修改 upstream 的 README、根级 AGENTS、CHANGELOG 或导航页，除非用户明确要求发布该架构说明。

## 证据入口

- `web/src/main.tsx:1-21` — Web 应用初始化和 ComfyUI 节点注册。
- `web/src/router.tsx:1-34` — 浏览器/Tauri 路由差异和主要页面入口。
- `web/src/components/layout/app-providers.tsx:20-66` — 全局 Provider、主题和桌面启动组件。
- `web/src/stores/canvas/use-canvas-store.ts:10-66` — `CanvasProject` 和画布持久化入口。
- `web/src/lib/localforage-storage.ts:1-33` — 主应用状态的 localForage/localStorage 适配。
- `web/src/stores/use-config-store.ts:201-246` — AI 配置持久化与 API Key 规范化。
- `web/src/services/platform/desktop-runtime.ts:7-69` — Tauri runtime 判断、invoke、HTTP 和文件 URL。
- `web/src/lib/canvas/node-registry.ts:8-60` — 节点定义与插件归属注册表。
- `web/src/lib/canvas/plugin-loader.ts:9-113` — 插件源码加载、启用和注销。
- `web/src/types/canvas-plugin.ts`、`web/src/lib/canvas/plugin-node-context.ts:7-33` — 画布插件宿主协议和上下文。
- `web/src-tauri/src/lib.rs:157-223` — Tauri 插件、媒体缓存和命令注册。
- `web/src-tauri/tauri.conf.json:1-80` — Web 资源、桌面窗口、CSP 和 asset protocol scope。
- `web/src/integrations/comfyui-local/index.ts:1-7` — ComfyUI 模块的宿主接入点。
- `web/src/integrations/comfyui-local/execution.ts:19-69` — 画布宏节点到 native client 的执行链路。
- `modules/comfyui-local/contracts/src/index.ts`、`core/src/workflow-inspector.ts`、`core/src/workflow-execution.ts` — ComfyUI 协议、解析和工作流物化。
- `modules/comfyui-local/frontend/src/native-client.ts:15-71` — 前端到 Tauri 命令的 ComfyUI client。
- `modules/comfyui-local/tauri-plugin/src/lib.rs:398-441`、`:888-990` — 原生命令注册、环境管理、`/history` 轮询和中断。
- `canvas-agent/src/index.ts`、`canvas-agent/src/server/http.ts`、`canvas-agent/src/server/mcp.ts`、`canvas-agent/src/canvas/session.ts` — 本地 Agent 的进程、HTTP/MCP 和画布会话。
- `plugins/canvas/README.md`、`web/src/lib/canvas/plugin-runtime.ts:6-41` — 画布插件 SDK、宿主 React 和信任边界。
- `web/package.json`、`web/scripts/package-desktop.mjs`、`docs/package.json` — 各构建入口和交付物。
- `docs/content/docs/development/`、`docs/product/comfyui-local-mode-prd.md` — 既有专题文档与设计上下文。
