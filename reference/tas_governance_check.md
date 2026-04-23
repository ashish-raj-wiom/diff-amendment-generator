# TAS governance check (Phase 5b)

Consumer-contract check, run against downstream aggregator / consumer governance. "TAS" is a generic label — it could be a task aggregation service, an event broker, a gateway, a downstream client library, an API consumer, or anything else that consumes from the base spec.

## Inputs

- The proposed set of DIFFs (output of Phase 4).
- User-supplied TAS / consumer-contract governance files (input #9).
- The base spec's identity-lock section if it exists (service_identity, service_id, api_prefix, etc.).

If input #9 is missing, skill skips this phase and flags the limitation. Unlike Phase 5a, there's no useful generic fallback — consumer contracts are inherently specific to the consumer.

## Checklist — run for every DIFF

Emit `PASS` or `[CONCERN-N]` per DIFF against each check.

### 1. Aggregation guards

**Check:** does the DIFF violate candidate-shape rules, cross-service key invariants, or aggregation preconditions?

Common consumer guards (names vary by domain):
- `GUARD-TAS-01` through `GUARD-TAS-NN` — candidate shape / field presence
- Cross-service-key consistency (e.g. `csp_id` matches across services)
- Candidate deduplication rules (e.g. one-candidate-per-authority_entity_id)
- Required field presence in aggregation responses

Scan proposed DIFFs for anything that:
- Removes a field the aggregator requires.
- Changes a field's type in a way the aggregator can't handle.
- Adds a candidate variant the aggregation rule would reject.
- Introduces a new cross-service key relationship the aggregator doesn't know about.

Common issues:
- New `event_id` not registered in the consumer's event catalogue → CONCERN; propose registering first or declaring as non-consumer event.
- Candidate payload loses a field that TAS aggregation reads → CONCERN; reject the removal or add a shim.

### 2. Consumer-contract schema

**Check:** do outbound event payloads match consumer expectations? Do inbound listeners match producer contracts?

For **outbound events** produced by this service:
- Every field in the payload is documented in the consumer's schema.
- Field types match. JSON naming matches (snake_case vs camelCase).
- Required/optional status matches.
- Version / schema_version field matches.

For **inbound events** consumed by this service:
- Listener-declared payload matches what the producer actually sends.
- Field names / types / required status match.

Common issues:
- Payload field added on producer side without consumer being told → CONCERN; coordinate.
- Field made required on producer side where consumer treated it as optional → CONCERN.
- JSON naming asymmetry that the consumer doesn't know about → CONCERN; document in consumer's contract file or fix the asymmetry.

### 3. Legacy-bridge classification

**Check:** are events correctly classified as OS-native vs legacy-bridge from the consumer's perspective?

A legacy-bridge event should:
- Be on a distinct transport (legacy exchange, separate bus, etc.)
- Be declared with `wire_key:` + `governance:` field
- Be absent from consumer's "first-class events" registry
- Have bidirectional note pointers to/from the canonical event

Risks this check catches:
- Legacy event declared as `event_id:` in producer spec → consumer's new-event-listener picks it up thinking it's OS-native.
- OS-native event declared with `wire_key:` → consumer's legacy-bridge handler picks it up, bypassing OS event registry rules.
- New `event_id:` with `channel:` field pointing at a legacy exchange → inconsistent classification.

Common resolution: unify the family's style (see `minimality_heuristics.md` rule 5 on reusing existing discriminators).

### 4. Identity-lock

**Check:** does the DIFF touch any identity-lock fields that consumers key on?

Typical identity-lock fields:
- `service_id` / `service_name` / `kubernetes_service_name` / `source_service_header`
- `api_prefix`
- `port`
- `database` name
- `package` name
- `prefix.event` / `prefix.parameter` / `prefix.metric`

If ANY DIFF changes these:
- CONCERN. These are load-bearing for consumers.
- Only acceptable change path: coordinate with all consumers + versioning + staged rollout. Single amendment generally insufficient.
- Propose splitting identity change into a separate, coordinated amendment.

## Per-DIFF reporting

Write the outcome into the DIFF's `governance_posture:`:

```yaml
governance_posture:
  state_machine:   "..."
  schema_change:   "..."
  new_events:      0
  os_check:        "PASS"
  tas_check:       "PASS"
  # OR:
  tas_check:       "CONCERN-2 (PENDING) — aggregation guard GUARD-TAS-03 affected"
```

## When Phase 5a and 5b find overlapping concerns

Sometimes the same underlying issue surfaces as both OS and TAS concerns. Example:
- OS: "this event should not be registered" (governance-by-absence).
- TAS: "consumers might subscribe thinking it's first-class" (consumer-side impact).

Emit a **single** `[CONCERN-N]` block and note both sides in its `Issue:` and `Where:` fields. Don't double-count.

## If input #9 (TAS governance files) is missing

Skip Phase 5b entirely. In the amendment header, write:

```
# TAS governance check: SKIPPED — user did not supply consumer-contract
# governance files. Consumer-side impact of this amendment has NOT been
# validated. Re-run with input #9 for coverage.
```

Do NOT fabricate a generic consumer-contract check. Unlike PII keywords or schema-additivity (which have universal patterns), consumer contracts are specific to the consumer; guessing generates noise, not signal.

## What this phase does NOT do

- Does not validate Phase 3's reality check findings against consumer repos. Unless the user supplies consumer code in input #4 (codebase pointer), Phase 3 only sees the primary service's code.
- Does not read the consumer's implementation. Only its declared contract / governance files.
- Does not propose consumer-side changes. It flags risks; consumer-side coordination is user's responsibility.
