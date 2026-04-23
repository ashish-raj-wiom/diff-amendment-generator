---
name: diff-amendment-generator
description: Generate a standalone diff-amendment file for a base spec (YAML / JSON / Markdown PRD, governance doc, API contract). Produces a surgical, reviewer-applyable patch — never modifies the base. Use when the user says "create an amendment", "generate a diff amendment", "write a patch amendment", "amend this spec", or references a base spec + a change request and needs an AMENDMENT-NN file. Bakes in one canonical format; enforces a codebase reality-check (including type discipline), a scenario-coverage pass, a cross-DIFF consistency sweep, and a three-sided governance check (OS + TAS + project conventions) so the amendment ships in one pass without tech-review churn.
---

# diff-amendment-generator

## What this skill does

Takes a base spec + a change request + the codebase + prior amendments, and produces a single standalone amendment file in a canonical format. Never modifies the base spec. Output is reviewable and surgically applyable by a backend team in one pass — the skill enforces enough checks (reality, scenario coverage, cross-DIFF consistency, governance) that tech review should find zero gaps to answer.

## Trigger

Invoke when the user asks to:
- "create an amendment for <spec>"
- "generate a diff amendment"
- "amend this PRD / YAML / governance doc"
- "write AMENDMENT-<N> that does X"
- "make a patch file against <spec>"

Do **not** use for rewriting specs in place, generating full spec versions, or producing non-diff documentation. For those, redirect the user to the relevant domain skill (e.g., `execution-spec-generator` for new specs).

## Required inputs — ask the user up front

**Mandatory (skill refuses to proceed without these):**

1. **Base spec file** — path to the OLD being amended
2. **Change request** — list / newer-version / plain-English description of what to change
3. **Amendment number** — e.g. `AMENDMENT-04`, `v2.1-patch-1`
4. **Codebase pointer** — root path of the repo that integrates the diff. Spec is aspirational; code is authoritative.
5. **Prior amendments** — paths to all amendments already applied to the base (in apply order), or explicit "none." Drives anchor resolution, prereqs declaration, conflict detection.
6. **Target deploy path** — where the amendment will land

**Strongly recommended:**

7. **Upstream git ref** — so the skill can inspect `git log` / `git diff` (read-only)
8. **OS governance files** — base spec's own invariants. Drives Phase 5a.
9. **TAS governance files** — downstream consumer-contract invariants. Drives Phase 5b.
10. **Project conventions doc** — error envelopes, endpoint naming patterns, standard response shapes, registration patterns. Drives Phase 5c.

**Optional:**

11. **Slice scope** — "X flow only" for scope fencing
12. **Codebase navigation hints** — migration path, DTO path, service path

Use `AskUserQuestion` once at intake for anything missing from #1–#6. Don't invent values.

## 7-phase workflow

**Phase 1 — Orientation** (read-only)
- Parse base spec. Inventory top-level sections + anchor keys.
- Parse every prior amendment in apply order. Build effective-anchor map (see `reference/prior_amendments_handling.md`).
- If upstream git ref provided: `git log --oneline` and `git diff` to flag uncommitted deltas. Never `git pull`, never mutate.
- Report: section inventory + prior-amendment footprint + effective anchors.

**Phase 2 — Intake & classification**
- Normalise change request into discrete items.
- Classify each by sub_op type using the closed set in `reference/sub_op_vocabulary.md`.
- AskUserQuestion for ambiguous classifications.

**Phase 3 — Reality check against codebase** (always runs; gate)
- For each change item, grep the codebase per `reference/reality_check_playbook.md`.
- Classify: `MATCH` / `DRIFT` / `GAP` / `PHANTOM`.
- **Type discipline sub-check:** for every cross-boundary field (payload ↔ entity, producer ↔ consumer, wire ↔ domain), verify types align. A silent `String → Long` cast, `UUID → Long` parse, or required/optional flip is a DRIFT.
- Present code-truth report to user. Do not proceed past Phase 3 until every non-MATCH item is acknowledged.

**Phase 4 — Minimality + coverage + consistency** (three sub-phases)

- **Phase 4a — Minimality pass.** Apply the 12 rules in `reference/minimality_heuristics.md`. Fold related changes into MULTI. Challenge nice-to-haves. Prefer semantic anchors. *(Heuristic — doesn't gate.)*

- **Phase 4b — Scenario coverage** (gate). Run `reference/scenario_coverage_check.md` against every DIFF. For each external call, every new mutable field, every state-touching step, force explicit answers to:
  - **Failure path** — what happens on 4xx / 5xx / timeout / null / exception?
  - **Retry policy** — retry? count? backoff? circuit-break?
  - **Concurrency** — which other paths can race with this? What's the outcome per interleaving? Locking strategy?
  - **Lifecycle** — for new fields/flags/tables/tokens: set_by / read_by / reset_by / unique_on / orphan_policy.
  - **Boundary inputs** — empty / null / over-long / multi-row / non-numeric / duplicate.
  Any unanswered question → `[CONCERN-N]`. Drafting blocks until resolved.

- **Phase 4c — Cross-DIFF consistency sweep** (gate). Run `reference/cross_diff_consistency_check.md`. Checks sibling DIFFs agree on shared rules:
  - Multi-candidate / multi-row resolution strategies (e.g. "take first" vs "for each" across sibling inbound listeners)
  - Naming/prefix conventions (routing keys, queues, envelopes)
  - ID types at boundary crossings (e.g. `customer_id` is UUID or Long — pick one and hold it across all DIFFs)
  - Transport vs registry alignment (e.g. if DIFF D registers in HTTP producer list, DIFF J must either use HTTP transport or the registration is inconsistent)
  - Actor types on audit rows (human actorId vs ActorType.SYSTEM — don't mix)
  Contradictions → `[CONCERN-N]`. Drafting blocks until resolved.

**Phase 5 — Governance validation** (three-sided; all run; each can gate)

- **Phase 5a — OS governance check** — `reference/os_governance_check.md`. Base-spec invariants: PII, state-machine, event-registry, guards, schema, naming. Widened to catch **policy-vs-enforcement gaps**: any `MUST` / `SHOULD` / invariant statement without a runtime enforcement mechanism (not merely a machine-readable guard condition) is flagged.
- **Phase 5b — TAS governance check** — `reference/tas_governance_check.md`. Consumer-contract invariants: aggregation guards, payload schemas, bridge classification, identity-lock.
- **Phase 5c — Project-convention alignment** — `reference/convention_alignment_check.md`. Match new error envelopes, endpoint shapes, registration patterns, unregister semantics against the project conventions doc (input #10).
- Per DIFF, record `os_check`, `tas_check`, `conv_check` → PASS | CONCERN-N.
- If any governance input missing: run reduced check, flag the limitation in the amendment header. Do not skip silently.
- All concerns must be RESOLVED before Phase 6.

**Phase 6 — Draft**
- Write the amendment file using the embedded format in `reference/embedded_format.md`.
- One DIFF per coherent slice. Each DIFF has `governance_posture:` block.
- END block summarises DIFFs and concern resolutions.

**Phase 7 — Hygiene**
- Scrub stale inline comments (e.g. `[CONCERN-N] PENDING` left over after resolution).
- Consistency-check: every numbered entity declared once, cross-references bidirectional.
- Folder tidy: co-locate base spec + change request + prior amendments. **Propose** renaming any superseded drafts (`DRAFT-*-superseded.*` / `DRAFT-*-unused.*`) — do not auto-rename.

## Default behaviours (auto-mode safe)

- **Git:** read-only. Use `git log`, `git diff`, `git status`. Never `git pull`, never commit, never push.
- **Superseded draft rename:** propose to user, do not auto-rename.
- **File creation:** only the amendment file, in the folder the user specified or adjacent to the base spec. Do not create new folders without confirmation.
- **Base spec:** never modify. If the user asks to edit the base directly, push back — that's not this skill's job.

## Reference docs (load on demand)

- `reference/embedded_format.md` — header / DIFF / END templates
- `reference/sub_op_vocabulary.md` — closed set of operations
- `reference/anchor_conventions.md` — preferred → fallback
- `reference/minimality_heuristics.md` — the 12 rules (Phase 4a)
- `reference/scenario_coverage_check.md` — Phase 4b coverage questions
- `reference/cross_diff_consistency_check.md` — Phase 4c consistency sweep
- `reference/concern_flagging_pattern.md` — `[CONCERN-N]` structure
- `reference/reality_check_playbook.md` — Phase 3 grep strategies + type discipline
- `reference/prior_amendments_handling.md` — anchor math
- `reference/os_governance_check.md` — Phase 5a checklist (incl. policy-vs-enforcement)
- `reference/tas_governance_check.md` — Phase 5b checklist
- `reference/convention_alignment_check.md` — Phase 5c project-conventions checklist

## Examples

- `examples/example-minimal-amendment.yaml` — 2-DIFF toy amendment, annotated
- `examples/example-multi-diff-amendment.yaml` — larger merged amendment with concerns

## Hard rules

- Never modify the base spec or prior amendments.
- Never add a DIFF without a specific change-request item backing it.
- Never introduce operations outside the closed sub_op vocabulary unless strictly symmetric to an existing one (e.g. `INSERT_BLOCK` is symmetric to `REMOVE_BLOCK`) — and declare the extension in the header.
- Never proceed past Phase 3 with unacknowledged DRIFT / GAP / PHANTOM / type-mismatch items.
- Never proceed past Phase 4b with unanswered scenario-coverage questions.
- Never proceed past Phase 4c with unresolved cross-DIFF contradictions.
- Never proceed past Phase 5 with PENDING concerns.
- Never use `anchor_base_line:` alone when a semantic anchor is available.
