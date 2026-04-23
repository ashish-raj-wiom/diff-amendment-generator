# Sub_op vocabulary — closed set

This is the complete list of operations the skill uses. Do not invent new ones unless strictly symmetric to an existing one (and declare the extension in the header).

## Top-level operations

Used as the value of `operation:` on a DIFF.

### `INSERT`
Insert a new structured entry (block with fields) into a list.

```yaml
diff_A_add_new_dependency:
  anchor_section: "dependencies"
  anchor_after_service: "existing-service-name"
  operation: INSERT
  new_entry:
    - service: new-service-name
      type: data-provider
      direction: upstream
      criticality: high
      timing: "..."
```

When to use: one new item going into a list of structured objects. If the list holds scalars, use `INSERT_LIST_ITEM` instead.

### `INSERT_LIST_ITEM`
Insert a scalar (or simple short-form item) into a list.

```yaml
diff_B_add_consumer:
  anchor_section: "events.produces[EVENT_X].consumers"
  anchor_event_id: "EVENT_X"
  operation: INSERT_LIST_ITEM
  insert_after_item: "existing-consumer"
  new_items:
    - "new-consumer (one-line description)"
```

When to use: the list holds scalar strings or one-line values.

### `MULTI`
Wrap multiple sub_ops under one anchor_section. Preferred when the same slice of the spec needs several related changes.

```yaml
diff_C_multi_example:
  anchor_section: "some_section"
  operation: MULTI
  changes:
    - sub_op: UPDATE_FIELD
      anchor_entry: "some_section.field_a"
      old_value: "..."
      new_value: "..."
    - sub_op: REMOVE_BLOCK
      anchor_entry: "some_section.entries[x]"
      remove_span: "lines A–B"
```

When to use: two or more related changes in the same section. Collapse them here rather than spawning parallel DIFFs.

### `RENAME_ALL`
Consistent rename of a name across multiple sites.

```yaml
diff_D_rename_old_to_new:
  operation: RENAME_ALL
  renames:
    - kind: state_name
      from: "OLD_STATE"
      to:   "NEW_STATE"
      occurrences:
        - "state_machine.states[OLD_STATE]"
        - "state_machine.terminal_states"
        - "flows.terminal_entry_flow.steps[4].action"
    - kind: db_column
      from: "old_col"
      to:   "new_col"
      migration: "ALTER TABLE ... RENAME COLUMN old_col TO new_col"
```

When to use: same rename applies to many locations. List all occurrences so reviewers can verify coverage.

### `REFERENCE_ONLY`
The DIFF points at an external artefact; no inline content.

```yaml
diff_E_governance_interpretation:
  operation: REFERENCE_ONLY
  artefact_path: "docs/governance/INTERPRETATION.md"
  rule: "<one-sentence restatement of the rule>"
  status: DEPLOYED
  governance_posture:
    prd_text_change: NONE
    state_machine_change: NONE
```

When to use: interpretation / governance note that already exists elsewhere. Don't duplicate the artefact inline.

## Sub_ops (used inside `MULTI` → `changes:`)

Each sub_op appears as a list item under `changes:`. Required fields vary by sub_op.

### `UPDATE_FIELD`
Swap a scalar / enum value.

```yaml
- sub_op: UPDATE_FIELD
  anchor_entry: "path.to.field"
  old_value: '"OLD_VALUE"'
  new_value: '"NEW_VALUE"'
```

### `UPDATE_FIELD_TEXT`
Edit prose. Two forms:

**Full replacement:**
```yaml
- sub_op: UPDATE_FIELD_TEXT
  anchor_entry: "path.to.note"
  old_text: "original prose here"
  new_text: "replacement prose here"
```

**Append:**
```yaml
- sub_op: UPDATE_FIELD_TEXT
  anchor_entry: "path.to.note"
  append: "additional sentence appended to the existing text."
```

Prefer `append:` when the change is additive — smaller diff to review.

### `UPDATE_LIST`
Swap the items of a list.

```yaml
- sub_op: UPDATE_LIST
  anchor_entry: "path.to.list"
  old_items:
    - "item A"
    - "item B"
  new_items:
    - "item A"
    - "item C"
```

### `UPDATE_FLOW_STEP`
Rewrite a numbered step's action inside a flow.

```yaml
- sub_op: UPDATE_FLOW_STEP
  anchor_section: "flows.my_flow.steps[2]"
  old_action: "original step action text"
  new_action: "replacement step action text"
```

### `INSERT_BLOCK`
Add a structured block inside a MULTI. Symmetric to `REMOVE_BLOCK`.

```yaml
- sub_op: INSERT_BLOCK
  anchor_section: "dependencies"
  insert_after_service: "some-existing-service"
  new_entry:
    - service: new-service
      type: data-provider
      ...
```

When to use: complex structured insert as part of a MULTI. If it's the only op, use top-level `INSERT` instead.

### `INSERT_LIST_ITEM` (as sub_op)
Same shape as the top-level form, but inside a MULTI.

### `REMOVE_BLOCK`
Delete a structured block.

```yaml
- sub_op: REMOVE_BLOCK
  anchor_entry: "states[SOME_STATE]"
  remove_span: "lines 312–332 (entire state entry)"
```

### `REMOVE_LIST_ITEMS`
Delete scalar items from a list.

```yaml
- sub_op: REMOVE_LIST_ITEMS
  anchor_key: "terminal_states"
  remove_items:
    - '"SOME_STATE"'
    - '"ANOTHER_STATE"'
```

### `REMOVE_FIELD`
Delete a single field from a block.

```yaml
- sub_op: REMOVE_FIELD
  anchor_entry: "events.produces[EVENT_X].payload"
  remove_field:
    - field: deprecated_field
      type: timestamptz
```

### `REMOVE_FLOW_STEP`
Delete a numbered step from a flow.

```yaml
- sub_op: REMOVE_FLOW_STEP
  anchor_section: "flows.my_flow.steps[9]"
  remove_step: "step 9 — <description of what was there>"
```

### `REMOVE_FLOW`
Delete an entire flow.

```yaml
- sub_op: REMOVE_FLOW
  anchor_section: "flows.obsolete_flow"
  remove_span: "lines A–B (entire flow)"
```

## Extension rule

New sub_ops are only acceptable if they are strictly symmetric to an existing one (example: `INSERT_BLOCK` is symmetric to `REMOVE_BLOCK`). Document the extension in the amendment header's `Format:` line.

If you find yourself wanting a new op that isn't symmetric, reconsider — the existing vocabulary probably covers the intent.
