# TDCanvas Documentation Review Checklist

## Scope and Conflict Risk

- [ ] All research findings and evidence are in `docs/drafts/`.
- [ ] Existing local drafts and user changes were preserved.
- [ ] Research-only work did not change app code or upstream-facing docs.
- [ ] Any requested edit outside `docs/drafts/` is narrow, additive, and has a clear low-conflict rationale.
- [ ] Root `README.md`, `AGENTS.md`, `CHANGELOG.md`, `docs/index*.md`, progress docs, and published MDX were not rewritten or reformatted unnecessarily.

## Evidence and Accuracy

- [ ] Conclusions distinguish implementation, documented intent, inference, and unverified behavior.
- [ ] Claims cite current repository files and line references where useful.
- [ ] External tools are described accurately as bundled, optional, locally installed, or remotely hosted.
- [ ] Known source/document inconsistencies are recorded rather than silently propagated.
- [ ] No tests, builds, or platform flows are claimed unless they actually ran.

## Readability and Validation

- [ ] The answer appears near the beginning and the draft has a clear purpose.
- [ ] The document is Chinese-first and avoids duplicating entire existing docs.
- [ ] Cited local paths and internal links resolve.
- [ ] No unnecessary date or date field was added.
- [ ] `todo.mdx` and `pending-test.mdx` were checked when feature status is in scope; no shared progress doc was edited for research-only changes.
- [ ] Drafts remain outside published navigation unless promotion was requested.
