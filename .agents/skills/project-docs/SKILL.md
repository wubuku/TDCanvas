---
name: project-docs
description: Research, write, and maintain TDCanvas project documentation with an upstream-conflict-minimizing workflow. Use for project research, documentation drafts, doc audits, or requested doc updates (调研、编写文档、完善文档、文档检查).
---

# TDCanvas Project Documentation

Use a research-first workflow to help people and coding agents understand TDCanvas without making upstream merges harder.

## Repository Context

- This local repository is a research fork of the upstream TDCanvas project. The `origin` remote is used for the research repository and `upstream` is the source repository in the current setup; always confirm remote URLs with `git remote -v` before remote operations.
- The root `AGENTS.md` is authoritative. The project is Chinese-first; follow its documentation, build, testing, and release rules over generic templates here.
- Main documentation homes:
  - `docs/index.md` and `docs/index.zh-CN.md`: documentation navigation.
  - `docs/content/docs/`: published product and development documentation.
  - `docs/product/`: product requirements and design context.
  - `docs/drafts/`: research findings, evidence, build records, and review drafts.
  - `.agents/skills/`: repository-local agent skills.
- The docs site has its own application/configuration. Inspect the current files and scripts before documenting or running a docs build; never invent build commands.

## Upstream Conflict Policy

1. Put all investigation results, evidence, comparisons, and uncertain conclusions in `docs/drafts/`. A research task does not authorize edits to application code or upstream-facing documentation.
2. Treat `docs/drafts/` as the preferred low-conflict area, not a guarantee that conflicts are impossible. Use a specific filename, inspect existing local content, and preserve user-authored drafts. If upstream refs are already available locally, check whether the intended path is already occupied there.
3. Do not edit existing upstream-facing files just to link, summarize, or promote a draft. In particular, avoid broad edits to root `README.md`, `AGENTS.md`, `CHANGELOG.md`, `docs/index*.md`, progress lists, and long published pages.
4. If the user asks to update a document outside `docs/drafts/`, prefer a narrow additive paragraph, bullet, or link. Before editing, explain why that location is necessary and why the diff should reduce rather than increase merge-conflict risk. Do not reformat unrelated text.
5. Never resolve a merge conflict by dropping upstream content or local research without reviewing both sides. Keep source evidence and research conclusions separate from implementation changes.
6. Do not merge, fetch, push, commit, or change remotes as part of a documentation task unless the user explicitly asks.

## Workflow

1. Start with `git status --short --branch`, applicable `AGENTS.md` files, `docs/index.md`, and the contents of `docs/drafts/`. Preserve all existing user changes.
2. Read the relevant implementation, tests, configuration, and current docs before making claims. Use `rg` to locate source and test references.
3. Separate claims into implemented behavior, documented intent, inference, and behavior not verified. Prefer current code over stale status prose; mention discrepancies rather than silently rewriting shared docs.
4. Write research directly to a topic-specific file under `docs/drafts/`, using the template in `references/templates.md`. Cite repository-relative paths with line numbers when useful. Do not add dates unless the user asks or a time-bounded record requires them.
5. Promote a stable conclusion only when the user asks or confirms. Add the smallest useful update to the correct existing doc; do not copy the full draft into published docs.
6. Validate that cited files and local links exist, claims match inspected code, and the document does not overstate tests or runtime verification. Follow `AGENTS.md` about builds/tests; for documentation-only work, do not run application builds or tests unless explicitly requested.
7. Before finishing, check `docs/content/docs/progress/todo.mdx` and `pending-test.mdx` when the work touches feature status. For research-only work with no feature/todo changes, record that no update is needed instead of editing those shared files.

For a single research request, do not create a project-wide documentation plan. For a user-requested documentation-system initiative, keep its plan and research index under `docs/drafts/` and follow the checklist in `references/checklist.md`.

## Draft Quality

- Lead with the answer the reader needs, then show the implementation path and evidence.
- Keep the content concise, Chinese-first, and specific to this repository.
- Distinguish an integrated external tool from a dependency bundled with TDCanvas.
- Record doc/code inconsistencies and unverified platform combinations as review items; do not fix unrelated source/docs during research.
- Do not introduce a new root-level documentation file or a parallel docs structure from a generic template.
- Do not update published navigation solely to advertise a draft; `docs/drafts/README.md` can serve as the low-conflict draft index when useful.

## Applicable References

- `references/templates.md`: TDCanvas research draft and draft-index templates.
- `references/checklist.md`: conflict, evidence, and final review checklist.

These files are part of this skill package. Keep references self-contained and link to them with relative paths.
