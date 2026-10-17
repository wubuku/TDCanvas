---
title: TDCanvas 如何集成本地 ComfyUI
status: draft
scope: research
---

# TDCanvas 如何集成本地 ComfyUI

> **目的**：厘清 TDCanvas 桌面端与 ComfyUI 的关系、用户可用功能和运行边界，避免把外部运行时误认为 TDCanvas 内置组件。
> **状态**：基于当前仓库代码与文档的调研草稿，尚未进行实际 ComfyUI 环境的端到端验收。
> **证据范围**：README、ComfyUI Local 模块、Tauri 2 桌面工程、画布集成代码、PRD 与待测试文档。

## 结论

- **已确认：TDCanvas 桌面端提供可选的本地 ComfyUI 集成。** 用户可以把自己维护的 ComfyUI API 工作流包装成 TDCanvas 画布宏节点，配置参数、连接画布素材、运行工作流，并把输出回填为画布结果节点。
- **已确认：ComfyUI 不是 TDCanvas Desktop 内置的生成引擎。** Tauri 应用注册了 TDCanvas 自己的原生 ComfyUI 插件，但插件要求用户选择本机 ComfyUI 安装目录，或连接已经运行的本机实例。项目 README 推荐另行下载的整合包，并明确说该包不含模型。
- **已确认：该集成不代表 TDCanvas 的所有 AI 生成功能都由 ComfyUI 驱动。** 它是独立的“ComfyUI 本地”能力，使用本机 ComfyUI 的模型、工作流和自定义节点；它与 README 中介绍的 Aitudou 生成能力是不同入口。
- **已确认：运行入口依赖 Tauri 桌面环境。** `/comfyui-local` 路由在 Web 工程中存在，但非桌面运行时显示“桌面端专用”说明，原生调用也会拒绝浏览器运行时。

## 架构与数据流

```text
TDCanvas「ComfyUI 本地」页面
    ├─ 选择 / 启动本机 ComfyUI，或连接已运行的本机实例
    ├─ 导入 API Format JSON，并读取运行时 /object_info
    └─ 将工作流定义保存到 WebView 本地工作流库
              ↓
TDCanvas 画布 ComfyUI 宏节点
    ├─ 输入：参数面板 + 可选的具名画布端口
    ├─ 执行：Tauri invoke → Rust 原生插件 → 本机 ComfyUI HTTP API
    └─ 输出：图片 / 视频 / 音频 / 文本 / 文件等原生结果节点
```

桌面工程通过 `web/src-tauri/Cargo.toml:41` 引入 `modules/comfyui-local/tauri-plugin`，并在 `web/src-tauri/src/lib.rs:171-172` 注册插件；主窗口 capability 也包含 `tdcanvas-comfyui-local:default`（`web/src-tauri/capabilities/main-window.json:22`）。页面在 `web/src/router.tsx:30` 注册，在 `web/src/pages/comfyui-local/index.tsx:20-27,247-259` 检测 Tauri 环境并限制非桌面操作。

## 用户如何使用

1. **准备 ComfyUI 环境**：先单独安装 ComfyUI，按工作流准备模型和自定义节点。在 TDCanvas「ComfyUI 本地」页面选择环境目录；检测不到可用 Python 时可另外指定解释器。也可以先在本机启动 ComfyUI，再在页面中选择端口连接。入口说明见根目录 `README.md:76-89` 和 `modules/comfyui-local/README.md:5-11`。
2. **启动或连接**：由 TDCanvas 启动时，原生插件用已授权的 `main.py` 与 Python 启动 ComfyUI，分配随机端口并绑定 `127.0.0.1`，同时传入 `--disable-auto-launch`；连接外部实例时，TDCanvas 检查指定的本机端口是否可用。连接模式下的“停止”实际是断开连接，不会结束外部进程（`modules/comfyui-local/tauri-plugin/src/lib.rs:398-440,517-625,628-675,1592-1616`）。
3. **导入工作流**：选择 JSON 文件。当前解析器只接受 ComfyUI **API Format JSON**，遇到普通 UI workflow JSON 会要求重新以 “Save (API Format)” 导出。导入时读取正在运行实例的 `/object_info`，检查节点类型、依赖和可暴露的输入输出；向导允许选择输入输出并命名，再保存到工作流库（`modules/comfyui-local/core/src/workflow-inspector.ts:34-105`、`web/src/integrations/comfyui-local/workflow-import-wizard.tsx:152-204`）。缺失依赖的工作流可以作为不可运行草稿保存，依赖恢复后再处理。
4. **加入画布并运行**：工作流可从工作流库加入画布，也可通过画布节点入口选择。它表现为一个带动态类型端口的“ComfyUI 工作流”宏节点；参数面板可编辑输入值或禁用输入。运行时，画布连接值优先于节点手动值，手动值优先于工作流默认值；图片、视频、音频等连接输入先上传至 ComfyUI，再提交工作流。核心执行顺序是 `materialize → /prompt → /history/{prompt_id}`（`web/src/integrations/comfyui-local/canvas-node.tsx:20-73,76-98`、`web/src/integrations/comfyui-local/execution.ts:19-68,94-129`、`modules/comfyui-local/tauri-plugin/src/lib.rs:774-955`）。
5. **查看与停止结果**：输出会映射为画布中的图片、视频、音频、文本等原生节点；多项输出可展开为多个结果节点。停止时先检查 `/queue`，等待中的任务按 prompt ID 删除；只有目标任务正在运行时才调用 `/interrupt`（`web/src/integrations/comfyui-local/result-nodes.ts:1-107`、`modules/comfyui-local/tauri-plugin/src/lib.rs:957-996`）。

## 桌面端与浏览器端的边界

- Tauri 原生插件负责系统目录选择、ComfyUI 进程生命周期、本机 HTTP 请求、日志、上传、任务队列和结果落盘。前端通过 `plugin:tdcanvas-comfyui-local|...` 命令调用原生层（`modules/comfyui-local/frontend/src/native-client.ts:1-81`）。
- 管理启动模式只能访问本机 loopback；当前原生代码固定访问 `127.0.0.1`。连接模式也只接受本机端口，而非任意 LAN 或远程 ComfyUI 主机。因此“本地模式”不是远程 ComfyUI 客户端。
- 用户选择的环境配置保存在 Tauri `AppLocalData/comfyui-local/environments.json`，进程所有权信息也在同一目录维护（`modules/comfyui-local/tauri-plugin/src/lib.rs:398-408`）。
- 工作流定义由 `localforage` 存进 WebView 本地工作流库 `tdcanvas-comfyui-local/workflow_library`；画布节点元数据保存工作流/环境 ID、已暴露的输入输出及参数值。节点引用工作流库中的 API JSON，而不是在节点快照中重复保存整份 API 工作流（`web/src/integrations/comfyui-local/workflow-library.ts:1-42`、`modules/comfyui-local/core/src/workflow-definition.ts:32-57`、`web/src/integrations/comfyui-local/canvas-node.tsx:87-98`）。
### 项目迁移/复现时的重要限制（代码推断）

- 画布项目 ZIP 导出包含 `CanvasProject` 对象和项目引用的媒体文件；检查当前导出结构没有发现它同时导出 `localforage` 中的 ComfyUI 工作流库（`web/src/lib/canvas/canvas-export.ts:11-31`、`web/src/types/canvas-export.ts:1-13`）。因此，复制或导出画布项目到另一台机器时，ComfyUI 宏节点虽然保存了工作流 ID、输入输出定义和参数值，却可能无法从那台机器的工作流库中解析完整 API 工作流。执行代码会因工作流定义不存在而失败（`web/src/integrations/comfyui-local/execution.ts:35-41`）。这是基于当前导出/加载路径的推断，尚未实际跨设备测试。
- 环境 ID 由 ComfyUI 根目录路径与 Python 解释器路径共同计算（`modules/comfyui-local/tauri-plugin/src/lib.rs:1886-1894`）。在另一台电脑或不同安装路径重新选择相同内容的环境，不应假定会得到相同 ID；画布节点所存的环境 ID 可能需要重新绑定/重建。当前源码页未发现该修复或迁移流程，本稿不判断其用户体验是否可绕过。
- 原生插件把多数文件结果放到 `AppLocalData/comfyui-local/results/<promptId>`。**视频是源码中的例外**：保存位置为所选 ComfyUI 环境的 `output/<画布标题>`（`modules/comfyui-local/tauri-plugin/src/lib.rs:1195-1312`）。

## 功能范围与未确认项

### 当前代码可确认

- 工作流包装为可复用宏节点；输入可映射文本、数字、枚举、布尔、JSON、图片、视频和音频控件，选定输入/输出可暴露为具名画布端口。
- 参数、画布连线与工作流默认值可组合；若干媒体入口在无连线或被禁用时，核心逻辑支持对安全可旁路的 ComfyUI 节点分支做旁路。
- 当前执行通过 `/upload/image`、`/prompt`、`/history/{prompt_id}`、`/queue`、`/interrupt` 等本机 API 完成；不是把 ComfyUI Web 页面嵌进 TDCanvas。
- 当前代码轮询 `/history/{prompt_id}` 取得本次执行结果，并把执行状态写到宏节点/结果节点。这与“可浏览的持久任务历史”不是一回事；PRD 和模块说明仍将更完整的运行历史能力列为待增强项（`docs/product/comfyui-local-mode-prd.md:3`、`modules/comfyui-local/README.md:48-50`）。

### 需要 review 的文档差异

- `modules/comfyui-local/README.md:45` 的概述容易让人理解为所有 ComfyUI 输出文件都缓存在 TDCanvas `AppLocalData`；但当前 Rust 代码对视频输出使用 ComfyUI 环境的 `output/<画布标题>`。建议后续确认是否更新模块说明。本次按“调研留在 drafts、降低 upstream 冲突”的约定，不改该共享模块文档。
- `docs/content/docs/progress/pending-test.mdx:31` 仍把 Windows portable、源码环境和 macOS 环境的 GUI/端到端验证列为待测。因此本稿只描述代码路径与仓库文档记录，不声称这些组合已在本机实际跑通。
- PRD 中“运行历史”的定义可能包括执行节点状态、结果版本或独立历史浏览能力；当前代码和文档用词不完全一致，需在后续 review 时明确范围。

## 源码与文档入口

- `README.md:76-89`：用户快速了解推荐整合包和导入步骤。
- `modules/comfyui-local/README.md:1-58`：模块边界、功能列表、自测命令。
- `docs/product/comfyui-local-mode-prd.md:1-20`：产品目标与场景；其中状态描述应与当前代码一并判断。
- `web/src/pages/comfyui-local/index.tsx:20-80,213-231`：桌面环境页、工作流入库及加入画布流程。
- `modules/comfyui-local/core/src/workflow-inspector.ts:34-105` 与 `workflow-definition.ts:32-57`：API 工作流解析、依赖检查、可暴露端口和定义结构。
- `web/src/integrations/comfyui-local/execution.ts:19-129`：宏节点运行、输入合并、媒体上传。
- `web/src/lib/canvas/canvas-export.ts:11-31` 与 `web/src/types/canvas-export.ts:1-13`：画布项目导出数据结构，用于判断 ComfyUI 项目迁移是否携带工作流库。
- `modules/comfyui-local/tauri-plugin/src/lib.rs:773-996,1073-1312`：原生 API 调用、队列/历史轮询、取消和结果缓存。
- `web/src-tauri/Cargo.toml:41`、`web/src-tauri/src/lib.rs:171-172`、`web/src-tauri/capabilities/main-window.json:22`：桌面工程接入点。

## 验证记录

- 已检查上述代码、README、PRD 和 `pending-test.mdx`，并对照了当前代码实现与文档声明。
- 未启动 ComfyUI，未运行 `npm --prefix modules/comfyui-local run verify`，也未进行 Windows/macOS 实机端到端验收；仓库已有待测项应继续视为待验证。
- 本次没有修改 `docs/content/docs/`、`docs/product/`、README、CHANGELOG 或应用代码；调研结论保留在 `docs/drafts/`，以避免增加与 `upstream` 同步时的冲突面。
