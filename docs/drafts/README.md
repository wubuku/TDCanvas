# 调研与构建草稿

> 本目录用于保存 TDCanvas research fork 的调研结论、证据和本机复现记录，作为低冲突的 review 区域；草稿不会自动并入发布文档。

## 草稿索引

- [macOS 桌面客户端构建与安装记录](desktop-build-macos.md)：本机桌面应用构建、打包、安装过程及复现条件。
- [TDCanvas 如何集成本地 ComfyUI](comfyui-integration-research.md)：本地 ComfyUI 的桌面接入、工作流导入、画布执行、输出和边界调研。
- [TDCanvas 技术架构说明](tdcanvas-technical-architecture.md)：Web、Tauri、画布插件、ComfyUI、Canvas Agent、存储和构建入口的跨模块架构调研。
- [TDCanvas 用户 Agent 与 Skill 机制调研](user-agent-and-skills-research.md)：面向 AI 影视制作者的 Agent 入口、画布能力、Skill 执行链和 Skill 管理边界。

## 维护约定

- 调研、证据、未确认结论和复现记录优先放在本目录。
- 研究 fork 会持续合并 `upstream`；如需更新目录外文档，采用小范围增量修改，并先评估冲突风险。
- 草稿结论经 review 后，只有在用户明确需要时才提炼到 `docs/content/docs/`、`docs/product/` 或导航页；不要整篇复制。
