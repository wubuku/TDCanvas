---
title: TDCanvas 用户 Agent 与 Skill 机制调研
status: draft
scope: research
---

# TDCanvas 用户 Agent 与 Skill 机制调研

> **目的**：确认当前项目是否存在面向 AI 影视制作者的用户 Agent，梳理它的实际能力，并回答 Agent 是否可以使用、配置和生成 Skill。
> **状态**：调研草稿；未进入发布文档导航。
> **证据范围**：检查 `web/` 的 Agent 面板与状态管理、`web/src-tauri/` 的桌面打包/原生入口、`canvas-agent/` 的 HTTP/SSE、Codex app-server、MCP 和 Skill 实现、`plugins/tdcanvas/skills/`、ComfyUI Local 集成、相关测试及现有开发文档。本文只依据当前源码和仓库文档，不把未运行验证的行为写成已验收事实。

## 结论

### 已确认

1. **当前项目确实有面向最终用户的 Agent 入口**：用户布局始终挂载右侧 `AgentPanel`，其中的 `LocalAgentPanel` 可以连接本机 TDCanvas Agent，并在面板里与 Codex 对话、查看历史和操作当前画布（`web/src/layouts/user-layout.tsx:6-14`、`web/src/components/agent/agent-panel.tsx:12-60`）。
2. 这个 Agent **不是 TDCanvas 内置的云端影视专用模型**，而是“网页画布 + 本机 TDCanvas Agent + Codex app-server”的本地桥接方案；仓库也保留了一个调用 Claude CLI 的兼容入口。真正的推理、规划和 Skill 解释执行由 Codex/Claude 负责，TDCanvas 负责提供画布上下文、MCP 工具、权限边界和结果回传（`canvas-agent/README.md:1-3`、`canvas-agent/src/agent/codex.ts:176-186`、`canvas-agent/src/agent/claude.ts:7-18`）。
3. 它**可以用于 AI 影视制作**，但当前定位是通用的画布创作/工作流编排 Agent：可以把剧本、提示词、参考素材和文本/图片/视频/音频生成节点组织成流程；代码中没有发现一个独立的“导演 Agent”“分镜 Agent”或面向影视领域的固定规划器。影视方法论可以通过 Skill 加进去，但那属于用户配置后的行为，不是内置领域能力。
4. **Codex 路径可以使用 Skill**。网页选择的 Skill 会随 `/agent/codex/turn` 请求发送到本机 Agent；本机 Agent 会重新从 Codex 原生 `skills/list` 结果校验名称、绝对路径和启用状态，然后把 Skill 作为 Codex turn input 传入（`web/src/components/agent/local-agent-panel.tsx:646-687`、`canvas-agent/src/server/http.ts:303-305`、`canvas-agent/src/agent/codex.ts:123-146`、`canvas-agent/src/agent/codex-client.ts:812-820`）。
5. **Skill 可以配置和管理**，但要区分“发现”和“可编辑管理”：Agent 面板支持查看、搜索、按 scope 过滤、启用/停用、选择、读取详情、创建、编辑和删除；只有位于站点工作空间 `.agents/skills` 标准入口下的 Skill 被 `SkillStore` 标记为 `managed`，可由这套 UI 修改。其他 `user`、`repo`、`system`、`admin` scope 的原生 Codex Skill 可以被发现，但通常不属于 TDCanvas SkillStore 的可编辑范围（`web/src/components/agent/agent-skills-view.tsx:20-70`、`canvas-agent/src/skills/store.ts:63-80`）。
6. Skill 可以从当前**已完成对话**或**当前画布**提炼为草稿。提炼过程使用临时、只读、无 MCP 的 Codex 线程，并做 JSON schema、Zod 和敏感信息检查；草稿不会自动写文件，必须经过编辑器确认保存（`web/src/components/agent/agent-skills-view.tsx:91-120`、`canvas-agent/src/server/http.ts:176-207`、`canvas-agent/src/agent/codex.ts:233-271`、`canvas-agent/src/agent/codex-client.ts:790-810`）。

### 需要避免的误解

- “有 Agent”不等于“有影视行业专用 Agent”。当前实现提供的是可调用工具的通用 Agent 运行时，影视能力主要来自画布节点、生成服务、可选的 ComfyUI 工作流宏节点、MCP 工具和用户 Skill 的组合。
- “可以使用 Skill”不等于 TDCanvas 自己解析并执行 `SKILL.md`。Skill 的发现、启用状态和 turn 输入由 TDCanvas 转发，具体解释和遵循由 Codex app-server 完成。
- `plugins/tdcanvas/skills/` 是随仓库插件提供的 Skill 资产；Agent 面板管理的用户/项目 Skill 则位于本机站点工作空间的 `.agents/skills/`。两者相关但不是同一个目录或同一个生命周期。
- `/agent/claude/turn` 当前只拼接 `agent-instructions.md` 并通过 Claude CLI 开启 `mcp__tdcanvas__*` 工具，没有看到与 Codex Skill 选择器等价的参数通路。因此本文关于“可使用、可管理 Skill”的结论以 Codex 面板路径为主，不能直接推广到 Claude 兼容入口。
- “画布里有 ComfyUI 节点”也不等于“当前 Agent 可以直接提交 ComfyUI 工作流”。当前 `canvas-agent` 没有 ComfyUI 专用 MCP 工具；ComfyUI 宏节点的实际运行仍由 `comfyui-local` 插件自己的节点面板和执行管理器完成。

### 面向最终用户的桌面版与浏览器版结论

**桌面版和浏览器版都包含同一个 Agent 面板 UI，但在已检查的构建与部署入口中，没有发现 Agent 进程被打包进桌面客户端，也没有发现 TDCanvas 自己部署的云端 Agent 后端。** 两种入口的区别主要在“承载 Web UI 的容器”和“是否具备 Tauri 原生能力”，不是两套不同的 Agent 实现。面板代码允许配置自定义 Agent URL，但这不等于仓库已经提供了可供多个用户使用的远程 Agent 服务。

- `web/src/router.tsx:1-33` 根据 `isTauriRuntime()` 选择 `HashRouter` 或 `BrowserRouter`，但两者都把所有用户路由包在同一个 `UserLayout` 中；`UserLayout` 无条件挂载 `AgentPanel`（`web/src/layouts/user-layout.tsx:6-14`）。因此，Tauri 桌面 WebView 和普通浏览器加载的 Vite/静态 Web 页面都会渲染 Agent 面板。
- Agent 面板本身使用浏览器的 `fetch`、`EventSource` 和本地存储来连接 `http://127.0.0.1:17371`；没有通过 `invokeDesktop()`、Tauri FS 或 Tauri Process 来运行 Agent（`web/src/components/agent/local-agent-panel.tsx:341-400`、`web/src/services/platform/desktop-runtime.ts:7-27`）。
- Docker/Render/Vercel 的 Web 部署只构建并提供 `web/dist` 静态资源。根 `Dockerfile` 的运行阶段是 Nginx，明确没有启动 `canvas-agent`；Vercel 配置也只有 Vite build 和静态输出（`Dockerfile:1-20`、`render.yaml:1-10`、`vercel.json:1-7`）。
- 用户如果想在浏览器版使用 Agent，需要在**同一台用户电脑**上另外启动 `canvas-agent`，在面板里填写 Local URL 和 Connect token，或通过 URL 参数自动连接；浏览器再用 SSE/HTTP 直接访问本机 Agent（`web/src/components/agent/agent-connect-view.tsx:12-151`、`web/src/components/agent/local-agent-panel.tsx:896-945`）。
- 本机 Agent 服务只监听 `127.0.0.1`，大多数状态与 Agent API 通过 token 和首次授权的网页 Origin 保护；`/health` 与 `/config` 则是用于发现服务的公开端点。服务还返回 `Access-Control-Allow-Private-Network: true`；这使“线上页面 + 用户本地 Agent”成为源码支持的目标路径，但具体线上 HTTPS 域名是否受浏览器混合内容、Private Network Access 或企业策略影响，当前没有端到端验收记录（`canvas-agent/src/server/http.ts:16-25,122-143,508-529`）。

因此，当前实际部署形态是：

```text
桌面版：Tauri WebView（同一套 Web UI）
          │
浏览器版：普通浏览器（同一套 Web UI）
          │
          └── 两者都通过 HTTP/SSE 连接
              用户本机独立运行的 canvas-agent Node 进程
```

如果只访问一个静态部署的网站、又没有在本机启动 `canvas-agent`，面板仍会显示，但只能停留在未连接/设置状态，不能执行 Agent 对话或画布工具。

### Agent 进程不是 Pi，而是 TDCanvas 自己的桥接层 + Codex app-server

当前 Agent 的实现可以拆成三层：

1. **TDCanvas 自己实现的本地桥接服务**：`canvas-agent` 是独立 Node.js 包，使用 Express 提供 HTTP/SSE、线程和 Skill 管理接口；使用 `@modelcontextprotocol/sdk` 提供 `tdcanvas` stdio MCP（`canvas-agent/package.json:1-28`、`canvas-agent/src/server/http.ts:16-25`、`canvas-agent/src/server/mcp.ts:9-31`）。
2. **现成的 Codex app-server**：当 Codex 路径开始运行时，`CodexAppClient` 会用当前 Node.js 启动依赖包中的 `bin/codex.js app-server --stdio` 子进程（效果等价于执行 `codex app-server --stdio`），并通过 JSON-RPC 与其通信（`canvas-agent/src/agent/codex-client.ts:33-100,893-895`）。因此，推理、计划、工具调用决策和 Skill 解释不是 TDCanvas 自己训练或实现的模型能力，而是交给本机依赖的 OpenAI Codex 运行时。
3. **TDCanvas MCP 工具适配**：TDCanvas 把画布读取、节点操作、媒体生成和站点资源能力注册为 MCP 工具；Codex 调用工具后，由本机 Agent 通过 SSE 发给网页，网页在 React/Zustand 状态中执行并回传结果（`canvas-agent/src/canvas/session.ts:435-500`）。

仓库还保留一个可选的 Claude CLI 入口，但它是另外的适配器，并非 Pi，也没有看到与 Codex 面板相同的 Skill 选择和管理通路（`canvas-agent/src/agent/claude.ts:7-18`、`canvas-agent/src/server/http.ts:401-414`）。当前代码中没有发现 Pi 或其他第三方 Agent 框架的接入。

可以用下面的矩阵区分几条容易混淆的入口：

| 入口 | 运行时 | 是否有右侧面板的 Skill 选择/管理 | 是否有当前项目的专用 ComfyUI Agent 工具 |
| --- | --- | --- | --- |
| TDCanvas 右侧 AgentPanel | `canvas-agent` → Codex app-server | 有，走 Codex 原生 Skill 协议 | 没有；只具备通用画布工具 |
| 外部 Codex + `tdcanvas-agent mcp` | 外部 Codex 客户端 → TDCanvas MCP | 由外部 Codex 自己决定，非右侧面板链路 | 没有专用 ComfyUI MCP 工具 |
| `/agent/claude/turn` 兼容入口 | `canvas-agent` → Claude CLI | 当前 HTTP 入口没有 TDCanvas Skill selector | 没有 |

### “本机私有云”类比与多用户边界

把本机 `canvas-agent` + Codex app-server 称作“本机私有云的服务端”是一个有帮助的类比：它确实提供 HTTP/SSE 服务、线程状态、工具调用和模型运行时。但从当前实现看，它更准确是**单机、单租户、单个 TDCanvas Agent 进程内的本地控制面**，不是已经具备多租户隔离的云平台。

当前一个 `canvas-agent` 进程只有：

- 一个监听 `127.0.0.1` 的 HTTP 服务、一个 Connect token 和一个共享的 Origin 列表（`canvas-agent/src/config.ts:6-23`、`canvas-agent/src/server/http.ts:16-25,508-529`）；
- 一个站点 workspace 和一个 `activeThreadId`（`canvas-agent/src/config.ts:33-57`）；
- 一个 `CanvasSession`，虽然可以登记多个 SSE 网页客户端，但只维护一个 `activeClientId`/`boundClientId`，工具调用目标是当前目标网页（`canvas-agent/src/canvas/session.ts:37-79,264-295,435-500`）；
- 一个共享的 Codex app-server 子进程、一个 `loadedThreadId` 和一个全局 turn 队列；所有 Codex mutation 还受到全局 busy 锁限制（`canvas-agent/src/agent/codex.ts:53-73,443-456`、`canvas-agent/src/canvas/session.ts:197-207`）。

因此，当前代码可以支持**同一用户的多个标签页/窗口连接、聚焦切换和线程切换**，但这不是多个页面同时独立运行的会话模型，更不能把多个陌生用户安全地当成互相隔离的租户。多个网页客户端共享本地 Agent 的进程、workspace、Skill 文件、Codex app-server 和会话控制状态；并发 turn 会被串行化，而不是并行运行。

“每个用户/每个会话启动一个 Codex 进程”在架构上当然可行，但不是当前实现的开关，而是一次运行时隔离重构。至少需要把以下对象从全局单例改成按租户/会话索引：

1. `CodexAppClient` 子进程及其 `loadedThreadId`、审批、事件历史和 turn 队列；
2. `CanvasSession` 的连接、目标网页、画布快照、附件和待处理工具调用；
3. workspace、`.agents/skills`、Codex 配置和持久化线程；
4. token、Origin、用户身份、权限模式、资源配额和进程生命周期；
5. MCP 服务启动、日志、崩溃回收和跨用户数据泄漏防护。

只把 `codex app-server` 多启动几个进程，而不隔离上述状态，仍然会产生错误的画布目标、共享 Skill/workspace、审批串线和历史泄漏。对于浏览器云端部署，还必须新增真正的认证、租户数据库、远程画布会话和沙箱策略；当前仓库没有这些组件。

另外，浏览器用户通常**不需要再单独安装一个系统级 Codex CLI**：`canvas-agent` 的依赖已经声明 `@openai/codex: 0.146.0`，并从该依赖包定位 `bin/codex.js`，由 Node Agent 启动 `codex app-server --stdio`（`canvas-agent/package.json:19-28`、`canvas-agent/bun.lock:86-98`、`canvas-agent/src/agent/codex-client.ts:68-100,773-779,893-895`）。但仍需要满足这个 Codex runtime 的账号/认证和网络要求；当前仓库没有实现独立的 Codex 登录服务，也没有在未登录环境做端到端验证。

`canvas-agent/README.md` 中的 `codex mcp add tdcanvas ...` 是“让外部 Codex CLI 主动连接 TDCanvas”的另一条用法，不是浏览器右侧 Agent 面板启动 Codex app-server 的必需步骤（`canvas-agent/README.md:36-56`、`canvas-agent/src/agent/codex-client.ts:781-788`）。

## 1. “给用户的 Agent”实际是什么

### 1.1 用户入口

用户布局把 Agent 侧栏作为应用级 UI 挂载，而不是只在开发页出现。面板由 `LocalAgentPanel` 实现，状态保存在 `useAgentStore`，默认连接本机 `http://127.0.0.1:17371`，并保存 URL、token、权限模式、模型、reasoning effort、线程和消息等状态（`web/src/layouts/user-layout.tsx:6-14`、`web/src/components/agent/local-agent-panel.tsx:68-82`、`web/src/stores/use-agent-store.ts:42-77`、`web/src/stores/use-agent-store.ts:92-127`）。

因此，回答“有没有给 AI 影视制作者的 Agent”时，准确表述应是：

> **有一个面向 TDCanvas 最终用户的本地 Agent 工作台；它可以服务 AI 影视制作，但不是为影视领域单独训练或硬编码的专用 Agent。**

### 1.2 运行链路

```text
用户
  │
  ▼
TDCanvas WebView / 浏览器右侧 AgentPanel
  │  HTTP + SSE；同步当前画布快照、附件和工具结果
  ▼
本机 TDCanvas Agent（Node.js，默认 127.0.0.1:17371）
  ├─ /agent/codex/*：线程、turn、模型、审批、Skill
  ├─ /api/tools：MCP 工具转发
  └─ /canvas/*：画布状态、SSE tool_call、结果回传
  │
  ▼
Codex app-server --stdio
  │  tdcanvas MCP
  ▼
当前网页画布（React/Zustand 状态）
```

右侧面板发送 prompt 的请求体包含当前线程、会话 revision、权限模式、模型、reasoning effort、显式 Skill 选择器和附件；Agent 校验后启动或恢复 Codex 线程，再将 turn 事件通过 SSE 广播给网页（`web/src/components/agent/local-agent-panel.tsx:646-689`、`canvas-agent/src/server/http.ts:285-342`）。

画布并不是被 Codex 直接读写 DOM。浏览器主动把画布快照送到 `/canvas/state`，Agent 将工具调用作为 SSE `tool_call` 发回网页，网页执行 `applyOps` 后再向 `/canvas/result` 回传结果；仓库已有文档明确记录了这一边界（`docs/content/docs/development/local-codex-canvas.zh-CN.mdx:1-27,90-142`、`canvas-agent/src/canvas/session.ts:435-500`）。

### 1.3 Agent 的运行后端

- **Codex app-server**：当前主路径。TDCanvas 创建/恢复 Codex thread，配置 TDCanvas MCP，发送文本、附件和 Skill 输入，并把计划、推理摘要、工具调用、审批和 turn 状态转为面板事件（`canvas-agent/src/agent/codex-client.ts:104-120,184-201,781-820`）。
- **Claude CLI**：仓库仍有 `/agent/claude/turn`，使用 `claude -p --output-format stream-json`，只允许 `mcp__tdcanvas__*` 工具，并将输出转成 Agent 事件（`canvas-agent/src/server/http.ts:401-414`、`canvas-agent/src/agent/claude.ts:7-38`）。它更像兼容/备用入口，不是当前 Skill 管理链路的证据。
- **TDCanvas MCP**：`canvas-agent/src/server/mcp.ts` 将 `canvas/schemas.ts` 中的工具注册到 stdio MCP，再转发到本地 HTTP `/api/tools`。Codex 在面板内启动时也会通过 `codexConfig()` 注入同一个 MCP（`canvas-agent/src/server/mcp.ts:9-31`、`canvas-agent/src/agent/codex-client.ts:781-788`）。

## 2. 面向用户的功能范围

### 2.1 对话与会话

当前 UI 和 HTTP 层支持：

- 新建、切换、恢复和归档 Codex 线程；
- 读取历史消息，展示 assistant 文本、推理摘要、计划、工具调用、MCP 启动状态、错误和事件日志；
- 选择模型和 reasoning effort；
- 停止当前 turn；
- 上传图片附件，并在消息历史中保留附件引用；
- 断线后使用 SSE 重放与历史确认机制，按 `threadId`、`turnId`、`itemId` 归属消息。

这些能力对应面板的 `chat`、`history`、`log`、`setup` 等 tab，以及 `useAgentStore` 中的线程、消息、运行状态和审批状态（`web/src/stores/use-agent-store.ts:6-36,42-77`、`web/src/components/agent/local-agent-panel.tsx:1291-1398`）。

### 2.2 画布读取和写入

MCP 工具清单覆盖：

| 类型 | 代表工具 | 实际作用 |
| --- | --- | --- |
| 导航/站点 | `site_navigate`、`canvas_list_projects` | 跳转站点页面、查找/选择画布项目 |
| 读取 | `canvas_get_state`、`canvas_get_selection`、`canvas_export_snapshot` | 读取当前画布压缩快照、选区或导出快照 |
| 节点 | `canvas_create_node`、`canvas_create_text_node`、`canvas_update_node`、`canvas_update_node_text`、`canvas_move_nodes`、`canvas_resize_node`、`canvas_delete_nodes` | 创建、修改、移动、缩放和删除节点 |
| 连接与视图 | `canvas_connect_nodes`、`canvas_select_nodes`、`canvas_set_viewport` | 连接节点、选择节点、调整视口 |
| 生成流程 | `canvas_create_image_prompt_flow`、`canvas_create_generation_flow`、`canvas_generate_text/image/video/audio`、`canvas_run_generation` | 创建提示词/配置/引用连线，并触发 TDCanvas 已接入的通用文本、图像、视频或音频生成回调；不等价于提交 ComfyUI 工作流 |
| 资源与任务 | `generation_get_status`、`prompts_search`、`assets_list`、`assets_add` | 查询生成状态、搜索提示词和操作素材资源 |

清单来源为 `canvas-agent/src/canvas/schemas.ts:9-41`；高级工具在 `canvas-agent/src/canvas/operations.ts:9-66` 被归一为网页可执行的 `add_node`、`update_node`、`delete_node`、`connect_nodes`、`set_viewport`、`select_nodes` 和 `run_generation` 操作。

### 2.3 权限与安全交互

面板有两层不同的确认机制：

1. **网页画布工具确认**：`confirmTools` 开启后，画布写工具先显示待确认卡片，用户批准后才调用 `applyOps`；前端还保留最近一次操作用于撤销（`web/src/components/agent/local-agent-panel.tsx:782-855`、`docs/content/docs/development/local-codex-canvas.zh-CN.mdx:126-142`）。
2. **Codex app-server 审批**：权限模式为 `request`、`automatic`、`full`。前两者保持 `on-request` 的 Codex 审批策略；`full` 使用 `never` 审批和 `danger-full-access`，UI 在切换到该模式时显式弹出高风险确认（`web/src/stores/use-agent-store.ts:11-12`、`web/src/components/agent/local-agent-panel.tsx:880-894`、`canvas-agent/src/agent/codex-client.ts:786-805`）。

这两个开关不要混同：前者决定网页是否逐次确认画布写入，后者决定 Codex 运行时对命令、文件和权限请求的审批策略。

## 3. Agent 如何使用 Skill

### 3.1 选择和传入

Skill 选择器只展示 `enabled === true` 的原生 Codex Skill，并允许搜索；选中后，如果 Skill 有 `interface.defaultPrompt`，会自动填充/维护 composer prompt（`web/src/components/agent/agent-skill-picker.tsx:11-83`、`web/src/stores/use-agent-skill-store.ts:39-68`）。

发送 turn 时，前端把 `{ name, path }` 作为 `skill` 字段传给 `/agent/codex/turn`。HTTP 层调用 `resolveCodexSkill(..., true)`，重新刷新 Codex 原生 Skill 清单并检查：名称和路径都匹配、路径为绝对路径、Skill 仍处于启用状态；失败时返回找不到或“已停用”的错误（`web/src/components/agent/local-agent-panel.tsx:672-687`、`canvas-agent/src/server/http.ts:303-305`、`canvas-agent/src/agent/codex.ts:123-138`）。

随后 `codexInput()` 同时完成两件事：

- 若 prompt 中还没有 `$skill-name`，就在文本前自动加入 `$skill-name`；
- 另附一个 `{ type: "skill", name, path }` 的 Codex 原生 turn input。

这说明 TDCanvas 是“选择、校验和转发”层，而不是 `SKILL.md` 的解释器；Skill 的具体语义由 Codex app-server 依据原生协议加载（`canvas-agent/src/agent/codex-protocol.ts:21-49,111-120`、`canvas-agent/src/agent/codex-client.ts:812-824`）。

### 3.2 Skill 对影视工作流能做什么

从当前能力看，Skill 最适合承载“稳定的工作方法”和“输出约束”，例如：

- 把一段故事梗概拆成角色、场景、镜头和生成提示词，并按固定中文格式写入多个文本节点；
- 为图像/视频生成节点统一规定镜头比例、时长、镜头运动、角色一致性参考和命名规则；
- 按一个可复用模板搭建“剧本 → 分镜文本 → 参考图 → 视频生成 → 结果整理”的画布流程；
- 规定何时读取当前画布、何时读取选区、何时先创建附件节点，再把节点加入生成流程；
- 约束一个包含 ComfyUI 宏节点的人工/插件工作流，例如规定输入检查、节点连接和结果验收步骤；但当前没有专用 ComfyUI MCP 工具，Skill 不能凭空增加直接提交或轮询 ComfyUI 工作流的能力。

这些属于 Skill instructions 对 Codex 行为的约束。Skill 本身不会新增 TDCanvas MCP 工具，也不会绕过网页确认；它仍然只能调用当前 Codex 会话已暴露的能力。

## 4. Skill 是否可以配置、创建和编辑

### 4.1 Agent 面板提供的管理操作

`AgentSkillsView` 的 Skills tab 提供：

- 列出当前 workspace 可发现的 Skill；
- 按 `user`、`repo`、`system`、`admin` scope 过滤并搜索；
- 启用/停用 Skill；
- 选择 Skill 进入聊天；
- 查看 Skill 详情；
- 手工创建 Skill；
- 编辑已管理 Skill；
- 删除已管理 Skill；
- 从当前对话或当前画布生成 Skill 草稿。

实现对应 `/agent/codex/skills`、`/agent/codex/skills/:name`、`/enabled`、`/delete`、`/draft` 这些 HTTP 路由（`web/src/components/agent/agent-skills-view.tsx:20-70,122-224`、`web/src/services/api/canvas-agent.ts:82-108`、`canvas-agent/src/server/http.ts:171-235`）。

其中“启用/停用”不是修改 `SKILL.md` 正文，而是调用 Codex app-server 的 `skills/config/write`；正文和界面元数据由 TDCanvas 的 `SkillStore` 写入，两个状态分别由 Codex 配置和本地 Skill 文件维护（`canvas-agent/src/agent/codex-client.ts:184-201`、`canvas-agent/src/skills/store.ts:93-148`）。

### 4.2 可配置的内容

编辑器允许配置：

- `name`：创建时填写，小写字母、数字和连字符；
- `description`：能力和触发场景，即何时使用；
- `instructions`：给 Codex 的可复用命令式步骤、约束和输出要求；
- `displayName`：UI 展示名称；
- `shortDescription`：25–64 个字符的短说明；
- `defaultPrompt`：选择 Skill 后用于 composer 的默认提示词；如填写，必须包含 `$skill-name`。

前端表单和后端 `SkillStore` 都会校验这些字段；后端额外限制 description 不能包含尖括号，正文最多 256 KiB，默认提示词最多 1024 字符（`web/src/components/agent/agent-skills-view.tsx:364-410`、`canvas-agent/src/skills/store.ts:8-15,264-307`）。

### 4.3 Skill 草稿不是自动落盘

“从对话生成”与“从画布生成”是两个独立入口：

1. 对话来源要求当前线程存在已完成的用户 turn；画布来源要求当前页面有画布，且会先把画布快照同步给 Agent（`web/src/components/agent/agent-skills-view.tsx:91-105`、`canvas-agent/src/server/http.ts:187-202`）。
2. Agent 为草稿创建临时线程：对话来源从当前线程 fork，画布来源新建临时线程；线程使用 `approvalPolicy: never`、只读 sandbox、无 MCP、`ephemeral: true` 和禁止工具/网络/文件修改的 developer instructions（`canvas-agent/src/agent/codex.ts:233-271`、`canvas-agent/src/agent/codex-client.ts:790-810`）。
3. Codex 必须按严格 JSON schema 返回草稿，TDCanvas 再用 Zod 校验，并拒绝包含本地路径、URL、token、密钥、临时标识或画布内部 ID 的内容（`canvas-agent/src/agent/codex.ts:18-42,246-249,393-414`）。
4. 草稿返回前端后，用户仍需在编辑器中检查并点击保存；保存才会调用 `POST /agent/codex/skills` 或 `POST /agent/codex/skills/:name` 写入文件（`web/src/components/agent/agent-skills-view.tsx:137-174`）。

这种设计适合把一次性的影视创作过程沉淀为可重复方法，但不能把自动提炼结果视为未经 review 的安全或正确的制作规范。

## 5. Skill 文件位置、作用域和安全边界

### 5.1 TDCanvas 管理的落点

本机站点级 Codex workspace 默认由 `ensureSiteWorkspace()` 建立在：

```text
~/.tdcanvas/codex-workspaces/site/
└── .agents/
    └── skills/
        └── <skill-name>/
            ├── SKILL.md
            └── agents/openai.yaml   # 可选
```

配置文件为 `~/.tdcanvas/tdcanvas-agent.json`；workspace 也可以被配置为其他绝对路径（`canvas-agent/src/config.ts:6-53`）。`SkillStore` 只管理该 workspace 下 `.agents/skills/<name>/SKILL.md` 及可选 `agents/openai.yaml`，不接受符号链接或目录联接（`canvas-agent/src/skills/store.ts:63-80,173-221`）。

### 5.2 原生 Codex scope

Codex `skills/list` 返回的元数据包含 `scope`、`path`、`enabled`、依赖和界面信息；当前协议声明的 scope 为 `user`、`repo`、`system`、`admin`（`canvas-agent/src/agent/codex-protocol.ts:21-44,111-117`）。这套 scope 是 Codex 的发现模型，不等价于 TDCanvas 的 workspace 目录：

| 维度 | 含义 |
| --- | --- |
| Codex scope | 原生 Skill 的来源/作用域分类，供发现列表和筛选使用 |
| `managed` | 路径是否落在 TDCanvas SkillStore 管理的标准 workspace `.agents/skills` 入口 |
| `enabled` | Codex 当前是否允许该 Skill 被选择和执行 |

因此，面板可能显示一个可发现但 `managed: false` 的 Skill；它可以被筛选和使用，但不能由 TDCanvas 的编辑/删除按钮改写。

### 5.3 并存的仓库插件 Skill

仓库还提供：

- `plugins/tdcanvas/skills/canvas/SKILL.md`：指导 Codex 读取画布、选择工具、创建文本/图片/视频/音频生成流程和批量操作；
- `plugins/tdcanvas/skills/open-canvas/SKILL.md`：指导 Codex 启动 Web、构建/启动本地 Agent、读取 URL/token 并打开画布。

它们属于随 `tdcanvas` Codex 插件一起提供的仓库资产（`plugins/tdcanvas/.codex-plugin/plugin.json`、`plugins/tdcanvas/README.md:1-27`），不是自动复制到 `~/.tdcanvas/codex-workspaces/site/.agents/skills` 的用户 Skill。只有在外部 Codex 中安装该插件后，外部 Codex 才可能按自己的插件机制发现这些 Skill；右侧 AgentPanel 的本地 Skill 管理仍以当前 workspace 的 `skills/list` 结果为准。另一个 `.agents/skills/project-docs/` 是本研究 fork 的开发调研技能，也不是面向最终用户的影视 Skill。

### 5.4 写入和并发保护

SkillStore 使用原子文件写入、串行写队列和 SHA-256 revision。编辑/删除时必须提交当前 revision，若 Skill 已被另一个窗口或外部编辑器改过，服务返回冲突而不是覆盖新内容（`canvas-agent/src/skills/store.ts:121-170,309-317,378-391`）。这使面板编辑相对适合多窗口使用，但不会替代 Git 或用户自己的备份。

## 6. 对 AI 影视制作的适用性评估

### 已具备的基础

- **素材与上下文**：可以把上传图片创建为真实图片节点，再作为生成参考；当前 Agent instructions 明确要求先调用 `canvas_create_attachment_nodes`，不要创建空占位图（`canvas-agent/agent-instructions.md:1-18`、`canvas-agent/src/canvas/session.ts:457-482`）。
- **多媒体流程**：画布节点类型包含 `image`、`text`、`config`、`video`、`audio`，工具可以创建并连接文本、图片、视频和音频生成流程（`canvas-agent/src/canvas/schemas.ts:6-7,17-37`、`canvas-agent/src/canvas/operations.ts:84-145`）。
- **流程化创作**：`canvas_create_generation_flow` 会创建提示词节点、配置节点、引用连线，并可选择自动运行，适合将提示词、参考图和生成参数显式化。
- **ComfyUI 组合空间**：桌面端的 `comfyui-local:workflow` 宏节点为画布提供另一类可编排的工作流节点；但当前 `canvas-agent` 的 MCP schema 没有 ComfyUI 专用工具，且通用 `canvas_create_node` 的节点类型只接受 `image`、`text`、`config`、`video`、`audio`，所以不能据此断言 Agent 能创建或直接提交 ComfyUI 工作流。`canvas_run_generation` 走的是通用网页生成回调；已有 ComfyUI 节点的实际运行仍由 `comfyui-local` 插件的节点面板/执行管理器处理。关于 ComfyUI 的具体接入边界，见 [`comfyui-integration-research.md`](comfyui-integration-research.md)。
- **方法论沉淀**：Skill 可以把一次被验证的“剧本/分镜/提示词/生成/检查”过程转成可重复的 instructions 和默认调用提示词。

### 当前缺口

- 没有发现专门的角色库、镜头库、分镜时间轴、剪辑时间线或影视知识图谱 Agent；这些需要用现有画布节点、外部服务或自定义 Skill 组合。
- 没有发现把 `canvas-agent` 直接连接到 ComfyUI `/prompt`、`/history` 等 API 的 Agent 路径；这部分属于桌面 `comfyui-local` 插件的独立执行链。
- Skill 只能约束 Codex 的工作方法，不能凭空获得新的模型、视频引擎、ComfyUI 自定义节点或媒体存储能力。
- 生成质量、镜头连续性、角色一致性、版权/安全审核等领域规则没有在当前 Agent 核心中形成独立的强制校验器；若要稳定化，应在 Skill、节点数据结构或服务层增加可验证的约束，而不是只依赖自然语言说明。
- Skill 草稿提炼已做敏感信息过滤，但没有运行时证明其影视方法论正确；用户仍应 review `description`、`instructions` 和 `defaultPrompt`。

## 7. 边界、风险和待确认问题

### 源码可确认的边界

- Agent 默认只监听 `127.0.0.1`；除 `/health`、`/config` 等发现端点外，网页通过 token 访问受保护 API，再通过 SSE 连接并同步画布快照（`canvas-agent/src/config.ts:6-23`、`canvas-agent/src/server/http.ts:122-143,508-529`、`canvas-agent/src/canvas/session.ts:264-295`）。
- 画布写入的最终执行者是网页前端，不是 Node Agent；因此“Codex 已提出操作”不等于“画布已经成功写入”，还要等待网页执行结果。
- Codex 的 `full` 权限模式会关闭审批并允许危险全访问，适合本地受控环境，不应当被当成默认安全模式。
- Skill 的 `defaultPrompt` 只是选择 Skill 时的 prompt 辅助；真正执行仍必须通过有效的 `$skill-name` 和原生 Skill input。

### 尚未由本次调研验证的事项

- 未在本机启动 Codex app-server、Claude CLI 或浏览器进行端到端 Agent 对话；本文没有声称当前账号、模型、MCP 登录状态或本机权限一定可用。
- 未实测一个自定义影视 Skill 从创建、启用、选择、执行到删除的完整闭环；结论来自源码、API 路由和已有单元测试。
- 未验证不同 Codex 版本对 `skills/list`、`skills/config/write` 和 `type: "skill"` 输入的兼容性；协议实现依赖当前仓库与本机 Codex app-server 版本匹配。
- 未验证 Claude CLI 是否会自行读取某种外部 Skill 目录；当前仓库的 `/agent/claude/turn` 代码没有提供 TDCanvas Skill selector，因此不把这种可能性写成 TDCanvas 已支持功能。
- 未验证跨设备迁移 workspace `.agents/skills` 后，路径、启用状态和依赖是否仍然可用；如果 Skill instructions 引用了外部工具或本地资源，迁移时仍需重新检查。

## 8. 推广建议

如果后续要把结论提炼到发布文档，建议只做小范围增量：

1. 在 `docs/content/docs/development/local-codex-canvas.zh-CN.mdx` 增加一段“右侧用户 Agent 与 Skill”说明，保留本文链接作为实现细节和 review 依据。
2. 在 `docs/content/docs/overview/features.mdx` 只描述已经准备好并经过用户验证的“画布 Agent 能力”，不要把“影视专用 Agent”或“自动生成 Skill”写成无条件承诺。
3. 如果产品决定支持影视工作流模板，优先在 `plugins/tdcanvas/skills/` 或站点 workspace 的 `.agents/skills/` 新增独立 Skill，并在 `docs/drafts/` 记录其输入、输出、失败边界和可复现步骤；不要把领域规则散落到多个共享页面。
4. 如果需要 Claude 路径也支持 Skill，应先明确采用 Claude 自身的 Skill/提示词机制，或在 TDCanvas 的 `/agent/claude/turn` 增加显式且可审计的 Skill 传递协议；当前不应把 Codex 路径的结论直接套到 Claude。

## 证据入口

- `web/src/layouts/user-layout.tsx:6-14`、`web/src/components/agent/agent-panel.tsx:12-60` — 用户布局挂载右侧 Agent 面板。
- `web/src/components/agent/local-agent-panel.tsx:646-689,782-894,1291-1398` — 发送 prompt、附件、Skill、工具确认、权限模式和 UI tab。
- `web/src/components/agent/agent-skill-picker.tsx:11-83`、`web/src/components/agent/agent-skills-view.tsx:20-224,364-410` — Skill 选择与管理 UI。
- `web/src/services/api/canvas-agent.ts:82-108`、`canvas-agent/src/server/http.ts:171-235,285-342` — Skill 列表、创建、编辑、启停、删除、草稿和 turn HTTP API。
- `canvas-agent/src/config.ts:6-53`、`canvas-agent/src/skills/store.ts:63-221,264-317,378-415` — workspace、文件落点、字段限制、revision 和安全检查。
- `canvas-agent/src/agent/codex.ts:123-146,176-222,233-271`、`canvas-agent/src/agent/codex-client.ts:184-201,781-824` — Skill 发现、启用校验、Codex turn input、MCP 配置和草稿线程。
- `canvas-agent/src/canvas/schemas.ts:9-41`、`canvas-agent/src/canvas/operations.ts:9-66`、`canvas-agent/src/canvas/session.ts:435-500` — MCP 工具、画布操作转换和网页执行回传。
- `canvas-agent/src/canvas/types.ts:1-6`、`web/src/integrations/comfyui-local/canvas-node.tsx:20-98`、`web/src/integrations/comfyui-local/execution.ts:19-129` — Agent 的内置节点类型与 ComfyUI 宏节点/执行链边界。
- `web/src-tauri/tauri.conf.json`、`web/package.json`、`web/scripts/package-desktop.mjs`、`Dockerfile`、`render.yaml`、`vercel.json` — 桌面打包入口和 Web 静态部署入口；未发现 `canvas-agent` sidecar 或云端 Agent 启动项。
- `canvas-agent/agent-instructions.md:1-18`、`plugins/tdcanvas/skills/canvas/SKILL.md`、`plugins/tdcanvas/skills/open-canvas/SKILL.md` — Agent 行为约束和仓库随插件提供的 Skill 示例。
- `docs/content/docs/development/local-codex-canvas.zh-CN.mdx:1-142`、`canvas-agent/README.md:1-75` — 现有架构说明、安装和 MCP 使用文档。

## 验证记录

- 已检查 `AGENTS.md`、`.agents/skills/project-docs/SKILL.md` 及其模板，并按 research fork 约定把结果留在 `docs/drafts/`。
- 已使用 `rg`、`sed` 和 `nl` 静态阅读上述实现、配置、桌面打包脚本、仓库 Skill、ComfyUI 集成、现有文档和相关测试入口；据此修正了“Agent 可直接运行 ComfyUI 工作流”的过宽表述。未修改应用代码或发布文档。
- 未运行构建、测试、Codex app-server、Claude CLI、浏览器端到端流程或真实媒体生成；因此本文是源码级调研，不是运行验收报告。
- `docs/content/docs/progress/todo.mdx` 和 `docs/content/docs/progress/pending-test.mdx` 未因本次 research-only 调研修改；本次没有改变产品功能状态。
