# Prior amendments handling

The skill treats every amendment as layered on top of the base spec + all prior amendments applied in order. Anchors, prereqs, and conflict detection all depend on the prior-amendment set being parsed correctly at Phase 1.

## Input contract

User must supply input #5 (Prior amendments) as one of:

1. **Path to a folder** containing all prior amendment files. Skill sorts by filename convention (e.g. `AMENDMENT-01.yaml`, `AMENDMENT-02.yaml`, ...).
2. **Ordered list of file paths.** Explicit apply-order.
3. **Literal string `"none"`** — base spec is virgin, this is AMENDMENT-01.

If the input is ambiguous (e.g. a folder with non-standard names and no numeric ordering), the skill asks the user to supply an explicit ordered list.

## Parsing each prior amendment

For each prior amendment file, extract:

1. **Amendment identifier** — e.g. `AMENDMENT-02` (from filename or header).
2. **Every DIFF** and its:
   - Target section (`anchor_section:`)
   - Target entity names (`anchor_after_*`, `anchor_before_*`, `anchor_entry:`, `anchor_event_id:`, etc.)
   - Operation and sub_ops
   - `governance_posture:` summary (quick scan — did it touch state machine? add events? rename things?)
3. **Inserts, removals, and renames** at entity-name granularity.

Skill doesn't need to fully interpret the sub_op semantics — just know which sections and which entities were touched.

## Building the effective-state map

Produce a map keyed by spec section, with values listing all prior-amendment operations against it:

```
state_machine:
  - AMENDMENT-02: UPDATE_FIELD initial_state
  - AMENDMENT-02: REMOVE_BLOCK states[AWAITING_PARTNER_RESPONSE]
  - AMENDMENT-03: ADD_STATE INSTALLATION_EXPIRED

events.produces:
  - AMENDMENT-02: REMOVE_BLOCK events[ES_INSTALL_DECLINED]
  - AMENDMENT-03: ADD_ENTRY events[BOOKING_ROHIT_ARRIVED]

dependencies:
  (untouched)
```

And a **rename ledger** tracking identity changes:

```
RENAME AMENDMENT-02:
  state_name: WITHDRAWN_BY_CSP → INSTALLATION_REPORTED_FAILED
  event_id:   ES_INSTALL_WITHDRAWN_BY_CSP → ES_INSTALL_REPORTED_FAILED
  db_column:  withdrawal_reason → failure_reason
  db_column:  withdrawn_at → failure_reported_at
```

## Conflict detection

For each change-request item in the new amendment, check:

1. **Does the item target a section that prior amendments have already heavily modified?**
   - If yes: surface as a heads-up; the current amendment may need to account for the modified state.

2. **Does the item reference an entity that was renamed?**
   - If yes: the amendment must use the post-rename name. The skill translates the change request's names into the post-rename names before drafting.

3. **Does the item reference an entity that was removed?**
   - If yes: block. The reference is invalid. User must either reinstate the entity in this amendment or drop the item.

4. **Does the item propose a field that a prior amendment already added?**
   - If yes: block. Don't add it twice. User must revise.

5. **Does the item use an `anchor_base_line:` number from the pre-prior-amendment base?**
   - Warn and replace with a semantic anchor or an effective-base line number.

## Resolving anchor line numbers

When a DIFF uses `anchor_base_line:` as a hint, the value must reference the **effective base** — the base spec after all prior amendments are applied in order.

Simple math: count insertions - removals in each prior amendment above the target line.

Example:
- Base spec: `events.produces[A]` at line 100.
- AMENDMENT-01 INSERT 15 lines at line 50 → `A` moves to line 115.
- AMENDMENT-02 REMOVE 8 lines at line 80 → `A` moves to line 107.
- AMENDMENT-03 INSERT 3 lines at line 120 → no shift (change is below A's line).

For AMENDMENT-04's anchor into `A`:
- Semantic: `anchor_event_id: "A"` (always correct).
- Line hint: `anchor_base_line: 107`.

If the skill cannot compute the effective line with confidence, it omits the line hint entirely and relies on the semantic anchor.

## Prereqs declaration

The header's `Prereqs:` line lists every prior amendment in apply order:

```
# Prereqs: AMENDMENT-01, AMENDMENT-02, AMENDMENT-03 already applied
```

If the user said "none":
```
# Prereqs: none
```

## Cross-amendment references

When the new amendment references an entity introduced by a prior amendment (e.g. it updates a note on an event declared in AMENDMENT-01), the DIFF must include `anchor_note:` documenting that cross-reference:

```yaml
- sub_op: UPDATE_FIELD_TEXT
  anchor_entry: "events.consumes[SOME_EVENT].note"
  anchor_note:  "SOME_EVENT was introduced by AMENDMENT-01 DIFF A. This update appends a bridge pointer."
  append: "Legacy transport: wire_key X — see entry below."
```

This saves the reviewer from wondering where `SOME_EVENT` came from.

## What to report to the user at the end of Phase 1

```
Phase 1 — Orientation report

Base spec:       <path>  (<N> lines)
Prior amendments applied in order:
  - AMENDMENT-01 (<path>) — <N> DIFFs — touched sections: <list>
  - AMENDMENT-02 (<path>) — <N> DIFFs — touched sections: <list>
  - AMENDMENT-03 (<path>) — <N> DIFFs — touched sections: <list>

Renames ledger:
  <list of name changes by kind>

Effective-anchor notes:
  - <section>: heavily modified by <prior amendment>; anchors should be semantic
  - <section>: untouched — raw base line numbers are still valid

New amendment will be: AMENDMENT-04 — Prereqs: AMENDMENT-01, -02, -03.
```

Present this before proceeding to Phase 2.
