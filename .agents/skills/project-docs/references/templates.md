# TDCanvas Documentation Templates

Use the research template for new investigations. Do not use generic templates to replace the existing README, AGENTS guide, product docs, or published documentation structure.

## Research Draft

```markdown
---
title: {中文标题}
status: draft
scope: research
---

# {中文标题}

> **目的**：说明本稿要回答的问题或支持的决定。
> **状态**：调研草稿；未进入发布文档导航。
> **证据范围**：列出检查过的代码、配置、测试和现有文档。

## 结论

区分已确认事实、推断和待验证事项。

## 当前实现

按实际代码描述数据流、依赖、平台边界和存储方式；不要把需求或旧文档当成实现证明。

## 边界与待确认问题

记录未支持场景、文档与代码差异、缺少的运行时验证。

## 证据入口

- `relative/path/to/file.ts:10` — 该入口说明什么。

## 验证记录

写出实际执行过的检查及结果；明确列出没有运行的测试或构建。

## 推广建议

如果需要进入正式文档，指出目标位置、最小增量和 upstream 冲突风险。未获请求或确认时，保留在本草稿目录。
```

## Draft Index

Create `docs/drafts/README.md` only when an index materially improves discovery or the user asks for one.

```markdown
# 调研与构建草稿

> 本目录保存研究 fork 的调研证据、复现记录和 review 草稿；不会自动进入发布文档。

## 草稿索引

- [草稿标题](topic-research.md)：一句话说明内容。

## 维护约定

- 研究结果和未确认结论留在本目录。
- 目录外文档只按用户要求进行最小、增量的更新。
```

## Path and Date Rules

- Use descriptive filenames such as `comfyui-integration-research.md` or `desktop-build-macos.md`; avoid generic names such as `NOTES.md`.
- Use repository-relative paths in the body so another checkout can follow the evidence.
- Do not add an `updated` date field by default. Follow root `AGENTS.md` and add a date only when explicitly requested or necessary to distinguish time-sensitive evidence.
- Do not add a second English copy unless the user or audience requires it.
