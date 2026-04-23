# OS governance check (Phase 5a)

Primary governance check, run against the base spec's own invariants. "OS" is a generic label for the domain's governance layer — it could be an OS PRD, a service governance doc, an API style guide, whatever the user supplied as input #8.

## Inputs

- The proposed set of DIFFs (output of Phase 4).
- User-supplied OS / base-spec governance files (input #8).
- Base spec (input #1) — often contains its own invariants in `structural_assumptions:`, `guard_specifications:`, `os_refs:`, `pii_fields:`, etc.

If input #8 is missing, the skill falls back to a generic check (see "Generic fallback" at bottom) and flags the limitation in the amendment header.

## Checklist — run for every DIFF

Emit `PASS` or `[CONCERN-N]` per DIFF against each check.

### 1. PII invariants

**Check:** does any DIFF persist data that governance says must not be persisted?

Scan for PII leaks in:
- New columns (direct persistence)
- URL path segments (indirect persistence — URL stored in JSONB = data persisted)
- JSONB / freeform keys (same)
- Payload fields on outbound events (transport = exposure)
- Log lines that would log PII fields
- Error messages that would leak identifying info

Common governance patterns to match:
- "customer mobile NEVER stored"
- "SSN/PII must not appear in logs"
- "Role-Appropriate Visibility" / P8-style rules
- "<role> never sees <field>"

If ANY DIFF violates: CONCERN, typically 3 options — swap identifier, add structural assumption accepting the exception, or reject the change.

### 2. State-machine invariants

**Check:** does any DIFF add transitions, states, rename edges, or introduce new trigger paths for an existing edge?

Scan for:
- New states introduced
- New transitions (edge from state A to state B)
- Rename of an existing state (redirect to the RENAME_ALL path in Phase 6)
- New trigger paths for an existing edge that aren't documented in `upstream_signal_mapping:` or equivalent

Common issues:
- Synthetic edge fires (same edge, new trigger source not in signal mapping) → propose a new structural assumption
- New state that could be expressed as a condition on an existing state → Rule 10 (structural-assumption > new state)
- Edge with aspirational note (`note: "only X"`) but machine-readable `condition:` is broader → propose to tighten the condition

### 3. Event-registry governance

**Check:** where does each new event entry go?

- Is the new event OS-native (first-class, canonical)? → must be in both `events.produces:` (or `events.consumes:`) AND `registry_event_mapping:`.
- Is it a legacy-bridge transport? → must use `wire_key:` (not `event_id:`), must carry a `governance:` field, must be absent from `registry_event_mapping:`.
- Is it both (parallel emit)? → canonical gets event_id + registry membership; bridge gets wire_key + governance flag + absent-from-registry. Bidirectional note pointers per Rule 11.

Common issues:
- Naively adding a legacy event as `event_id:` → CONCERN; reclassify as `wire_key:`.
- Canonical event with `channel:` field suggesting non-OS transport → CONCERN.
- Registry entry added for a legacy bridge → CONCERN; remove from registry.

### 4. Guard / structural-assumption coverage

**Check:** do the DIFFs exercise guard conditions correctly? Are there aspirational notes (`note:` fields saying what should happen) without machine-readable enforcement?

- Does the change hit a guard whose `condition:` is weaker than its `note:` intent? → CONCERN; propose tightening the condition.
- Does the change create an exception to an existing invariant without a new ASM entry? → CONCERN; propose a new `ASM-<NN>`.
- Is the new structural assumption numbered correctly (next free integer)?

### 4b. Policy-vs-enforcement gaps (widened)

**Check:** any `MUST` / `SHOULD` / "never" / "always" / invariant-style language in the DIFF — is there a concrete runtime mechanism that enforces it?

This is a superset of the aspirational-note check above. That check catches the specific case of guard-note-vs-guard-condition mismatch. This wider check catches policy statements that have **no enforcement mechanism at all**, not just a weaker one.

Examples of what to flag:
- `"Customer app is the sole visible OTP surface; partner app MUST NOT render the OTP."` — what code prevents the partner app from rendering? If the only answer is "the partner app developers know not to" or "it's a policy, not a constraint" → CONCERN.
- `"Installer types the OTP into the partner app in person at the premises."` — what enforces "in person"? Physical presence verification? GPS? Or is it purely trust-based? If trust-based, the security model depends on policy not mechanism — flag.
- `"Partner-app verify MUST respect the customer-app OTP TTL."` — where does the TTL come from? Is it synchronised? What if they drift? If unanswered → CONCERN.
- `"Customer-app UI enforces that payment is only enabled after ARRIVED_AT_SITE."` — this is a cross-surface enforcement claim. Does the server defensively reject payments for pre-arrival candidates, or does it trust the UI? (Both are valid; one must be chosen explicitly.)

For each policy statement, ask the author to classify:
1. **Enforced by code** — which function / class / guard? (Name it.)
2. **Enforced by schema** — which constraint? (Name it.)
3. **Enforced by upstream** — which other service enforces, and is the trust documented?
4. **Enforced by human process** — acknowledged as trust-based; any detection-after-violation mechanism (audit log, anomaly alert)?

If the answer is (3) or (4), the policy statement stays but a structural assumption captures the trust boundary explicitly. Implementers then know not to add redundant checks, and auditors know where the trust is placed.

### 5. Schema governance

**Check:** are migrations additive only?

- `ADD COLUMN ... NOT NULL DEFAULT <safe>` → ✅ additive.
- `CREATE TABLE ...` → ✅ additive.
- `ALTER TABLE ... RENAME COLUMN ...` → CONCERN. Renames break running code.
- `ALTER TABLE ... DROP COLUMN ...` → CONCERN. Blocks rollbacks and in-flight readers.
- `ALTER TABLE ... ALTER COLUMN ... TYPE ...` → CONCERN. Type changes are not additive.

If the change request genuinely needs a non-additive change, propose splitting across multiple amendments (add new column → migrate readers → drop old column, across 2–3 releases).

### 6. Naming governance

**Check:** do new identifiers match declared conventions?

- Casing: snake_case / camelCase / PascalCase consistent with neighbouring entries.
- Prefixes: `ES_*`, `SVC_*`, `LEGACY_*`, or whatever the spec uses.
- Legacy contract freezes: some field names are intentionally unchanged (e.g. typos preserved because consumers parse them). Respect these.
- Event_id format: `<PREFIX>_<VERB>_<OBJECT>` or whatever the spec follows.

Common issues:
- New event uses `camelCase` when the spec has `UPPER_SNAKE` → CONCERN.
- Column uses different casing than neighbours → CONCERN.
- Prefix mismatches the domain (e.g. `ORDER_PAID` in a spec whose prefix is `PAYMENT_*`) → CONCERN.

## Per-DIFF reporting

Write the outcome of all six checks into the DIFF's `governance_posture:`:

```yaml
governance_posture:
  state_machine:   "NO CHANGE"
  schema_change:   "NONE"
  new_events:      0
  new_guards:      0
  os_check:        "PASS"
  # OR, if concerns raised:
  os_check:        "CONCERN-1 (PENDING), CONCERN-3 (RESOLVED via b)"
  tas_check:       "..."
```

## Generic fallback when OS governance files not supplied

If input #8 missing, run a trimmed default:

1. **PII check:** scan for likely PII field names in new columns / URLs / payloads (`email`, `phone`, `mobile`, `ssn`, `aadhar`, `pan`, `dob`, `address`, `pincode`, etc.). Flag any that appear in persistence paths.
2. **Schema additivity:** enforce additive-only migrations regardless of user rules.
3. **Naming consistency:** enforce "matches neighbouring entries' casing / prefix."

Note in amendment header:
```
# OS governance check: fallback mode — user did not supply governance files.
# Only PII-keyword / schema-additivity / naming-consistency checks applied.
# For full invariant coverage, re-run with input #8 supplied.
```

## What this phase does NOT do

- **Does not re-run the reality check.** Phase 3 already did that; Phase 5a trusts Phase 3's findings.
- **Does not validate consumer contracts.** That's Phase 5b (TAS governance check).
- **Does not make the drafting decision.** It surfaces concerns; user decides; draft follows.
