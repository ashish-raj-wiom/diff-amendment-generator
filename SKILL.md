---
name: diff-amendment-generator
description: Produce a deployment-ready diff-amendment file for a base spec (YAML / JSON / Markdown PRD, governance doc, API contract). "Deployment-ready" = applyable surgically, matches live code, internally and cross-DIFF consistent, every external call / mutable field / state transition fully specified (failure, retry, concurrency, lifecycle), and governance-clean on OS invariants + consumer contracts + project conventions. Never modifies the base spec. Use when the user says "create an amendment", "generate a diff amendment", "write a patch amendment", "amend this spec", or references a base spec + a change request and needs an AMENDMENT-NN file. Bakes in one canonical format; enforces a codebase reality-check (including type discipline), a scenario-coverage pass, a cross-DIFF consistency sweep, and a three-sided governance check (OS + TAS + project conventions). Output ships in one pass — no follow-up questions from tech review.
---

# diff-amendment-generator

## What this skill does

Produces a **deployment-ready diff-amendment** for a base spec. Takes a base spec + a change request + the codebase + prior amendments, and outputs a single standalone amendment file in a canonical format. Never modifies the base spec.

"Deployment-ready" means the file the tech team receives is applyable and ships in one pass:

- **Applies surgically** — unambiguous anchors, reviewer-applyable patch
- **Matches live code** — every claim verified via codebase grep + cross-boundary type alignment
- **Internally consistent** — sibling DIFFs agree on shared rules (resolution strategies, ID types, naming conventions)
- **Fully specified** — every external call, new field, state transition, and listener declares failure / retry / concurrency / lifecycle semantics
- **Governance-clean** — passes OS invariants, consumer contracts, and project conventions
- **Format-stable** — one canonical structure regardless of what predecessor amendments look like

The skill refuses to draft anything until all gates pass. Tech review finds nothing left to ask.

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
4. **Codebase source** — both of:
   - `local_path:` root of the local git clone of the repo that integrates the diff
   - `expected_remote_url:` the canonical remote (e.g. `https://github.com/org/repo`) the local path should point at
   - `branch:` the branch the amendment targets (e.g. `qa`, `main`, `release-01`)
   Skill will verify `local_path` is a git clone of `expected_remote_url` on `branch`, run `git fetch` (read-only), and check `git status -sb`. If local is behind / ahead / dirty / pointing at the wrong remote, drafting blocks until user either pulls, cleans, or explicitly acknowledges the divergence.
5. **Prior amendments** — paths to all amendments already applied to the base (in apply order), or explicit "none." Drives anchor resolution, prereqs declaration, conflict detection.
6. **Target deploy path** — where the amendment will land

**Strongly recommended:**

7. **OS governance files** — base spec's own invariants. Drives Phase 5a.
8. **TAS governance files** — downstream consumer-contract invariants. Drives Phase 5b.
9. **Project conventions doc** — error envelopes, endpoint naming patterns, standard response shapes, registration patterns. Drives Phase 5c.

**Optional:**

10. **Slice scope** — "X flow only" for scope fencing
11. **Codebase navigation hints** — migration path, DTO path, service path

Use `AskUserQuestion` once at intake for anything missing from #1–#6. Don't invent values. For #4 specifically, never assume a pre-existing local folder is the right clone — always verify remote URL + branch + freshness.

## 7-phase workflow

**Phase 1 — Orientation** (read-only; includes codebase freshness gate)

**1a — Codebase freshness check (mandatory gate):**
- Verify `local_path` (input #4) is a git repository: `git -C <local_path> rev-parse --is-inside-work-tree`.
- Verify the configured remote matches `expected_remote_url`: `git -C <local_path> config --get remote.origin.url`. If mismatched, BLOCK with clear error — this is the wrong clone.
- Verify current branch matches `branch`: `git -C <local_path> rev-parse --abbrev-ref HEAD`. If mismatched, BLOCK or prompt user to switch.
- Run `git fetch origin <branch>` (read-only; updates remote-tracking refs, never mutates working tree).
- Run `git status -sb` to check local state vs fetched remote.
- Report one of:
  - `UP-TO-DATE` — local matches `origin/<branch>`, clean working tree → proceed.
  - `BEHIND` — local is behind `origin/<branch>` by N commits → BLOCK. Ask user to pull or explicitly confirm they want to anchor against the stale version.
  - `AHEAD` — local has commits not on origin → BLOCK. Ask user whether those commits should be part of the amendment's "effective base" or ignored.
  - `DIRTY` — uncommitted changes in working tree → BLOCK. Ask user to stash or confirm which tree state to anchor against.
  - `DIVERGED` — both ahead and behind → BLOCK. Investigate before proceeding.
- Do not run `git pull`, `git merge`, `git checkout`, or any working-tree mutation. Only `git fetch` (remote-tracking refs) and read-only inspection commands.

**1b — Spec parsing:**
- Parse base spec. Inventory top-level sections + anchor keys.
- Parse every prior amendment in apply order. Build effective-anchor map (see `reference/prior_amendments_handling.md`).

**1c — Orientation report:**
- Codebase freshness state (from 1a) — explicit PASS/BLOCK.
- Section inventory + prior-amendment footprint + effective anchors.
- `git log -5 --oneline` of `origin/<branch>` so the user sees the most recent commits the amendment is anchoring against.

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

- **Git:** read-only on the working tree. Allowed: `git log`, `git diff`, `git status`, `git fetch` (updates remote-tracking refs only, never mutates the working tree), `git rev-parse`, `git config --get remote.*`. Forbidden: `git pull`, `git merge`, `git checkout`, `git reset`, `git clone` (unless explicitly confirmed by user), `git commit`, `git push`.
- **Codebase freshness:** mandatory gate in Phase 1a. Never trust a pre-existing local folder without verifying remote URL + branch + freshness via `git fetch` + `git status -sb`.
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
- **Never trust a pre-existing local codebase folder** without verifying its remote URL, branch, and freshness state via the Phase 1a gate. A stale local clone can pass `grep`-level reality checks while being wrong about what actually ships.
- Never proceed past Phase 1a with BEHIND / AHEAD / DIRTY / DIVERGED codebase state unless the user explicitly acknowledges the divergence.
- Never proceed past Phase 3 with unacknowledged DRIFT / GAP / PHANTOM / type-mismatch items.
- Never proceed past Phase 4b with unanswered scenario-coverage questions.
- Never proceed past Phase 4c with unresolved cross-DIFF contradictions.
- Never proceed past Phase 5 with PENDING concerns.
- Never use `anchor_base_line:` alone when a semantic anchor is available.
