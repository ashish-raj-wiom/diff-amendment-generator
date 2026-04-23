# Concern flagging pattern

When Phase 5 (OS or TAS governance check) finds a risk, emit a numbered `[CONCERN-N]` block. Surface concerns **before** drafting (Phase 6) so the user decides up front.

## Block structure

```
[CONCERN-<N>] <one-line summary>
  Where:   <DIFF / section / field>
  Issue:   <what's wrong, 1-2 sentences>
  Options:
    (a) <minimal-effort fix, may leave some debt>
    (b) <cleanest fix, may cost more>
    (c) <do nothing — accept the risk explicitly>
  Recommendation: (<letter>) — <one-sentence reason>
```

**Rules:**
- Number concerns sequentially across the whole amendment (`CONCERN-1`, `CONCERN-2`, ... — not per-DIFF).
- Every concern gets 2 or 3 options. If you can only think of one, think again — there's always "do nothing."
- Give a recommendation. "Pick whichever" is a cop-out.
- The "Issue:" line is for the reviewer who's never seen this context before.

## Examples

```
[CONCERN-1] Customer mobile embedded in S3 URL path
  Where:   DIFF C — photo_asset_registry.object_key_template
  Issue:   Template 'installation/<cxMobile>/...' places customer mobile as a path
           segment; every URL stored in extra_data JSONB then contains mobile as a
           substring, violating the "mobile never stored" invariant.
  Options:
    (a) Switch path segment to <booking_id> (== customer_id) — removes mobile but
        still exposes an internal identifier.
    (b) Switch path segment to <execution_candidate_id> — ES-internal UUID; no
        customer-identifying information in URL.
    (c) Accept the current path; add ASM-<NN> declaring URL-level mobile as an
        explicit exception.
  Recommendation: (b) — strongest PII posture and no contract change visible to
                  external consumers.
```

```
[CONCERN-2] FCM egress not declared as dependency
  Where:   DIFF G — adds FcmClient and Firebase Admin SDK
  Issue:   Spec Section 2 (dependencies) has zero firebase/FCM references; the
           change adds a new downstream channel invisible to the spec.
  Options:
    (a) Add firebase-cloud-messaging to Section 2 as push-provider, downstream,
        fire-and-log semantics.
    (b) Omit from Section 2; treat FCM as an implementation-only concern.
  Recommendation: (a) — consistent with how other external providers are declared.
```

## State tracking

Each concern has a lifecycle: `PENDING` → `RESOLVED via option (<letter>)`.

**In the amendment header:**

```
# CONCERN STATUS:
#   [CONCERN-1] PENDING                   — <one-line summary>
#   [CONCERN-2] RESOLVED via option (a)   — <one-line summary>
```

**In the relevant DIFF's `governance_posture:`:**

```yaml
governance_posture:
  os_check:  "CONCERN-1 (PENDING)"     # or: "CONCERN-1 (RESOLVED)" after user decides
  tas_check: "PASS"
```

**Rules for state transitions:**
- Concerns start at `PENDING`.
- Only the user can resolve a concern. The skill proposes; the user decides.
- Once resolved, the skill writes the chosen option's content into the amendment and flips the status to `RESOLVED via option (<letter>)`.
- **No amendment ships with any `PENDING` concerns.** Phase 6 drafting does not start until all concerns are RESOLVED (or explicitly downgraded to "accepted risk" via option c, which is still RESOLVED).

## Avoiding stale state markers (Phase 7 hygiene)

A common drift: a concern gets resolved in one place but an inline note elsewhere still says "PENDING" or "see CONCERN-N — pending user decision."

Phase 7 scan:
1. Grep the amendment for every `[CONCERN-N]` reference.
2. Confirm each reference matches the header state (`RESOLVED via (<letter>)` or `PENDING`).
3. Flag mismatches. Offer to fix.

## How many concerns is too many?

More than ~4 concerns in one amendment is usually a signal that:
- The change request is too broad → consider splitting into multiple amendments.
- The amendment is doing governance-level rethinking → that belongs in a separate governance doc, not squeezed into a DIFF amendment.
- Several concerns share a root cause → consolidate them into one.

Two or three concerns is healthy for a medium-sized amendment; zero is fine for a truly minimal one.

## Presenting concerns to the user

At the end of Phase 5, present concerns as a batch to the user:

> Three concerns to resolve before drafting:
>
> **[CONCERN-1]** Customer mobile in S3 URL path — options (a/b/c) — recommend (b).
> **[CONCERN-2]** FCM egress not declared — options (a/b) — recommend (a).
> **[CONCERN-3]** Deferred FEE_PAID trigger not in signal-mapping — options (a/b/c) — recommend (b).
>
> Your picks?

Don't draft anything until all three come back answered.
