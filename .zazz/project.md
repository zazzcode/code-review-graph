# Code Review Graph Fork

## Purpose

This repository is the Zazz-maintained fork of `code-review-graph`, a local-first code
intelligence graph for token-efficient code review through CLI and MCP surfaces.

The fork exists to improve support for the Zazz `pr-review` methodology, especially:

- token-efficient review context generation
- trustworthy review-session token measurement
- multiple independent review axes
- better signals for net-new-file and large-change PRs
- reviewability and cohesion scoring that is explainable to human reviewers

## Upstream

- Fork remote: `https://github.com/zazzcode/code-review-graph.git`
- Upstream project: `https://github.com/tirth8205/code-review-graph`

## Methodology Docs

This repo uses `.zazz` as `DOCS_ROOT`.

Key locations:

- `.zazz/standards/index.yaml`
- `.zazz/proposals/`
- `.zazz/features/`
- `.zazz/architecture/`
- `.zazz/specifications/`
- `.zazz/execution/`

## Current Direction

The initial proposal is:

- `.zazz/proposals/code-review-graph-token-efficiency-and-turso-engine.md`

That proposal should be refined into feature documents and deliverable specifications
before implementation work begins.
