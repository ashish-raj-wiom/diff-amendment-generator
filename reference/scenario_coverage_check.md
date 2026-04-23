# Scenario coverage check (Phase 4b)

Gate phase. For every DIFF, the skill forces explicit answers to a fixed set of scenario questions. Unanswered questions become `[CONCERN-N]` blocks and block drafting until resolved.

The goal: eliminate the class of tech-review feedback where the implementer asks *"what happens when…?"* and the spec is silent.

## When this phase runs

After Phase 4a minimality pass. Runs per DIFF. A single DIFF that adds one migration, one external call, one new state-touching step may generate 10–15 scenario questions before any are answered.

## Triggers — what attracts scenario questions

Classify every element touched by a DIFF into one of these categories and apply the matching checklist.

### A. External call (invokes a service outside this process)

Applies whenever a DIFF touches: HTTP client, RMQ listener/publisher, FCM / push, Exotel-style telephony gateway, database link to another service, any SDK call that crosses a process boundary.

Required answers:
1. **Failure:** on 4xx / 5xx / timeout / null / exception — what happens? (log? retry? fail the enclosing txn? fall back to a default? emit a different event?)
2. **Timeout:** what's the request timeout in ms? If inherited from a default, say so and point at the default.
3. **Retry:** single retry? exponential backoff? count? jitter? none (fire-and-forget)?
4. **Circuit-break:** any threshold at which we stop calling this endpoint temporarily?
5. **Idempotency:** is the remote side idempotent? If not, what prevents double-execution on retry?
6. **Observability:** what metric / log / trace records the outcome?
7. **PII posture:** does the call payload contain anything classified PII? (If yes, cross-link to Phase 5a.)
8. **Response schema:** what fields does the response contain? What if any are missing / null / extra?

### B. New mutable field (column, flag, token, counter, entry)

Applies when a DIFF adds a column via migration, adds a JPA/ORM entity field, adds a new JSONB key, adds a new row type (token table, etc.).

Required answers:
1. **set_by:** which code path writes this field? Under what conditions?
2. **read_by:** which code paths read it? What do they do with it?
3. **reset_by:** does it ever get reset? When? If "never reset," state that explicitly in the DIFF note.
4. **unique_on:** if it's on a table with natural keys, what's the uniqueness constraint? What's the upsert key?
5. **concurrent-write:** can two writers hit this row simultaneously? What's the isolation strategy?
6. **orphan_policy:** if the field references an external resource (S3 object, token, etc.), what happens when the parent row is deleted or the external resource is re-uploaded? Is there a cleanup job?
7. **rehydration:** if this is a nullable / boxed type, what happens when existing rows rehydrate to null? NPE risk?
8. **default:** if the migration adds the column, what's the NOT NULL DEFAULT? Is that default semantically meaningful for existing rows, or is it a placeholder?

### C. State-touching step (fires a state transition, synthesises an edge, emits a state-change event)

Applies when a DIFF touches: state machine transitions, synthetic fires (like a deferred fire inside another handler), new trigger paths for existing edges, new terminal states.

Required answers:
1. **Trigger enumeration:** list every trigger that can cause this transition. (If a trigger is being added silently, flag it.)
2. **Concurrency:** what if two triggers fire concurrently for the same candidate? Locking strategy? Optimistic version check?
3. **Race window:** is there a window where state has been read but not yet committed, and another path reads it stale? Describe the interleaving and the outcome.
4. **Audit trail:** how many state-transition-log rows does a single user-visible transition produce? (For synthetic fires, this is usually 2 — the "real" one and the synthetic one. Specify.)
5. **Attention / notification side effects:** does the transition fire any attention bump, FCM push, analytics event? If yes, same-txn or after-commit?
6. **Retry semantics:** if the transition handler fails partway through, is the transition rolled back? What about side effects (push, emit)?
7. **Idempotency on replay:** if the same trigger event is redelivered, is the second call a safe no-op or does it re-fire?

### D. Inbound event listener (RMQ consumer, HTTP webhook, polling loop)

Applies when a DIFF adds or modifies a consumer that receives external events.

Required answers:
1. **Deduplication key:** AMQP messageId? Business key? Both?
2. **Deduplication store:** in-memory? DB table? TTL? How does a DLQ replay after TTL expires behave?
3. **Unknown event type:** what if the envelope carries an unexpected `key` / `type`? (ack + log + ignore? DLQ? nack?)
4. **Malformed payload:** what if the JSON doesn't parse? What if required fields are missing?
5. **Resolution failure:** if the consumer resolves the event to a business entity (booking_id → candidate), what if resolution returns empty / multiple / non-matching-state?
6. **Duplicate business receipt:** if the same logical event arrives with different dedup keys (e.g. legacy system retries at their end), what prevents double-processing?
7. **Handler exception:** nack/requeue? nack/drop? nack with timeout?

### E. Outbound event publisher (emits to bus, after-commit hook)

Applies when a DIFF adds a publisher or extends an existing listener to publish additional wire_keys.

Required answers:
1. **Trigger:** what triggers the publish? Same-txn or AFTER_COMMIT?
2. **Publish failure:** roll back the business txn? (Usually no — fire-and-log.) If roll back, say so explicitly.
3. **Payload resolution failure:** if any payload field can't be resolved at publish time, skip-with-WARN or fail-with-ERROR?
4. **Duplicate publish:** can the same transition trigger the publisher twice? What idempotency does the consumer have?
5. **Sequencing:** must this publish happen before another one? Document the order.

### F. Policy / invariant statement

Applies when a DIFF's note, description, or structural_assumption contains `MUST` / `SHOULD` / "never" / "always" / "only" language.

Required answers:
1. **Enforcement mechanism:** what code / schema / UI constraint enforces this at runtime?
2. **Violation detection:** if the policy is violated, how is the violation detected? Test? Audit log? Runtime exception?
3. **Responsible surface:** is this policy the ES's responsibility, the UI's, an upstream service's, or a human process? Say which.

If the only enforcement mechanism is "the UI is supposed to" or "people won't do that," that's a policy-vs-enforcement gap — emit a `[CONCERN-N]` cross-linked to Phase 5a.

## Output — per DIFF

For each DIFF, record in its `governance_posture:`:

```yaml
governance_posture:
  ...
  scenario_coverage:
    external_calls:    "N covered, 0 unanswered"   # or list the unanswered ones
    mutable_fields:    "N covered, 0 unanswered"
    state_steps:       "N covered, 0 unanswered"
    inbound_listeners: "N covered, 0 unanswered"
    outbound_publishers: "N covered, 0 unanswered"
    policy_statements: "N covered, 0 unanswered"
```

If any row is non-zero, drafting is blocked until each unanswered question is converted into either (a) an explicit DIFF field / note answering it, or (b) a resolved `[CONCERN-N]`.

## Presenting coverage questions to the user

Batch them. Don't drip-feed. Example:

> Scenario coverage — 8 unanswered for DIFF G (CustomerSecurityFeePaid ingress + FCM egress):
>
> **External call (RMQ listener):**
> 1. What's the timeout on `getConnectionsByCustomer` during resolution?
> 2. If the CL call fails, ack or nack the AMQP message?
>
> **Mutable field (`pre_paid_bypass_applied`):**
> 3. Is this field ever reset to FALSE after synthetic fire? Or does it stay TRUE forever?
> 4. If a candidate transitions to a terminal state (cancelled) with flag TRUE, any cleanup?
>
> **State step (synthetic FEE_PAID fire):**
> 5. If `submitAadhar` is called concurrently with the payment handler, describe the interleaving and outcome.
> 6. How many StateTransitionLog rows on the bypass path? (Expecting 2 — confirm.)
>
> **Inbound listener (SuccessfulPaymentEventListener):**
> 7. If the same `booking_id` arrives with two different `messageId`s (legacy retry), what prevents double-fire?
> 8. FCM push on branch (b): fire once on payment receipt, or twice (receipt + synthetic transition)?
>
> Your answers drive the DIFF text — nothing drafts until you answer all 8.

## Scenario coverage interacts with minimality

Minimality (Phase 4a) says "don't include nice-to-haves." Scenario coverage (Phase 4b) says "include every answer the implementer needs." These aren't in tension — minimality cuts speculative additions; scenario coverage cuts speculative *ambiguity*. The skill resolves tension in favour of coverage.

If answering a scenario question results in content that feels like a nice-to-have (e.g. "document the retry policy for a call that will never fail in practice"), it's still required — implementers need the answer to build confidently.

## Why this phase exists

Without it, amendments produce long tech-review back-and-forth: "what if X?" / "the spec doesn't say, let me go ask." Each round-trip is ~1 day of latency. Phase 4b front-loads those questions so they're resolved at author time, when the PM has the context to answer.
