# Convention alignment check (Phase 5c)

Gate phase. Matches every new externally-visible contract the amendment introduces (error envelopes, endpoint shapes, event types, registration patterns, logging formats) against the project's declared conventions.

## When this phase runs

After Phase 5a (OS) and 5b (TAS). Input: the full set of draft DIFFs + the user-supplied project conventions doc (input #10). Output: PASS, or one or more `[CONCERN-N]` for each convention mismatch.

If input #10 is missing: skill runs a minimal check (common-sense shape matching against patterns seen in the base spec) and flags the limitation.

## What a "project conventions doc" looks like

The user-supplied conventions doc typically declares:

- **Standard error envelope** — e.g. `{ error_code, message, details, trace_id }`. Any new endpoint error response should match.
- **Endpoint naming patterns** — REST path shape, verb-noun rules, plural vs singular, versioning scheme (`/v1/...` vs `/api/v1/...`).
- **Registration patterns** — if the project uses a central `registry-service.producers[]` or similar, what's the registration shape, what's the required metadata, when should something be registered vs not.
- **Unregister / cleanup semantics** — if resources (tokens, subscriptions, registrations) have a register endpoint, what's the unregister pattern? DELETE? PATCH? Token TTL?
- **Logging / telemetry** — structured-log field names, required fields per log level, metric naming (`service_component_metric` vs `metric.service.component`).
- **Auth / permission annotations** — how new endpoints declare auth (`@CheckPermission`, `@Secured`, etc.).
- **Versioning** — schema_version conventions, deprecated-field handling.

Anything that crosses service boundaries or is visible to users / operators / other teams is in scope.

## Checklist — run for every DIFF that introduces externally-visible contracts

### CV-1 — Error envelope alignment

Does any new error response in the DIFF match the project's standard envelope?

Examples of mismatches to catch:
- New endpoint returns `{ error: "MSG", missing_keys: [...], invalid_keys: [...], reason: "..." }` while the project standard is `{ error_code, message, details, trace_id }`.
- New exception handler returns bare string body while others return structured JSON.
- HTTP status choice conflicts with project pattern (e.g. project uses 422 for validation, DIFF uses 400).

### CV-2 — Endpoint path pattern

Does any new endpoint's path match the project's pattern?

Examples:
- Project standard: `/api/v{N}/{resource}/{id}/{action}` — DIFF introduces `/v1/{resource}/...` (missing `/api/`).
- Project standard: singular resource names — DIFF uses plural.
- Project standard: kebab-case path segments — DIFF uses camelCase or snake_case.

### CV-3 — Event-type / wire-key naming

Do new event_ids or wire_keys match the project's naming pattern?

Examples:
- Project standard: `{PREFIX}_{VERB}_{OBJECT}` in UPPER_SNAKE_CASE — DIFF introduces `onSomethingHappened` or `object.verbed`.
- Project standard: ES-native events prefix `ES_<MODULE>_*` — DIFF uses `<MODULE>_*` without the prefix.

### CV-4 — Registration / unregister symmetry

If the DIFF adds something register-able (FCM token, webhook subscription, producer entry), does it also declare the unregister / cleanup path?

Examples:
- DIFF adds `POST /api/csp/fcm-tokens` to register — unregister endpoint not declared. Project pattern: every register has a DELETE counterpart. CONCERN.
- DIFF adds RMQ subscription but no deregistration flow for terminated services. CONCERN.
- DIFF registers event in producer list but doesn't say how it gets deregistered on event retirement.

### CV-5 — Logging and telemetry alignment

Do log statements / metrics in the DIFF match project patterns?

Examples:
- Project standard: structured logs with fields like `{ candidate_id, event_id, correlation_id }` — DIFF suggests `log.info("processed for {}", id)` (format-string style).
- Project metric naming: `service_subcomponent_metric` — DIFF introduces `metric.subcomponent.service`.

### CV-6 — Auth / permission annotation consistency

Does every new endpoint declare auth the same way as siblings?

Examples:
- Siblings use `@CheckPermission(action="X", resource="Y")` — DIFF's new endpoint uses `@Secured("ROLE_X")`. CONCERN.
- Siblings require JWT — DIFF's new endpoint declares `security.public-endpoints`. CONCERN unless explicitly a network-only endpoint.

### CV-7 — Version-number discipline

Do schema_version declarations / migration numbers / API versions follow the project pattern?

Examples:
- Project migrations `V001__...sql`, `V002__...sql` — DIFF introduces `V045A__...sql` with unusual suffix.
- Project events declare `schema_version: "1.0"` — DIFF declares `version: 1`.

### CV-8 — Forward-compat clauses without current use

Does any DIFF include "tolerate unknown X" / "handle future Y" clauses where no such X or Y exists today?

Examples:
- `partner_app_contract: "Unknown event enum → ignore and re-poll"` — but the enum has only one value (`SECURITY_FEE_PAID`) and no second value is planned.
- `version_handling: "Support schema_version N+1 as backward-compatible"` — but there's no plan to bump schema_version.

Speculative forward-compat wastes reviewer attention. CONCERN. Options: (a) remove the clause until a real second value exists, (b) add the real second value now if one is actually imminent.

## Running the sweep

```
Convention alignment report — AMENDMENT-04

CV-1 (error envelope):         ⚠ CONCERN-7 — DIFF C photo-upload 400 body doesn't match standard
CV-2 (endpoint path pattern):  ✓ PASS
CV-3 (event/wire naming):      ✓ PASS
CV-4 (register/unregister):    ⚠ CONCERN-8 — DIFF G FCM token register has no unregister path
CV-5 (logging/telemetry):      ✓ PASS
CV-6 (auth annotations):       ✓ PASS
CV-7 (version numbers):        ✓ PASS
CV-8 (speculative forward-compat): ⚠ CONCERN-9 — "tolerate unknown event enum" with no unknowns

3 concerns to resolve before drafting.
```

## Presenting concerns

Same batched pattern as other gate phases:

> Project convention alignment — 3 concerns:
>
> **[CONCERN-7]** Photo-upload 400 body shape doesn't match project error envelope.
>   Project standard: `{ error_code, message, details, trace_id }`.
>   DIFF C returns: `{ error, missing_keys, invalid_keys, reason }`.
>   Options: (a) reshape to project standard; (b) argue for an exception (unusual — validation errors usually fit the standard).
>   Recommendation: (a).
>
> **[CONCERN-8]** FCM token register endpoint has no unregister counterpart.
>   Options: (a) add `DELETE /api/csp/fcm-tokens/{token_id}`; (b) add `PATCH` with empty token; (c) document auto-expiry TTL and explain why no explicit unregister.
>   Recommendation: (a) — matches project pattern.
>
> **[CONCERN-9]** `partner_app_contract` says "tolerate unknown event enum" but enum has one value.
>   Options: (a) remove the clause; (b) add a second planned enum value now.
>   Recommendation: (a).

## Fallback when conventions doc not supplied

If input #10 missing, skill runs a reduced check:

1. **Shape matching against the base spec** — does the new error response / endpoint / event declaration look like similar ones already in the base? If they differ, flag as potential convention drift.
2. **Speculative forward-compat detection** — no conventions doc needed; minimality rule 12 (tightened version) catches these.

Note in amendment header:

```
# Convention alignment check (5c): reduced mode — no project conventions doc supplied.
# Only shape-matching against base spec applied. Full alignment unverified.
```

## Why this phase exists

New endpoints, new events, new registrations are all boundary surfaces the rest of the organisation has to integrate with. If each amendment picks its own error envelope or endpoint pattern, integration teams do translation work per boundary. Phase 5c flattens the translation surface to zero by requiring conformance to one project-wide convention.

Tech reviewers shouldn't have to say "our errors don't look like this — please reshape." The skill should have caught it.
