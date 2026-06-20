# Proposal — code-review-graph token-efficiency + Turso (Rust) engine

**Proposal slug:** `code-review-graph-token-efficiency-and-turso-engine`
**Type:** Joint (feature-scoped review-quality + deliverable-scoped engine swap)
**Status:** Draft — engine choice (Turso) settled this turn; token/axis decisions still open — not yet approved
**Owner:** Michael (fork maintainer)
**Upstream context:** Fork of `github.com/tirth8205/code-review-graph`; engine target `github.com/tursodatabase/turso`
**Docs root:** `docs/` (this proposal lives at `docs/proposals/`)
**Issue tracking:** `bd` (beads) per `AGENTS.md`

> This is an exploratory, non-authoritative proposal. It informs decisions before any SPEC/PLAN. Authoritative contracts (Feature Requirements Document / Deliverable SPEC) come later. It was produced with the `proposal-builder` skill and follows its 15-section structure.

---

## 1. Context and Problem Statement

`code-review-graph` is a local-first code intelligence graph: Tree-sitter parses the repo into a SQLite-backed graph of nodes/edges/flows/communities; `detect-changes` and the MCP tools hand a compact, risk-scored review context to an AI reviewer. It is the optional accelerator consumed by the `pr-review` skill (see `zazz-skills/.agents/skills/pr-review/`), which gates graph usage on changed-file count > 10 and feeds its blast-radius/flow/test-gap output to parallel Standards + Spec sub-agents.

Three problems surfaced from a real WITH/WITHOUT A/B experiment (documented in `code-review-graph-fork-and-ab-handoff.md`, sibling to this proposal):

1. **Token cost is real and not yet trustworthy in-product.** The handoff's first cost figure ($29.39) was wrong by ~2.4× because Claude Code transcripts duplicate each assistant message 3–4× and sub-agent usage lives in separate transcript files. The corrected, de-duplicated method lives only in an *external* scorer (`pr_review_token_cost.py`) in the consuming repo — not in `code-review-graph` itself. So the product advertises a "Token Savings ~42%" panel that measures only the change-analysis step in isolation, while the only trustworthy whole-session number requires a bespoke external script. A consumer cannot easily answer "is the graph worth it for *my* review?" from the product alone.
2. **The product's metrics assume incremental diffs against an existing graph — but most of this consumer's work is large swaths of new files.** `detect-changes` (`code_review_graph/changes.py`) maps a git diff's changed line-ranges onto existing nodes, scores risk from caller-count / flow-membership / test-coverage of *existing* functions, and lists "test gaps" as changed functions without `TESTED_BY` edges. When a PR is 90% net-new files, there is little prior graph to map onto, callers/flows are absent, and "no test edge yet" is the expected state of all new code — not a signal. The existing axes under-serve the dominant working mode.
3. **Storage is SQLite (correct), but the maintainer wants to evaluate a modern Rust engine.** The graph is persisted in a SQLite DB (`code_review_graph/graph.py`, WAL mode, schema v9 via `migrations.py`). JSON is only the output format. The token problem is **not** a storage problem — but the owner wants to explore replacing the engine with the **Turso Database** (the Rust rewrite of SQLite, formerly codenamed Limbo) and is willing to fix gaps and contribute upstream to `github.com/tursodatabase/turso`.

---

## 2. Scope and Non-Goals

**In scope — Phase 1 (token efficiency + measurement + new-file axes):**
- Build trustworthy token measurement **into the product** (de-duplicated, sub-agent-inclusive, per-session), so the WITH/WITHOUT A/B is reproducible from the product without an external scorer.
- Ship the token-efficiency output improvements from the prior fork analysis (repo-relative paths, compact `--for-review` mode, one-shot `review-context`, per-section scope, test-gap allowlist, scope-honest savings).
- Define and ship **new review axes** suited to large-swaths-of-new-files PRs (complexity, new-code mass, structural cohesion, sibling-template divergence), since the existing change-analysis metrics are often inapplicable there.
- Keep the `pr-review` skill's `[boulder]/[rock]/[pebble]/[sand]` sizing and two-axis (Standards + Spec) structure intact — extend, don't replace.

**In scope — Phase 2 (Turso Rust engine + upstream contributions):**
- Replace the SQLite engine with Turso Database (Rust, beta) as the **sole engine**, embedded in-process via `pyturso` ("integrated in the same application," not a separate DB server), behind the existing `GraphStore` interface and preserving the schema/migration model.
- Fix Turso gaps that block parity (notably FTS) and contribute those fixes upstream to `github.com/tursodatabase/turso`.
- Rely on the graph being a **rebuildable derived cache** (gitignored, regenerable from source) as the primary beta-risk mitigation, rather than a dual-engine fallback.

**Non-goals:**
- Not proposing a new review process or replacing the `pr-review` skill's two-axis design.
- Not adopting Turso Cloud / embedded-replica sync in Phase 2's default path (the graph is a local-first, gitignored cache; sync is a separate future conversation — see §6 and §9).
- Not re-architecting the parser (Tree-sitter) or the MCP surface.
- Not committing to a SPEC/PLAN; this proposal only decides *whether and how* to proceed.

---

## 3. Business Justification

- **Review cost is paid per PR, every PR, in dollars and latency.** The handoff's corrected A/B showed WITH-graph ~$11–14 vs WITHOUT ~$16 on one PR — a real but small delta, measured with a bespoke external script. If the product can make that measurement cheap and trustworthy *inside* it, adoption decisions stop requiring a research project per team.
- **Most of this consumer's work is net-new files.** A review accelerator whose headline metrics are designed for incremental diffs is leaving its highest-frequency use case under-served. New axes that grade *new-code mass and complexity* turn the graph from "advisory when applicable" into "advisory for the work we actually do."
- **Finding quality, not just cost.** The handoff's §4A showed the with-graph review *missed* a surgical sibling-template divergence that the plain review caught, because blast-radius reasoning biased it toward "matches the family." A first-class sibling-divergence signal is a capability the graph is uniquely positioned to provide and plain diff-reading is not — that is a quality argument, not a cost argument.
- **Engine modernization as a strategic option.** Moving to a memory-safe, async, Rust-based SQLite-compatible engine positions the fork for multi-process access, concurrent writes, and a contribution relationship with an active upstream — while keeping the door open to a SQLite fallback so beta risk is bounded.

---

## 4. Technical Justification

**Storage today (confirmed against source):** SQLite, WAL mode, schema v9 with versioned migrations (`code_review_graph/migrations.py`). Tables: `nodes`, `edges`, `metadata`, `communities`, `flows`, `flow_memberships`, `nodes_fts` (**FTS5 virtual table**), `community_summaries`, `flow_snapshots`, `risk_index`. Node `file_path` is stored **absolute** (`code_review_graph/parser.py:999`, `incremental.py:862`), which bloats tokens and busts cross-machine prompt-cache reuse.

**Why the token problem is an output problem, not a storage problem:** the ~110 KB blob the handoff observed is `json.dumps(result, indent=2)` at `code_review_graph/cli.py:1240` — the serialized `detect-changes` result, not the DB. The DB is already the right shape (indexed, FTS5, WAL). So Phase 1 targets the output/serialization/ingest path; Phase 2 targets the engine for modernization/strategic reasons, not because SQLite is failing.

**Turso (Rust) facts represented honestly (from `github.com/tursodatabase/turso` and Turso's own posts, June 2026):**
- Turso Database is an in-process SQL DB written in Rust, compatible with SQLite at the SQL dialect, file-format, and C-API level. It is the project formerly codenamed **Limbo**. Its own README states verbatim: *"⚠️ Warning: This software is in BETA. It may still contain bugs and unexpected behavior. Use caution with production data and ensure you have backups."* The latest stable on the crate index trail is ~v0.5.3 with v0.6.0 (May 2026) adding MVCC, Tantivy FTS, vector types, STRICT tables, CDC, extended ALTER, and multi-process WAL coordination (`.tshm` sidecar).
- It has a **Python binding `pyturso`** (`uv pip install pyturso`) with a `turso.connect("path.db")` API — the practical integration point for this Python codebase.
- It includes a built-in **MCP server mode** (`tursodb db.db --mcp`) — thematically aligned with this product, though not required.
- **Hard parity blocker for this codebase: FTS.** `code-review-graph` depends on SQLite **FTS5** (`nodes_fts`, `migrations.py:147-157`, `porter unicode61` tokenizer, `content='nodes'` external-content table). Turso's FTS is **Tantivy-based and experimental**, with different DDL syntax (`CREATE INDEX ... USING fts (...) WITH (tokenizer=...)`) and different query functions (`fts_match`, `fts_score`). This is not a drop-in swap; it is the first thing Phase 2 must absorb, likely as an upstream contribution.
- Turso's headline features — **sync / embedded replicas / edge replication / CDC / MVCC concurrent writes** — are largely *not* relevant to a local-first, gitignored, embedded read-cache used by a single review session. They become relevant only if we later pursue the ROADMAP's "Team sync (shared graph via git-tracked DB)" item (see §9).

---

## 5. Value Proposition and Expected Outcomes

**Phase 1 outcomes:**
- A consumer can run a trustworthy WITH/WITHOUT A/B using only the product (no external scorer), with de-duplicated, sub-agent-inclusive per-session token accounting.
- Each of the 3 section sub-agents in the `pr-review` fan-out receives a compact, repo-relative, budget-sized slice instead of a shared 110 KB blob → lower per-agent cache-write cost and re-enabled cross-agent prompt-cache reuse.
- New-code-heavy PRs get meaningful review signal (complexity, new-code mass, sibling-template divergence) instead of near-empty incremental-diff metrics.
- Machine-readable, scope-honest savings records the harness can log per review (so "~42%" is never misread as whole-session savings again).

**Phase 2 outcomes:**
- The fork runs on a memory-safe, async Rust engine while preserving the schema/migration model and the `GraphStore` interface, with a SQLite fallback so beta risk is bounded.
- A concrete upstream-contribution relationship with `github.com/tursodatabase/turso`, seeded by whatever parity gaps we hit (FTS being the expected first one).

**Measurable success criteria (to be tightened in SPEC):**
- Phase 1: A/B cost delta measurable from the product alone, with the de-duped method matching `pr_review_token_cost.py` to within rounding; per-agent inline payload ≤ an agreed token budget; new-file axes produce non-empty signal on a representative net-new-files PR.
- Phase 2: All existing `tests/` pass on the Turso path (with FTS parity) or on the SQLite fallback; at least one upstream issue/PR opened to `tursodatabase/turso` for a parity gap we hit.

---

## 6. Alternatives Considered

The skill requires at least one alternative to the recommendation. Three are material here, across two axes (token work and engine).

**Alternative A — Token work only, no engine change (Phase 1, skip Phase 2).**
Lowest risk. Keeps the proven SQLite engine. Delivers all the token/measurement/new-axes value. Loses the modernization/strategic option and the upstream-contribution relationship. *This is the highest-confidence option if the engine work is "nice to have."*

**Alternative B — Engine change only (Phase 2, skip Phase 1).**
Doesn't address the actual problem (output/serialization, not storage). Risks spending high effort on a beta-engine swap for no token benefit. *Not recommended on its own — it inverts the value/effort ordering.*

**Alternative C — Adopt libSQL (C fork, production-ready) instead of the Rust Turso. — *considered and rejected by owner***.
libSQL is the production-ready sibling: same SQLite compatibility, adds replication/embedded-replicas/vector-search, and would be a near drop-in for FTS5 (it *is* SQLite), at far lower risk than the beta Rust rewrite. **Owner decision (this turn): rejected — "only interested in using Turso."** The owner's goal is the Rust engine and an upstream-contribution relationship with `tursodatabase/turso`, not the lowest-risk swap. Recorded here to satisfy alternatives analysis, not as a live option.

**Alternative D — Keep SQLite, add a thin "shared graph" layer via the ROADMAP's "git-tracked DB" plan instead of any engine swap.**
Gets team sync (a ROADMAP planned item) without changing engines. Doesn't get memory-safety/async/Rust. *A reasonable Phase-2-free path to the one Turso feature we might actually want (shared graph).*

---

## 7. Tradeoff Analysis

| Dimension | Phase 1 only (Alt A) | Phase 1 + Turso Rust (chosen) | Phase 1 + libSQL (Alt C, rejected) |
|---|---|---|---|
| Token/measurement value | Full | Full | Full |
| Engine risk | None | Beta (mitigated: graph is a rebuildable cache + pinned releases) | Low (production-ready) |
| FTS parity effort | None | **High** — Tantivy≠FTS5, likely upstream contribution | None (FTS5 native) |
| Strategic/upstream upside | None | High (`tursodatabase/turso`) | Low |
| Memory safety / async | No | Yes | No (C) |
| Effort to first value | Low–medium | Medium (P1) then high (P2) | Medium (P1) then low–medium (P2) |
| "Fun"/learning value | Low | High | Low |

The central tradeoff: **the Rust Turso path is the highest-effort and highest-beta-risk engine option, but it is the only one that satisfies the owner's stated goal of using Turso and contributing upstream to `github.com/tursodatabase/turso`.** libSQL (Alt C) would be the pragmatic conservative choice on pure risk grounds, but the owner has explicitly rejected it in favor of Turso.

---

## 8. Standards / Constraints Analysis

There is no `docs/standards/index.yaml` in this repo (confirmed: `docs/standards/` does not exist), so the proposal-builder standards integration is lighter here and draws on the repo's own governance docs:

- **`AGENTS.md`** — issue tracking is `bd` (beads); session completion requires `git pull --rebase`, `bd dolt push`, `git push`. Any work arising from this proposal should be tracked via `bd`, not ad-hoc TODO lists.
- **`docs/architecture.md`** — documents the SQLite schema and the FTS5 virtual table as part of the architecture. Phase 2 must preserve this contract (or update the doc) — FTS parity is an architecture-level constraint, not a detail.
- **`docs/ROADMAP.md`** — lists "Team sync (shared graph via git-tracked DB)" as **Planned**. This is the one ROADMAP item that would make Turso's sync/replica features actually relevant to this product; it is explicitly *not* in Phase 2's default scope (see Non-Goals) but is the natural Phase 3.
- **`pr-review` skill (`zazz-skills/.../pr-review/`)** — the consuming process. Constraints it imposes on any new axes: findings must keep the `[boulder]/[rock]/[pebble]/[sand]` sizing; the two-axis (Standards + Spec) separation must be preserved; graph output stays *advisory* and must be verifiable against source. New complexity axes must surface as advisory signal the sub-agents can use, not as hard gates.
- **`code-review-graph.md` utility file** — declares the Zazz boundary: no upstream companion skills/hooks by default, graph output is advisory. Phase 1's compact `--for-review` mode must remain consistent with this boundary (it's an output improvement, not a behavior-change).

No exceptions to repo standards are requested by this proposal.

---

## 9. Risks and Mitigations

- **R1 — Turso is beta; data loss / bugs possible.** *Turso's own README says so verbatim.* **Mitigation (owner has accepted this risk to use Turso):** the graph is a **rebuildable derived cache** — gitignored and regenerable from source at any time — so a beta-engine bug costs rebuild time, not user data. No dual-engine fallback is planned (owner wants Turso as sole engine); instead, treat the graph as disposable, keep `build`/`update` robust, and pin to a known-good Turso release tag before each fork release.
- **R2 — FTS5 vs Tantivy FTS parity is the hardest technical blocker.** `nodes_fts` is FTS5 with `porter unicode61` and external-content-table semantics; Turso FTS is Tantivy with different DDL and query functions. **Mitigation:** this is Phase 2's first work item; budget for an upstream contribution to `tursodatabase/turso` (FTS5-compat surface or a translation layer). Because there is no alternate engine, if FTS parity can't be reached on Turso in a given release, that release ships with search degraded or deferred behind a flag — not with a different engine underneath.
- **R3 — Phase 2 effort swamps Phase 1 value.** An engine swap is high-effort and the token benefit lives in Phase 1. **Mitigation:** strict phasing — Phase 1 ships and is measured before Phase 2 starts in earnest; Phase 2 is explicitly re-justifiable after Phase 1's A/B results.
- **R4 — New "complexity axes" could bias reviewers like blast-radius did (§4A of the handoff).** Any new signal can be sized away as "matches siblings / low complexity." **Mitigation:** ship the sibling-template *divergence* signal (not just similarity) so the signal is "where this diverges from the template," which is harder to dismiss; keep the `pr-review` decompose rule in the prompt so both arms check line-by-line.
- **R5 — `pyturso` binding maturity.** The Python binding exists but is younger than the Rust/JS ones. **Mitigation:** validate `pyturso` against the full existing `tests/` suite early in Phase 2; if a blocking binding gap appears, file it upstream and keep a local shim or wait for a fix rather than switching engines.
- **R6 — Upstream contribution friction.** Contributing to `tursodatabase/turso` requires their CLA/process and may not merge on our timeline. **Mitigation:** design Phase 2 so the fork works with a *local* FTS-compat shim even if upstream merges slowly; upstream contribution is a bonus, not a prerequisite for shipping.

---

## 10. Dependencies and Sequencing Considerations

- **Phase 1 has no external dependency.** It is pure-Python work inside the fork. Internal ordering (from the prior fork analysis): repo-relative paths + deterministic output → compact `--for-review` mode → one-shot `review-context` → per-section scope → test-gap allowlist → scope-honest savings. The in-product token measurement and the new-file complexity axes can proceed in parallel with the output work.
- **Phase 2 depends on Phase 1's measurement being in place** so the engine swap can be evaluated against a trustworthy baseline (otherwise we can't tell whether the swap helped or hurt tokens — though the expectation is "neutral on tokens," since the token problem is not a storage problem).
- **Phase 2 external dependencies:** `pyturso` (Python binding to Turso), the Turso CLI (`tursodb`) for any local validation, and the `tursodatabase/turso` contribution process (CLA, issues/PRs).
- **Phase 3 (out of scope, named for context):** if the ROADMAP "shared graph via git-tracked DB" item becomes active, *that* is when Turso's sync/embedded-replica features become relevant — not before. Sequencing it after Phase 2 keeps the engine work decoupled from the distribution work.

---

## 11. Recommendation

**Proceed with the two-phase plan, Phase 1 first and Phase 2 contingent on Phase 1's measurement.**

- **Phase 1 — ship the token-efficiency output work + in-product trustworthy token measurement + new-file complexity axes.** This is where the actual value is, it is low-risk, and it makes every later decision (including whether Phase 2 is worth it) measurable. Order the output work P1 (repo-relative paths) → P3 (compact `--for-review`) → P2 (one-shot `review-context`) → P4 (per-section scope) → P5 (test-gap allowlist) → P7 (scope-honest savings), per the prior fork analysis.
- **Phase 2 — adopt the Rust Turso Database as the sole engine, embedded in-process via `pyturso` behind `GraphStore`, and contribute parity gaps (FTS first) upstream to `github.com/tursodatabase/turso`.** This satisfies the owner's stated Rust + upstream-contribution goal. Beta risk is bounded by the rebuildable-cache property (R1) and pinned releases, not by a fallback engine. It is *not* justified by token efficiency (the token problem is not a storage problem); it is justified by modernization, memory safety, async, and the strategic upstream relationship.
- **libSQL (Alternative C) is rejected** per owner decision — Turso is the sole engine target. If Phase 2 hits a hard blocker, the response is to fix it (and contribute upstream), fall back to a known-good pinned Turso release, or pause Phase 2 — not to switch engines.

**What must be true to proceed:** Phase 1 has no gate beyond approval. Phase 2 proceeds only after (a) Phase 1's in-product measurement shows a trustworthy baseline and (b) an early Phase-2 spike confirms `pyturso` can pass the existing `tests/` on at least the non-FTS path.

---

## 12. Decision Checklist / Approval Questions

1. Do you approve **Phase 1** (token-efficiency output + in-product measurement + new-file complexity axes) as described, to ship first?
2. Do you approve **Phase 2** (Rust Turso as the sole engine, embedded in-process via `pyturso` + upstream contributions to `github.com/tursodatabase/turso`), contingent on Phase 1's measurement? (No dual-engine fallback; beta risk bounded by rebuildable cache + pinned releases per R1.)
3. ~~Is the Rust Turso + upstream-contribution goal firm, or is libSQL acceptable as a fallback?~~ **Answered (this turn): Turso only; libSQL rejected.** Remaining: is the owner comfortable with **no dual-engine fallback** (relying on the rebuildable-cache mitigation in R1), or should a transitional SQLite path stay during Phase 2's early parity work?
4. For the new complexity axes: which dimensions matter most for your net-new-files work — cyclomatic/cognitive complexity, new-code mass, structural cohesion, sibling-template divergence, or others? (This shapes the Phase 1 axis selection before SPEC.)
5. Should **Phase 3 (shared graph via the ROADMAP's git-tracked DB / Turso sync)** be explicitly listed as a future follow-up, or left out of this proposal entirely?
6. Confirm issue tracking via `bd` (beads) per `AGENTS.md`, not ad-hoc TODO lists, for any work approved here.

---

## 13. Open Questions

- Which **complexity dimensions** should the new axes surface for net-new-file PRs? (See decision Q4.) The `pr-review` skill already cares about "hot path," "complexity," and "blast radius," but the *new-file* analogs need definition.
- How should the in-product token measurement handle the **sub-agent transcript** problem generically (not just Claude Code's `…/subagents/agent-*.jsonl` layout)? The external `pr_review_token_cost.py` is Claude-Code-specific; the in-product version should be provider-aware or provider-agnostic.
- For the compact `--for-review` mode, what is the **default token budget** per agent slice? (Proposal suggests ~1–2 KB; needs an owner number.)
- For Phase 2 FTS parity: is the preferred path an **FTS5-compatible surface inside the product** (translate to Tantivy at the `GraphStore` boundary) or an **upstream contribution to Turso** to improve FTS5 compatibility? (These have very different timelines.)
- Does the **Turso MCP server mode** (`tursodb db --mcp`) matter for this product, or is it thematic only? (We already expose MCP; overlap needs clarifying.)

---

## 14. Discussion Log / Notable Arguments

- **Owner position (this turn):** Two-phase. Phase 1 = token efficiency + accurate token measurement built into the product + new axes for large-swaths-of-new-files work (existing metrics often inapplicable). Phase 2 = Turso Rust engine, and we may bug-fix + contribute upstream to `github.com/tursodatabase/turso`. Read the `pr-review` skill for context.
- **Facilitator tension surfaced (honestly):** the prior storage analysis concluded the token problem is an **output/serialization** problem, *not* a storage problem. Therefore the engine swap is **not justified by token efficiency**; it is justified by modernization/Rust/upstream-contribution. The proposal keeps both phases but does not claim the engine swap buys tokens.
- **Facilitator constraint surfaced:** `code-review-graph` depends on SQLite **FTS5**; Turso's FTS is **Tantivy-based and experimental** with different syntax. This is the hardest Phase-2 blocker and the most likely upstream-contribution target. Turso is explicitly **BETA** per its own README; a SQLite fallback is non-negotiable.
- **`pr-review` skill constraints noted:** graph is advisory; two-axis (Standards + Spec) and `[boulder]/[rock]/[pebble]/[sand]` sizing must be preserved; new axes extend, don't replace. Graph usage is gated on changed-file count > 10.
- **Owner decision (this turn):** Turso only — "only interested in using Turso; it's cool." libSQL (Alternative C) explicitly rejected; the Rust engine and the upstream-contribution relationship with `tursodatabase/turso` are the point, not risk minimization. Phase 2 = integrate Turso as the engine inside the same application (in-process via `pyturso`), not a separate DB service.
- **No stakeholder disagreement recorded** — single-stakeholder dialogue; the engine choice is now settled. Remaining open items: Phase 1 complexity-axis selection (Q4) and the no-fallback risk posture.

---

## 15. Sign-off Outcome and Next-Phase Handoff

**Status:** Not yet approved — draft for discussion. No sign-off recorded.

**On approval, the handoff into the next authoritative phase would carry:**
1. **Approved scope:** joint — Phase 1 (feature-scoped review-quality + deliverable-scoped output work) and Phase 2 (deliverable-scoped engine swap), sequenced.
2. **Final recommendation:** Phase 1 first; Phase 2 = Turso (Rust) as sole engine, embedded in-process; libSQL rejected. Beta risk bounded by rebuildable cache + pinned releases, not a fallback engine.
3. **Chosen approach and rejected alternatives:** chosen = Phase 1 + Rust Turso as sole engine (Alt recommendation); rejected = engine-only (Alt B, inverts value/effort); rejected by owner = libSQL (Alt C) — Turso only. If Phase 2 hits a hard blocker, the response is fix/upstream/pin/pause, not engine-switch.
4. **Key constraints and standards implications:** preserve `GraphStore` interface + schema/migration model; preserve FTS5 contract or update `docs/architecture.md`; track work in `bd`; respect `pr-review` skill's advisory/sizing/two-axis constraints.
5. **Risks to cover in SPEC:** R1 beta data loss, R2 FTS parity, R3 Phase 2 effort swamps Phase 1, R4 new-axis bias, R5 `pyturso` maturity, R6 upstream friction.
6. **Open questions to resolve in feature-document/SPEC dialogue:** the six in §13, especially Q4 (complexity dimensions) and the FTS-parity-path question.
7. **Suggested initial focus areas:** Phase 1 — repo-relative path normalization + in-product token measurement as the first two concrete deliverables; Phase 2 — an FTS-parity spike against `pyturso` + the existing `tests/` suite as the first go/no-go gate.

**Next-phase skills to hand into:** `feature-doc-builder` (for the review-quality/axes feature scope) and/or `spec-builder` (for the deliverable-scoped output work and the engine swap), depending on how the approved scope is split.

---

### Appendix — source references (key file:line citations)

Storage / engine:
- `code_review_graph/graph.py:1,143-167,32-79` — SQLite `GraphStore`, WAL, schema.
- `code_review_graph/migrations.py:1-285` — versioned migrations through v9.
- `code_review_graph/migrations.py:147-157` — `nodes_fts` FTS5 virtual table (Phase-2 blocker).
- `code_review_graph/incremental.py:301-324` — `get_db_path` → `.code-review-graph/graph.db`.

Token / output path (Phase 1):
- `code_review_graph/cli.py:1240` — full-JSON `detect-changes` output (the ~110 KB blob).
- `code_review_graph/cli.py:1217-1238` — `--brief` path (summary + savings panel only).
- `code_review_graph/tools/review.py:356,449-462` — `detect_changes_func` + `detail_level="minimal"`.
- `code_review_graph/tools/review.py:25` — `get_review_context` (no path/subsystem filter today).
- `code_review_graph/changes.py:277-409` — `analyze_changes` (risk scoring, test gaps, priorities).
- `code_review_graph/changes.py:330-333` — `CRG_MAX_CHANGED_FUNCS` cap.
- `code_review_graph/context_savings.py:13-32,78-82,201-317` — 4-chars/token estimate, `estimated:true`, savings panel.
- `code_review_graph/parser.py:999` and `code_review_graph/incremental.py:808-815,862` — absolute `file_path` storage.
- `code_review_graph/tools/_common.py:103-146` — `_resolve_graph_file_paths` (input-side path bridge; no output-side equivalent).

Sibling / community (Phase 1 new axis):
- `code_review_graph/communities.py` — Leiden community detection (igraph) + file-based fallback.
- `code_review_graph/analysis.py` — hub/bridge/surprise/gap analysis (no sibling-template *divergence* today).
- `code_review_graph/tools/build.py:36-60` — per-node `signature` computation (backing for sibling-divergence).

Consumer process:
- `zazz-skills/.agents/skills/pr-review/SKILL.md` — two-axis design, >10-file graph gate, sizing/approval rules.
- `zazz-skills/.agents/skills/pr-review/shared-rules.md` — `[boulder]/[rock]/[pebble]/[sand]` sizing, diff-scope discipline.
- `zazz-skills/.agents/skills/pr-review/code-review-graph.md` — Zazz boundary, advisory use, setup choices.
- `code-review-graph-fork-and-ab-handoff.md` (sibling) — the A/B experiment, corrected token methodology, §4 hypotheses, §4A finding-quality difference.

External:
- Turso Database (Rust, beta; formerly Limbo): https://github.com/tursodatabase/turso
