# Minimality heuristics — 12 rules

Applied in Phase 4. For each proposed DIFF, challenge it against these rules in order. Anything that fails a rule gets folded, collapsed, or dropped.

## Rule 1 — Fold related changes into `MULTI`

**Bad (3 separate DIFFs touching the same section):**
```yaml
diff_A_update_field_x: { operation: UPDATE_FIELD, ... }
diff_B_update_field_y: { operation: UPDATE_FIELD, ... }
diff_C_remove_block_z: { operation: MULTI, changes: [REMOVE_BLOCK] }
```

**Good (one MULTI):**
```yaml
diff_A_section_cleanup:
  anchor_section: "the_section"
  operation: MULTI
  changes:
    - sub_op: UPDATE_FIELD
      ...
    - sub_op: UPDATE_FIELD
      ...
    - sub_op: REMOVE_BLOCK
      ...
```

One anchor, one governance posture, one review unit. Applies when the changes are logically one slice.

## Rule 2 — Prefer semantic anchors over line numbers

**Bad:**
```yaml
anchor_base_line: 189
```

**Good:**
```yaml
anchor_after_service: "existing-service-name"
anchor_base_line: 189   # optional hint — semantic is authoritative
```

Line numbers drift with prior amendments. See `anchor_conventions.md`.

## Rule 3 — Use append over full replace

**Bad (full replace for an additive change):**
```yaml
- sub_op: UPDATE_FIELD_TEXT
  anchor_entry: "path.to.note"
  old_text: "Existing note with several sentences. It has multiple clauses. It is long."
  new_text: "Existing note with several sentences. It has multiple clauses. It is long. Here is the additional sentence."
```

**Good:**
```yaml
- sub_op: UPDATE_FIELD_TEXT
  anchor_entry: "path.to.note"
  append: "Here is the additional sentence."
```

Smaller diff to review; less risk of typo in the replicated old_text.

## Rule 4 — Absence is a governance statement

If the governance says "X is not tracked" / "not persisted" / "not in registry," then the invariant is that X's **absence** is the contract. Don't add a flag or field to assert the absence — that defeats the point.

**Bad:**
```yaml
- wire_key: "LEGACY_EVENT"
  is_not_os_native: true    # redundant
  not_in_registry: true      # redundant
```

**Good:**
```yaml
- wire_key: "LEGACY_EVENT"
  governance: "legacy-bridge — NOT in OS event registry; NOT an event_id"
```

The single `governance:` line says what the absence from `registry_event_mapping` already says structurally. And the fact that the entry uses `wire_key:` instead of `event_id:` is itself the discriminator.

## Rule 5 — Reuse existing discriminators

If the spec already distinguishes category A from category B via an existing field, use that same field.

**Bad (adding parallel discriminator):**
```yaml
- event_id: "LEGACY_X"
  is_bridge: true         # new parallel flag nobody else uses
```

**Good (using existing convention):**
```yaml
- wire_key: "LEGACY_X"    # the existing convention: wire_key vs event_id
  channel: "LegacyBus"
```

## Rule 6 — Don't add top-level sections when existing ones fit

**Bad:**
```yaml
# Base spec already has: events.produces, events.consumes
# Proposed amendment adds new top-level section: legacy_bridges_produces
legacy_bridges_produces:   # new top-level section
  - wire_key: ...
```

**Good:**
```yaml
# Amendment inserts INSIDE events.produces with a wire_key discriminator
- sub_op: INSERT_BLOCK
  anchor_section: "events.produces"
  new_entry:
    - wire_key: "LEGACY_X"
      governance: "legacy-bridge — ..."
```

Nesting inside existing sections preserves the base spec's table of contents.

## Rule 7 — Additive-only schema changes

**Bad (renames, drops):**
```yaml
migration: "ALTER TABLE users RENAME COLUMN email TO email_address"
migration: "ALTER TABLE users DROP COLUMN legacy_field"
```

**Good (additive only):**
```yaml
migration: "ALTER TABLE users ADD COLUMN new_field BOOLEAN NOT NULL DEFAULT FALSE"
migration: "CREATE TABLE user_preferences (...)"
```

Amendments apply to live systems. Additive migrations are rollback-safe and don't require coordinated app-deploy cycles.

## Rule 8 — One concept per DIFF

**Bad (unrelated changes in one DIFF):**
```yaml
diff_A_misc:
  operation: MULTI
  changes:
    - sub_op: UPDATE_FIELD    # update auth timeout
    - sub_op: INSERT_BLOCK    # add new event
    - sub_op: REMOVE_FLOW     # remove deprecated flow
```

**Good (one concept per DIFF):**
```yaml
diff_A_auth_timeout_update: { ... }
diff_B_new_event:            { ... }
diff_C_remove_deprecated_flow: { ... }
```

Unrelated changes → separate DIFFs. But related changes across sections can legitimately MULTI together (see Rule 1).

## Rule 9 — Reference, don't duplicate

**Bad:**
```yaml
diff_E_governance_interpretation:
  operation: MULTI
  changes:
    - sub_op: UPDATE_FIELD_TEXT
      # inlines the entire content of an external governance doc
      new_text: "<500 lines of governance prose>"
```

**Good:**
```yaml
diff_E_governance_interpretation:
  operation: REFERENCE_ONLY
  artefact_path: "docs/governance/B-CL-1_interpretation.md"
  rule: "<one-line restatement of the rule>"
  status: DEPLOYED
```

If the content already lives elsewhere, point at it.

## Rule 10 — Structural-assumption > new state / event

When a corner case doesn't fit the existing state machine, ask: can I describe it as an exception in `structural_assumptions:` instead of growing the state machine?

**Bad:**
```yaml
# Adds a whole new state just to describe a minor timing condition
- state: "WAITING_FOR_FLAG_B"
  ...
```

**Good:**
```yaml
- id: "ASM-<NN>"
  assumption: "Under condition B, the FLAG-transition may fire deferred..."
  code_impact: "..."
```

Assumptions are cheap governance; new states cost schema + UI + ops.

## Rule 11 — Bidirectional pointers where pairs exist

If entry A references entry B, B must reference A.

**Bad (one-way pointer):**
```yaml
# In the legacy-bridge entry:
- wire_key: "LEGACY_X"
  governance: "bridge for canonical OS event EVENT_X"

# In the canonical EVENT_X entry: no mention of the bridge
- event_id: "EVENT_X"
  note: "..."
```

**Good (bidirectional):**
```yaml
# Bridge entry points at canonical:
- wire_key: "LEGACY_X"
  governance: "bridge for canonical OS event EVENT_X"

# Canonical entry points back at bridge:
- event_id: "EVENT_X"
  note: "... Parallel legacy-bridge emit LEGACY_X — see wire_key entry."
```

Otherwise a reader landing on one entry misses the other.

## Rule 12 — Challenge every nice-to-have

For every proposed change, ask: "Does a specific change-request item or governance rule require this?" If not, cut it.

**Examples of nice-to-haves to cut:**
- Renaming a field "for consistency" when the rename isn't requested.
- Adding a governance comment "to document intent" when the intent is obvious from the change itself.
- Inserting a new structural assumption "to future-proof" when no one asked for future-proofing.
- Touching a note field to fix a minor grammar issue while you're in the neighbourhood.

### 12a — Forward-compat clauses without current users

Specific sub-rule: any clause that says "tolerate unknown X" / "handle future Y" / "accept version N+1" **where no such X, Y, or version N+1 exists today** is a nice-to-have. Cut it, unless:
- A concrete second value is imminent (planned, in-flight, has a ticket), OR
- Interface is explicitly extensible-by-design (e.g. plugin architectures where unknown values are expected).

**Examples to cut:**
- `partner_app_contract: "Unknown event enum → ignore and re-poll"` when the enum has exactly one value.
- `schema_version_handling: "Reader tolerates schema_version N+1 additive fields"` when there's no plan to bump schema.
- `note: "Future bridge variants may add field X"` when there are no planned variants.

Forward-compat clauses without real users waste reviewer attention. If a reviewer has to ask "what's the second enum value?" and the answer is "there isn't one yet" — cut it. Add the clause back when the real X exists.

**Why this matters:** forward-compat text invites implementer scope creep. An implementer seeing "tolerate unknown enum" will write defensive parsing code, error branches, and tests for hypothetical values. Zero of that is needed today; all of it has to be maintained.

---

Speculative additions inflate review surface without earning reviewer trust. When a reviewer asks "why is this here?" and the answer is "I was nearby and thought I'd..." — cut it.

## Applying the rules

Run them in order. Rule 1 merges the most stuff; Rule 12 cuts the rest. After Phase 4 the DIFF count should be at or below the count of distinct change-request items (not proportional to lines changed).
