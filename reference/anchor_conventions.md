# Anchor conventions — preferred → fallback

Anchors tell the applier *where* in the base spec a DIFF targets. Line numbers drift; semantic anchors don't. Always prefer semantic anchors.

## Priority order

Use the first applicable option for each DIFF.

### 1. Semantic relative anchors (preferred)

Reference the entry *adjacent* to where the change goes, by its identifying name.

```yaml
anchor_after_service:   "exotel-ivr-gateway"     # insert after this dependency
anchor_before_event_id: "CUSTOMER_SECURITY_FEE_PAID"  # insert before this event
anchor_after_field:     "call_enabled"           # insert after this derived field
anchor_after_assumption_id: "ASM-07"             # insert after this assumption
```

Why preferred: these names don't change when prior amendments apply. If someone inserts before your anchor, your anchor still resolves.

Pattern: `anchor_<after|before>_<entity_type>: "<name>"` where `<entity_type>` is whatever makes semantic sense in the spec (service, event_id, field, assumption_id, state, etc.).

### 2. Dotted path anchors (`anchor_entry:`)

Reference the target node by its path in the spec tree.

```yaml
anchor_entry: "data_model.execution_candidates.columns[extra_data].note"
anchor_entry: "events.produces[EVENT_X].consumers"
anchor_entry: "state_machine.states[STATE_NAME].description"
```

Why acceptable: also survives line drift; slightly less readable than a relative anchor but works when there's no natural "before / after" sibling.

### 3. Key-scoped anchors (`anchor_key:`)

Point to a field by its key within an already-scoped section.

```yaml
anchor_section: "state_machine"
anchor_key:     "initial_state"
```

Why acceptable: works when the change touches a well-known top-level field of a section.

### 4. Line numbers (`anchor_base_line:`) — LAST RESORT

```yaml
anchor_base_line: 2585
```

Why discouraged: the moment any prior amendment inserts or removes content above line 2585, this anchor is wrong. Use only when no semantic option exists, and always accompany with a semantic anchor too.

**Combined form (acceptable):**
```yaml
anchor_section:  "flows.call_flow.steps[2]"     # primary — survives drift
anchor_base_line: 2585                           # secondary — hint for the applier
```

The semantic anchor is authoritative; the line number is a human hint.

## Declaring drift risk

When any DIFF uses `anchor_base_line:` only (no semantic anchor), the amendment's END block must include a drift warning:

```
# Anchor line numbers, where used, are base-spec lines AT PRE-AMENDMENT-<N>
# APPLY TIME; verify before applying.
```

Include this warning even when semantic anchors are used — line numbers are only ever informational in this format.

## Resolving anchors under prior amendments

When computing line numbers for the `anchor_base_line:` hint, always resolve against the **effective** base — the base spec *after* prior amendments have been applied in order.

Example: base spec has `events.produces[A]` at line 100. AMENDMENT-01 inserts 15 lines before it. AMENDMENT-02 inserts 20 lines after it. For AMENDMENT-03's anchor into that entry:
- Semantic anchor: `anchor_event_id: "A"` — always resolves correctly.
- Line hint: `anchor_base_line: 115` (100 + 15 from AM-01; AM-02's insertion is *after*, so doesn't shift).

See `prior_amendments_handling.md` for the full computation.

## Common anchor patterns by entity type

| Entity kind in spec | Preferred anchor |
|---|---|
| Dependency block | `anchor_after_service: "<name>"` |
| Event entry | `anchor_after_event_id: "<ID>"` / `anchor_before_event_id: "<ID>"` |
| State block | `anchor_entry: "state_machine.states[<STATE>]"` |
| Flow step | `anchor_section: "flows.<flow>.steps[<n>]"` |
| Structural assumption | `anchor_after_assumption_id: "ASM-<NN>"` |
| Derived field | `anchor_after_field: "<name>"` |
| Consumer scalar in a list | `anchor_event_id` + `insert_after_item` (when using INSERT_LIST_ITEM) |
| Column note | `anchor_entry: "data_model.<table>.columns[<col>].note"` |

When the spec uses an entity type not listed here, invent the pattern using the same `anchor_<position>_<entity_type>` shape.
