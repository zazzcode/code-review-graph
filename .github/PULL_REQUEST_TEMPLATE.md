<!--
ENSURE YOUR PR TITLE HAS AT LEAST ONE CATEGORY LABEL.

Categories:
  CRG — Code Review Graph runtime, CLI, MCP tools, graph analysis, or review payloads
  DOC — Documentation, Zazz specs, architecture docs, standards, README, or usage guides
  SKILL — Agent skills, hooks, prompts, or workflow guidance under .agents/
  TEST — Test-only changes, fixtures, or validation harness updates
  CI — CI/CD, release, packaging, dependency, or infrastructure changes

Combine with + for cross-cutting PRs.

Example PR titles using these labels:
  [CRG] Add scoped review-context payloads
  [DOC] Document CRG architecture and data flow
  [SKILL] Add code-review-graph review skill
  [TEST] Add contract mismatch regression coverage
  [CI] Update publish workflow
  [CRG+DOC] Add context-savings command docs
  [CRG+SKILL] Route PR review through CRG packets
  [CRG+TEST] Add advisory review-signal coverage
-->

### Zazz Context

<!--
Link the governing Zazz specification, proposal, feature doc, architecture note,
execution record, or PR/body-only intent.

No Avaza, Jira, or external ticket is required for this fork. If no durable
Zazz document applies, briefly say why.
-->

- Spec:
- Proposal / feature / architecture:
- Execution notes:

### WHY

<!--
Summarize why this PR exists, not just what changed.

Example: "CRG review packets currently make Zazz section agents read too much
source context. This PR makes review evidence portable, budgeted, and easier to
route through standards/spec review."
-->

### WHAT

<!-- Keep this section only if the WHAT is not self-evident from WHY. -->

### Instructions for Reviewers

<!--
Tell reviewers what to inspect or validate.

For CRG changes, include the exact command, base branch, scope, token budget,
and expected output shape. For skill/docs changes, name the workflow or review
scenario the reviewer should walk through.
-->

### How It Was Tested

<!--
Paste the exact commands run and summarize their results. Include failed or
partially passing checks when they are relevant to review.
-->

```bash
uv run pytest tests/ --tb=short -q
uv run ruff check code_review_graph/
uv run mypy code_review_graph/ --ignore-missing-imports --no-strict-optional
```

### Checklist

- [ ] Tests added or updated for new behavior
- [ ] Relevant automated tests pass
- [ ] Relevant lint/type/doc checks pass, or known unrelated failures are called out
- [ ] Docs updated where behavior or workflow changed
- [ ] Zazz standards/specs reviewed for the touched paths
- [ ] Human review remains the merge authority

### Demo

<!--
Optional. Include screenshots, terminal output excerpts, generated graph views,
or before/after payload snippets when they help the reviewer understand the
change.
-->
