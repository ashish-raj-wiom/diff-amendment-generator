# Embedded format — canonical

This is the format every amendment this skill produces. No predecessor matching; always this structure.

## File-level structure

```
<Header block>

<DIFF A>
<DIFF B>
...

<END block>
```

## Header block

```yaml
# ==============================================================================
# AMENDMENT-<N> — <spec filename>
# ==============================================================================
# Purpose:      <one-sentence summary of what this amendment does>
# Scope:        <which slice of the spec this touches>
# Out of scope (explicit):
#   - <bullet 1 — non-obvious exclusion>
#   - <bullet 2>
# Target:       <deploy path, e.g. repo/path/to/spec.yaml>
# Prereqs:      AMENDMENT-<M>, AMENDMENT-<M+1>, ... already applied  (or "none")
# Coordination: <deploy sequencing / cross-service notes / consumer dependencies>
#
# CONCERN STATUS (if any concerns arose in Phase 5):
#   [CONCERN-1] RESOLVED via option (b) — <one-line summary>
#   [CONCERN-2] PENDING                  — <one-line summary>
# ==============================================================================
```

Rules for the header:
- `Purpose:` — one sentence. If you can't summarise in one sentence, the amendment is probably doing too much.
- `Out of scope:` — list things a reasonable reviewer might assume are included. Silence on obvious exclusions is fine; call out non-obvious ones.
- `Prereqs:` — every prior amendment in the order it was applied. If the user said "none," write `Prereqs: none`.
- `Coordination:` — anything that affects release sequencing (new dependency goes live before caller; consumer subscribes to bus before producer emits; etc.). Prose, not a DIFF.
- `CONCERN STATUS:` — omit if no concerns. Otherwise list every concern with its state.

## DIFF block

```yaml
# ------------------------------------------------------------------------------
# DIFF <LETTER> — <section being touched, short>
# <1-2 sentence intent — why this DIFF exists>
# ------------------------------------------------------------------------------
diff_<letter>_<snake_case_name>:
  anchor_section: "<section path in the base spec>"
  anchor_<type>: "<value>"              # semantic anchor — see anchor_conventions.md
  anchor_base_line: <N>                  # optional fallback; only if semantic unavailable
  operation: <TOP_LEVEL_OP>              # INSERT | INSERT_LIST_ITEM | MULTI | RENAME_ALL | REFERENCE_ONLY

  # Contents vary by operation type. See sub_op_vocabulary.md.
  # For INSERT:             new_entry: [<block>]
  # For INSERT_LIST_ITEM:   insert_after_item: "<str>";  new_items: [<scalar>, ...]
  # For MULTI:              changes: [<list of sub_op dicts>]
  # For RENAME_ALL:         renames: [<list of rename dicts>]
  # For REFERENCE_ONLY:     artefact_path: "<path>";  rule: "<one line>";  status: DEPLOYED

  governance_posture:
    state_machine:   "NO CHANGE"          | "NEW transitions: ..."
    schema_change:   "NONE"               | "additive: V<N> adds <column/table>"
    new_events:      <count>
    new_guards:      <count>
    os_check:        "PASS"               | "CONCERN-<N>"
    tas_check:       "PASS"               | "CONCERN-<N>"

  note: "<optional — anything that doesn't fit the above fields>"
```

Rules:
- `diff_<letter>_<snake_case_name>:` — letters A, B, C... in file order. Name describes what the DIFF does, not where it is.
- `anchor_section:` — always set. The YAML path or section name that scopes this DIFF's target.
- `anchor_<type>:` — preferred semantic anchor from `anchor_conventions.md`. Use `anchor_after_X` / `anchor_before_Y` / `anchor_entry:` / `anchor_event_id:` etc. Pick one.
- `anchor_base_line:` — include only if the base spec truly has no stable semantic anchor (rare). Flag this as a weakness in the DIFF's comment.
- `operation:` — from the closed top-level set.
- `governance_posture:` — mandatory. Four structural fields (state_machine, schema_change, new_events, new_guards) plus the two check outcomes.

## END block

```yaml
# ==============================================================================
# END OF AMENDMENT-<N>
# ==============================================================================
# <N> DIFFs. <brief overall outcome>.
#   A — <one-line summary>        [C-<M> RESOLVED via (b), if applicable]
#   B — <one-line summary>
#   ...
# Anchor line numbers, where used, are base-spec lines AT PRE-AMENDMENT-<N>
# APPLY TIME; verify before applying.
# ==============================================================================
```

Rules:
- Count DIFFs explicitly — easy reviewer sanity check.
- Summarise each DIFF in one line. Concern resolution state in brackets.
- End with the anchor-line-drift warning.

## Full skeleton example (empty)

```yaml
# ==============================================================================
# AMENDMENT-05 — example-service-prd.yaml
# ==============================================================================
# Purpose:      <fill in>
# Scope:        <fill in>
# Out of scope (explicit):
#   - <fill in>
# Target:       repo/path/to/spec.yaml
# Prereqs:      AMENDMENT-01, AMENDMENT-02, AMENDMENT-03, AMENDMENT-04 already applied
# Coordination: <fill in>
# ==============================================================================

# ------------------------------------------------------------------------------
# DIFF A — <section>
# <intent>
# ------------------------------------------------------------------------------
diff_A_<name>:
  anchor_section: "<path>"
  anchor_after_entry: "<name>"
  operation: INSERT
  new_entry:
    - <fields>
  governance_posture:
    state_machine: "NO CHANGE"
    schema_change: "NONE"
    new_events: 0
    new_guards: 0
    os_check: PASS
    tas_check: PASS

# ==============================================================================
# END OF AMENDMENT-05
# ==============================================================================
# 1 DIFF.
#   A — <one-line summary>
# ==============================================================================
```

See `examples/` for filled-in versions.
