# Cross-DIFF consistency check (Phase 4c)

Gate phase. After individual DIFFs are minimised (Phase 4a) and scenario-covered (Phase 4b), the skill runs one pass across the whole amendment to verify sibling DIFFs agree on shared rules.

Each DIFF in isolation can be internally correct and still contradict a sibling. This phase catches those contradictions before they ship to the tech team and produce "why do DIFF X and DIFF Y do this differently?" review comments.

## When this phase runs

After Phase 4b scenario coverage. Input: the full set of draft DIFFs (pre-write). Output: PASS, or one or more `[CONCERN-N]` entries representing cross-DIFF contradictions.

## Consistency rules — the sweep

Run every rule across every pair (or family) of related DIFFs. Fail fast on contradictions.

### Rule CC-1 — Shared resolution strategy

If two or more DIFFs perform the same lookup (e.g. resolve `customer_id → candidate` via `getConnectionsByCustomer`), they must apply the same resolution rule when multiple rows match.

Examples of contradictions to catch:
- DIFF J-9: "filter to `PENDING_INSTALL`, take first"
- DIFF J-11: "for each matching candidate, invoke…"

→ CONCERN. Decide the rule once ("first" / "each" / "most recent" / "reject-on-multi") and apply uniformly, or document why they differ.

### Rule CC-2 — Shared naming conventions

When multiple DIFFs introduce names in the same namespace (routing keys, queue names, event_ids, wire_keys, table names, column names, env var names), they must follow the same pattern.

Examples:
- DIFF J outbound uses routing key `{env}.booking.installation.success`
- DIFF J inbound uses routing key `booking.security.payment.success` (no `{env}.` prefix)

→ CONCERN. Either the legacy system genuinely doesn't prefix (document in the bridge's `legacy_system_quirk:` field) or a human picked the wrong one.

### Rule CC-3 — Shared ID type at boundary crossings

If multiple DIFFs reference the same logical identifier (e.g. `customer_id` / `booking_id` / `candidate_id`), the declared type must be the same everywhere it crosses a process boundary.

Examples:
- DIFF J payload: `booking_id: Long`
- DIFF F resolver: `Long.parseLong(customerId)` where `customerId: String` from CL
- DIFF G DTO: `booking_id: Long`

→ Consistent internally, but Rule CC-3 requires flagging the load-bearing `Long.parseLong(String)` cast so it's traced back to a source-of-truth declaration ("`connections.customer_id` is numeric-stringified"). If not locked, CONCERN.

### Rule CC-4 — Transport vs registration alignment

If one DIFF registers an event in a producer registry that keys on transport (e.g. HTTP routing publisher consulting `registry-service.producers[]`), every other DIFF's claim about that event's transport must match.

Examples:
- DIFF D: adds `BOOKING_ROHIT_ARRIVED` to `registry-service.producers[]`
- DIFF J: says `BOOKING_ROHIT_ARRIVED` transport is `BookingInstallationRmqDestination` (not HTTP)

→ CONCERN unless the registry uses a transport-indicator field (or the publisher has a filter). The contradiction: if `HttpRoutingPublisher` consults `producers[]`, it'll try to resolve an HTTP route that doesn't exist. Options: (a) add a `transport:` marker to the registry entry, (b) omit BOOKING_* from the HTTP-keyed registry, (c) add a filter in the publisher that skips non-HTTP transports.

### Rule CC-5 — Actor consistency on audit rows

If multiple DIFFs write to the same audit / state-transition log, their actor fields must agree on typing.

Examples:
- DIFF F: writes `state_transition_log` row with `actor_csp_id = actorId` (a human installer) AND `actor_type = SYSTEM` (because it's a synthetic fire).

→ CONCERN. Actor should be one OR the other: `SYSTEM` with null/system actor_id, or `USER` with the human actor_id. Mixing is a schema smell.

### Rule CC-6 — Matching trigger → emit cardinality

If a DIFF describes a listener that triggers an emit, and another DIFF describes the emitted event, the two must agree on how often the emit fires per listener invocation.

Examples:
- DIFF F branch (b): "set flag, do not transition." No emit specified.
- DIFF G sub_op E-5: FCM fires on `CustomerSecurityFeePaid` arrival.
- Later DIFF K says synthetic transition at AADHAR end also fires FCM.

→ CONCERN unless one-push-vs-two is explicitly decided. The envelope has fields implying one push; the transitions imply two.

### Rule CC-7 — Legacy-bridge vs canonical classification consistency

If multiple DIFFs introduce events in the same family (e.g. all `BOOKING_*` wire_keys), they must use the same classification style uniformly.

Examples:
- DIFF J-1 through J-6: use `wire_key:` + `governance: "legacy-bridge…"`
- DIFF F-7: uses `event_id:` + `channel: "BookingInstallationRmqDestination"`

→ CONCERN. Pick one style for the family and apply to all members.

### Rule CC-8 — Idempotency-key consistency

If multiple DIFFs describe inbound listeners, they should dedupe consistently unless there's a documented reason to differ.

Examples:
- `BookingSlotConfirmedListener`: dedupes on AMQP `messageId`
- `SuccessfulPaymentEventListener`: dedupes on AMQP `messageId`

→ PASS.

But:
- Listener A: dedupes on AMQP `messageId`
- Listener B: dedupes on `(booking_id, timestamp)` composite

→ CONCERN. Pick one dedup model per listener family.

### Rule CC-9 — Deploy-order preconditions stated once

If a DIFF's code path depends on another service's endpoint that lives in another DIFF (or in the Coordination section of the header), the dependency must be declared in one place, not contradicted.

Examples:
- Header Coordination: "CL must ship `getConnectionDetail` before TAS deploys"
- DIFF A dependency note: (silent) — should reference the header's coordination statement or restate it

→ Usually PASS (header is the canonical place), but if the DIFF explicitly contradicts the header (e.g. header says "CL before TAS", DIFF note says "TAS can deploy independently") → CONCERN.

### Rule CC-10 — Type nullability consistency

If multiple DIFFs introduce fields with matching semantics (e.g. two migrations adding `*_at` timestamp columns), their nullability and defaults should agree unless there's a reason to differ.

Examples:
- DIFF F V045: `pre_paid_bypass_applied BOOLEAN NOT NULL DEFAULT FALSE`
- DIFF G V046 device_fcm_tokens: `registered_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`

→ Different columns, different semantics — PASS. But two new `*_status` columns with different nullability → flag for review.

## Running the sweep

The skill produces a cross-DIFF report:

```
Cross-DIFF consistency report — AMENDMENT-04

Rule CC-1 (shared resolution strategy):     ⚠ CONCERN-3 — J-9 says "first", J-11 says "each"
Rule CC-2 (naming conventions):             ⚠ CONCERN-4 — routing key prefix inconsistent
Rule CC-3 (ID type at boundaries):          ⚠ CONCERN-2 — Long.parseLong(UUID) unmarked
Rule CC-4 (transport vs registration):      ⚠ CONCERN-5 — BOOKING_* in HTTP producers[]
Rule CC-5 (actor consistency):              ⚠ CONCERN-6 — actorId (human) + ActorType.SYSTEM
Rule CC-6 (emit cardinality):               ⚠ CONCERN-7 — FCM push once-or-twice on bypass
Rule CC-7 (legacy-bridge classification):   ✓ PASS
Rule CC-8 (idempotency-key consistency):    ✓ PASS
Rule CC-9 (deploy-order preconditions):     ✓ PASS
Rule CC-10 (type nullability):              ✓ PASS

6 concerns to resolve before drafting.
```

Each concern lands in the amendment header's `CONCERN STATUS:` block as `PENDING`. Drafting blocks until all are `RESOLVED`.

## Presenting to the user

Same pattern as scenario coverage — batch the concerns, show options per concern, await user decisions.

> Cross-DIFF consistency — 6 concerns:
>
> **[CONCERN-3]** Multi-candidate resolution: J-9 says "first", J-11 says "each."
>   Options: (a) both take first; (b) both iterate each; (c) keep asymmetric, document why.
>   Recommendation: (a) — simpler, matches the common case.
>
> **[CONCERN-4]** Routing-key prefix: BOOKING_* uses `{env}.`, SUCCESSFUL_PAYMENT_EVENT doesn't.
>   Options: (a) normalise both to `{env}.` (requires legacy producer alignment); (b) document the asymmetry as `legacy_system_quirk:`; (c) re-check with tech — maybe the legacy producer is wrong.
>   Recommendation: (b) — we don't control the legacy producer, document and move on.
>
> … (4 more) …

## Why this phase exists

Single-DIFF review catches single-DIFF problems. No amount of per-DIFF governance catches contradictions *between* DIFFs — those are emergent properties of the whole amendment. Phase 4c is the only layer in the skill that treats the amendment as a single coherent artefact rather than a bag of DIFFs.

Every tech-review contradiction call-out we've seen historically would have been caught here.
