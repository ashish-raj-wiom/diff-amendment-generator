# Reality-check playbook (Phase 3)

Phase 3 is a gate: no drafting until every non-MATCH item is acknowledged by the user. The skill greps the codebase for every entity the amendment would touch or assert, and classifies each change-request item into one of four buckets.

## Classification

For each change-request item:

- **`MATCH`** — spec change aligns with code. Proceed.
- **`DRIFT`** — spec says X, code has Y. User must choose: align the spec to the code, OR the amendment also needs code-side changes.
- **`GAP`** — spec is silent but the behaviour exists in code. User decides whether to include documentation in this amendment or leave silent.
- **`PHANTOM`** — spec asserts / wants to assert X, but code has nothing of the kind. Blocks drafting until resolved. Either the code needs to land first, or the change request is mis-scoped.

## What to grep for

The skill inspects each change-request item and extracts any of the following entity references. For each, it runs the listed check.

### Class / interface name

Item says: "add dependency on `NewServiceClient`"

Check: `Grep pattern: "class NewServiceClient|interface NewServiceClient"`
- Found → MATCH
- Not found → PHANTOM (blocker)

### DB column

Item says: "persist `paid_at` on `orders` table"

Checks:
1. Entity class: `Grep "orders" AND "paidAt|paid_at"` in JPA entity / ORM files
2. Migration: look in migrations directory for `ALTER TABLE orders ADD COLUMN paid_at` or `CREATE TABLE orders` with `paid_at`

Matrix:
- Entity has it + migration creates it → MATCH
- Entity missing + migration missing → PHANTOM
- Entity has it + migration missing → DRIFT (inconsistent state; investigate)
- Migration has it + entity missing → DRIFT

### Event / message key

Item says: "emit `EVENT_X`"

Check: `Grep "EVENT_X"` across:
- Event classes (`domain/event/...`)
- Publishers
- Listeners
- application.yml registry blocks

- Found in publisher → MATCH (if item asserts emit)
- Found only in listener → DRIFT (code consumes, amendment says produces)
- Not found anywhere → PHANTOM

### Migration file

Item says: "V042 adds column X"

Check: `Glob db/migration/V042__*.sql` (or equivalent migration path pattern)
- File exists, contains `ADD COLUMN X` → MATCH
- File exists, doesn't mention X → DRIFT
- File missing → PHANTOM

### DTO field

Item says: "payload has field `booking_id` of type long"

Check: `Grep "booking_id\|bookingId"` in DTO class, check its declared type:
- Matches → MATCH
- Field exists but type differs → DRIFT
- Field missing → PHANTOM

### Handler method / flow step

Item says: "submitAadhar fires synthetic FEE_PAID at end of method"

Check: `Read the relevant handler class`:
- Method exists and implements the asserted logic → MATCH
- Method exists but doesn't do what's asserted → DRIFT (most common case — spec has aspiration, code hasn't caught up)
- Method missing → PHANTOM

## Grep strategy

**Start broad, narrow down.**

1. Identify the entity name (class, column, event_id, field, migration tag).
2. First pass: `Grep` across the whole codebase root for the exact name.
3. If too many hits: scope with `glob` or `path` to the service / module directory.
4. If entity has ambiguous name (e.g. `status`): grep with context like `private.*status` or `@Column.*status`.

**Use case-sensitive matching by default.** Reality-check hits on entity names (classes, events, constants) are almost always case-correct. Case-insensitive greps produce too much noise.

**Look at file types:**
- Java / Kotlin entities → `infrastructure/persistence/*Entity.java` (or equivalent path convention)
- DTOs → `infrastructure/.../dto/*.java`
- Domain events → `domain/event/...`
- Listeners / consumers → `infrastructure/messaging/...listener/...`
- Migrations → `resources/db/migration/*.sql` (Flyway) or `migrations/*.rb` (Rails) etc.
- Config → `application.yml` / `application-*.yml`
- Handlers → `application/impl/*ServiceImpl.java` (or equivalent)

If navigation hints were provided in input #10, start with those paths. Otherwise grep from the codebase root.

## Code-truth report format

After Phase 3 greps complete, present results to the user:

```
Phase 3 — Reality check report

CHANGE-REQUEST ITEM                              | CODE STATUS | NOTES
─────────────────────────────────────────────────┼─────────────┼────────────────────
Add i2e1-customer-service as dependency          | MATCH       | Live in CustomerServiceClient.java
Mobile fetched at call time                      | DRIFT       | Method exists but hardcodes CONNECTING,
                                                 |             | doesn't actually call Exotel
Add column `pre_paid_bypass_applied`             | MATCH       | V045 migration + entity field present
Event SUCCESSFUL_PAYMENT_EVENT is consumed       | PHANTOM     | No listener exists; Rabbit config missing
Emit EsInstallCallInitiated                      | MATCH       | Handler emits it at line 654
Set securityFeePaidAt from event payload         | DRIFT       | Code stamps clock.now(); payload has no
                                                 |             | paidAt field

Summary: 2 MATCH, 2 DRIFT, 1 PHANTOM. Cannot proceed to Phase 6.

For each DRIFT / PHANTOM, the user must choose:
  - Align the spec to code (spec changes, code stays)
  - Require code changes alongside this amendment (flagged in Coordination)
  - Drop the item from this amendment (deferred)
  - (PHANTOM only) Block until code lands

Await user's decision on each before proceeding.
```

## When the codebase spans multiple repos / services

Some base specs describe a service whose implementation spans the base-spec repo plus consumer repos. The skill only reality-checks against the repo(s) the user supplied in input #4.

If a change-request item references behaviour in a consumer service the user didn't supply:
- Emit an `UNVERIFIABLE` finding (similar to DRIFT / GAP) and document it as a limitation in the amendment header.
- Do not assume the consumer has / doesn't have the behaviour.

## Type discipline (new in Phase 3)

After the four-class check above, run a second pass specifically on **cross-boundary field types**. This catches silent type drift that compiles but breaks at runtime.

### Boundaries to check

For every field that crosses a boundary, verify both sides declare the same type:

- **Wire payload ↔ DTO class** — payload `booking_id: long` must align with DTO `Long bookingId` or `long bookingId`. A `UUID` on one side and `Long` on the other is DRIFT.
- **DTO ↔ Entity column** — DTO `String customerId` ↔ entity `@Column customer_id VARCHAR` aligns. DTO `String customerId` ↔ `Long.parseLong(...)` in the handler → **load-bearing parse**: flag if the source column is ever non-numeric elsewhere.
- **Producer payload ↔ Consumer payload** — for each wire_key or event_id, the producer's declared payload type must match what the consumer parses.
- **Required / optional flip** — producer emits field as required; consumer treats as optional (or vice versa). DRIFT.
- **JSON naming drift** — producer uses `snake_case`, consumer expects `camelCase`. DRIFT unless a Jackson/Jackson-equivalent naming strategy is configured on both sides to agree.

### Common DRIFT patterns worth calling out loudly

1. **UUID-stringified-as-Long** — `Long.parseLong(someUuidString)` assumes the upstream identifier is numeric. If the source column is ever a true UUID (`8-4-4-4-12` hex), parse throws. Flag this every time. Solution: lock the type at the source OR hold as String everywhere.
2. **Instant ↔ LocalDateTime** — subtle but common. One side carries timezone, the other doesn't. Flag and propose one canonical representation.
3. **Enum string ↔ enum ordinal** — producer serialises string name, consumer reads ordinal, or vice versa. Flag.
4. **Boolean nullable ↔ primitive boolean** — `Boolean isActive` (nullable) on DTO, `boolean isActive` (primitive `NOT NULL DEFAULT FALSE`) on entity. Rehydration of null throws NPE. Flag.
5. **Column type vs migration DEFAULT clause** — `VARCHAR(255)` column with `DEFAULT 0` in a migration is incoherent. Flag.

### Output classification

Type mismatches are classified as `DRIFT`. Presented to the user alongside other DRIFT items:

```
CHANGE-REQUEST ITEM                              | CODE STATUS | NOTES
─────────────────────────────────────────────────┼─────────────┼────────────────────
booking_id type (payload vs parse)               | DRIFT       | Payload declares Long;
                                                 |             | resolver does Long.parseLong(UUID).
                                                 |             | Load-bearing on customer_id
                                                 |             | being numeric-stringified.
```

User must choose: align at the source, switch to String everywhere, or accept the load-bearing assumption with an explicit structural-assumption entry.

### When type information is insufficient

If the codebase uses duck-typing / dynamic languages (Python, Ruby, untyped JSON) or if the payload comes from a legacy system whose types aren't declared, the skill can still flag common-sense mismatches (`Long.parseLong(...)` on a field that looks like a UUID string, `new Date(...)` on an Instant, etc.) but should mark these as `DRIFT (inferred)` to signal lower confidence.

## What this phase does NOT do

- **Does not run the code.** Grep + static type inspection only.
- **Does not validate payload schemas at runtime.** It verifies field names and declared types match across boundaries.
- **Does not check semantic correctness.** If the code writes to a column the governance says must not be written to, that's Phase 5a (OS governance check), not Phase 3.

Phase 3 is structural + type verification only. Phase 5 is rules verification.
