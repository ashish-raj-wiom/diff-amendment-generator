# Diff Amendment Generator — Claude Code Skill

A Claude Code skill that turns a **base spec + a change request + a codebase** into a **standalone diff-amendment file** that a backend team can apply surgically. Never modifies the base spec. Output is minimal, format-consistent, and pre-validated against code reality, governance invariants, and cross-DIFF consistency so it ships in one tech-review pass.

---

## What problem does this solve?

*"Here's AMENDMENT-04. Please implement."* → *"We have 22 questions."*

Governance YAML / PRD / service-spec amendments often fail tech review not because of bugs, but because of **unanswered questions**:

- "What happens if this external call times out?"
- "These two DIFFs describe the same lookup but with different resolution rules — which one wins?"
- "This field is declared as `Long` here and `UUID` there. Which is right?"
- "The spec says `MUST NOT render the OTP` but the flow requires rendering it. How is the policy enforced?"
- "You registered this event in the HTTP producer list but the transport is Rabbit. Bug?"

Each of these kicks the amendment back to the author for a round-trip, costing days. This skill front-loads all of them at author time.

---

## What you get

A single **`AMENDMENT-<N>.yaml`** file in a canonical format with:

- Header block (purpose / scope / out-of-scope / target / prereqs / coordination / concern status)
- One DIFF per coherent slice of change
- Each DIFF carries its `governance_posture:` block recording `os_check`, `tas_check`, `conv_check`, plus scenario-coverage counts
- END block with DIFF-count summary and concern resolutions
- Matching `CONCERN STATUS:` entries in the header where PENDING → RESOLVED via option `(X)` is explicit

The skill refuses to produce the amendment until every gate passes — tech review finds nothing left to ask.

---

## How it works — 7 phases with 3 gates

```
Phase 1 — Orientation        (read-only)
Phase 2 — Intake & classify
Phase 3 — Reality check      ★ GATE (code-truth + type discipline)
Phase 4a — Minimality
Phase 4b — Scenario coverage ★ GATE (failure / retry / concurrency / lifecycle)
Phase 4c — Cross-DIFF sweep  ★ GATE (sibling-DIFF contradictions)
Phase 5a — OS governance
Phase 5b — TAS governance
Phase 5c — Project conventions
Phase 6 — Draft
Phase 7 — Hygiene
```

Every gate produces numbered `[CONCERN-N]` blocks with 2–3 options and a recommendation. Drafting blocks until the user resolves each.

---

## What inputs the skill needs

**Mandatory (skill won't proceed without these):**

1. Base spec file
2. Change request (list / richer version / plain-English)
3. Amendment number (e.g. `AMENDMENT-04`)
4. Codebase root path — spec is aspirational; code is authoritative
5. Prior amendments (ordered list) or explicit "none"
6. Target deploy path

**Strongly recommended:**

7. Upstream git ref (read-only `git log` / `git diff`)
8. OS / base-spec governance files — drives Phase 5a
9. TAS / consumer-contract governance files — drives Phase 5b
10. Project conventions doc — drives Phase 5c

**Optional:**

11. Slice scope ("X flow only")
12. Codebase navigation hints (migration path, DTO path, service path)

---

## Install

### One-liner (macOS / Linux / Windows Git Bash)

```bash
git clone https://github.com/ashish-raj-wiom/diff-amendment-generator ~/.claude/skills/diff-amendment-generator
```

### Windows PowerShell

```powershell
git clone https://github.com/ashish-raj-wiom/diff-amendment-generator "$HOME\.claude\skills\diff-amendment-generator"
```

That's it. Claude Code auto-detects skills in `~/.claude/skills/`. Restart your Claude Code session if it was already open.

---

## Use

In Claude Code, say any of these:

- `create an amendment for <spec>`
- `generate a diff amendment`
- `amend this PRD / YAML`
- `write AMENDMENT-<N> that does <X>`
- `make a patch file against <spec>`

The skill will ask for the 6 mandatory inputs, then walk the 7 phases — presenting concerns batched at each gate for your decision before drafting.

---

## Design principles

- **Format is embedded.** The skill produces one canonical style, regardless of what predecessor amendments look like. No format drift.
- **Code is ground truth.** Phase 3 greps the codebase for every claim the amendment would make. Mismatches block drafting.
- **Minimality + coverage.** Minimality cuts speculative *additions*; scenario coverage cuts speculative *ambiguity*. Both apply.
- **Cross-DIFF consistency.** The amendment is validated as a single coherent artefact, not a bag of independent DIFFs.
- **Three-sided governance.** Base-spec invariants (OS), consumer contracts (TAS), and project-wide conventions all checked.
- **Concerns are explicit.** Every risk → numbered `[CONCERN-N]` with options + recommendation. User picks; skill writes. No silent drafting decisions.

---

## Bundle layout

```
diff-amendment-generator/
├── README.md                           # this file
├── LICENSE                             # MIT
├── SKILL.md                            # entry point — workflow + hard rules
├── reference/                          # load-on-demand reference docs
│   ├── embedded_format.md
│   ├── sub_op_vocabulary.md
│   ├── anchor_conventions.md
│   ├── minimality_heuristics.md        # Phase 4a — 12 rules
│   ├── scenario_coverage_check.md      # Phase 4b — 6 categories of coverage
│   ├── cross_diff_consistency_check.md # Phase 4c — 10 consistency rules
│   ├── concern_flagging_pattern.md
│   ├── reality_check_playbook.md       # Phase 3 — grep + type discipline
│   ├── prior_amendments_handling.md    # anchor math
│   ├── os_governance_check.md          # Phase 5a — 6 OS checks
│   ├── tas_governance_check.md         # Phase 5b — 4 consumer-contract checks
│   └── convention_alignment_check.md   # Phase 5c — 8 project-convention checks
└── examples/
    ├── example-minimal-amendment.yaml
    └── example-multi-diff-amendment.yaml
```

---

## Default behaviours (auto-mode safe)

- **Git:** read-only. Uses `git log` / `git diff` / `git status` only. Never `git pull`, never commits, never pushes.
- **Base spec:** never modified. If asked to edit the base directly, skill pushes back.
- **Superseded drafts:** proposed for rename in Phase 7 (`DRAFT-*-superseded.*`) — not auto-renamed.
- **File creation:** only the amendment file, in the folder the user specified or adjacent to the base spec.

---

## Status

v1 — in use for governance / service-spec amendments. Contributions via PR welcome; issues welcome too.

## License

MIT.
