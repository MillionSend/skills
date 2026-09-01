---
name: millionsend-contacts
description: Manage contacts, typed contact properties, segments, topics, and the suppression list on MillionSend (Resend-compatible email API). Use when creating, listing, updating, or deleting contacts, bulk-loading contacts via POST /contacts/batch (skip/upsert), defining custom property keys, handling unsubscribes, building segments (saved filters or manual lists), managing segment membership, managing topic subscriptions, or adding/removing suppressed addresses via the MillionSend REST API or SDKs.
---

# MillionSend contacts, properties, segments, and topics

Contacts are **team-global**: one list per team, one row per `(team, lowercased email)`. There is no "audiences" concept — `POST /contacts` creates directly, and segments replace audience lists (`/audiences/{id}/contacts[...]` still works as a legacy alias for Resend v5-style SDKs; the audience id is a segment id, and creating through it also joins that segment). All routes need `Authorization: Bearer ms_...` with a **full_access** key (a sending-only key gets 403 `restricted_api_key`). Base URL: `https://api.millionsend.com` (cloud) or the instance's API origin (default `http://localhost:3001`).

## Contacts

Create — `POST /contacts` (200 `{ "object": "contact", "id": "<uuid>" }`; duplicate email → 409 with name `validation_error`, message `Contact already exists`):

```sh
curl -X POST "$MILLIONSEND_BASE_URL/contacts" \
  -H "Authorization: Bearer $MILLIONSEND_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "ana@example.com",
    "first_name": "Ana",
    "last_name": "Silva",
    "unsubscribed": false,
    "properties": { "plan": "pro", "seats": 4 },
    "segments": [{ "id": "4ef9a417-02e9-4d39-ad75-9611e0fcc33c" }],
    "topics": [{ "id": "<topic-uuid>", "subscription": "opt_in" }]
  }'
```

- `email` must be a bare address (no display name).
- `properties` is a flat map; scalar values are accepted (stored as strings), nested objects/arrays are a 422. Reads type them back per the team's property definitions.
- `segments` / `topics` are optional initial associations, written in the same transaction as the contact; a missing segment/topic id → 404.

Read/update/delete — the `{id}` path segment accepts **either the contact UUID or its email** (case-insensitive):

- `GET /contacts/{id}` → full contact; each entry of `properties` is a typed wrapper `{ "type": "string" | "number", "value": ... }` (the type comes from the team's property definitions, see below).
- `PATCH /contacts/{id}` — any of `first_name`/`last_name` (nullable to clear), `unsubscribed`, `properties` (merged; a `null` value removes the key).
- `DELETE /contacts/{id}` → `{ "object": "contact", "contact": "<uuid>", "deleted": true }`.
- `GET /contacts?limit=&after=|before=` — keyset pagination: `limit` 1–100 (default 20), cursors are item ids, `after`/`before` mutually exclusive; response `{ object: "list", data: [...], has_more }`.

Unsubscribe semantics: `"unsubscribed": true` records the timestamp and excludes the contact from **all broadcasts** (transactional `POST /emails` sends are unaffected). Separate from this, hard bounces and complaints land on the team suppression list (below), which blocks both broadcasts and transactional sends. Broadcast emails carry RFC 8058 one-click unsubscribe links/headers automatically; recipients who click get `unsubscribed: true` set for them **and** a retained suppression entry with origin `unsubscribe` — only an explicit `PATCH /contacts/{id}` with `"unsubscribed": false` clears it (re-creating or batch-importing the address never does).

### Bulk create — POST /contacts/batch

A MillionSend extension (Resend imports contacts only via CSV): a JSON **array** of 1–1000 `POST /contacts` bodies, written in one transaction. The dashboard's CSV import is the other bulk path.

```sh
curl -X POST "$MILLIONSEND_BASE_URL/contacts/batch?on_conflict=upsert" \
  -H "Authorization: Bearer $MILLIONSEND_API_KEY" \
  -H "Content-Type: application/json" \
  -H "x-batch-validation: permissive" \
  -d '[
    { "email": "ana@example.com", "first_name": "Ana", "properties": { "plan": "pro" } },
    { "email": "bob@example.com", "segments": [{ "id": "<segment-uuid>" }] }
  ]'
# → { "data": [{ "object": "contact", "index": 0, "id": "<uuid>", "status": "updated" }, { "object": "contact", "index": 1, "id": "<uuid>", "status": "created" }],
#     "counts": { "created": 1, "updated": 1, "skipped": 0, "failed": 0 } }
```

- `on_conflict` (query, default `error`) — what to do with an email that already belongs to a contact, and with an email repeated inside the batch: `error` → the item fails (409 `Contact already exists` / 422 `Duplicate email in batch`); `skip` → existing contact (or first occurrence) untouched, reported as `status: "skipped"` with its id; `upsert` → merge: `first_name`/`last_name` only when provided, `properties` merged key by key, `segments` added, `topics` upserted; repeats collapse into one write (later scalars win, associations union).
- **Never re-subscribes**: `unsubscribed: true` opts out; `unsubscribed: false` on an already-unsubscribed contact is ignored. Suppressions are never touched. Use `PATCH /contacts/{id}` to re-subscribe deliberately.
- `x-batch-validation` (header, default `strict`) — strict: the first failing item (by index) fails the whole batch with its own status and a `contacts.<index>: <message>` prefix, nothing written. `permissive`: valid subset written, failures listed as `errors: [{ index, message }]`.
- Response: `data` in request order, one entry per successful item; `counts` sum to the request length; `errors` only in permissive mode. Unknown segment/topic id → 404 `not_found`; 0 or >1000 items, unknown `on_conflict`/header value → 422.

## Contact properties — typed keys at /contact-properties

Property **definitions** give each key a type (`string` | `number`) and an optional fallback used when a contact lacks the key (e.g. in broadcast merge fields):

```sh
curl -X POST "$MILLIONSEND_BASE_URL/contact-properties" \
  -H "Authorization: Bearer $MILLIONSEND_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "key": "plan", "type": "string", "fallback_value": "free" }'
# → { "object": "contact_property", "id": "<uuid>" }
```

- `key` ≤ 200 chars; `fallback_value` must match `type` (a `number` property rejects non-numeric fallbacks); string fallbacks ≤ 1000 chars.
- `GET /contact-properties` (paginated list) · `GET /contact-properties/{id}` · `PATCH /contact-properties/{id}` (**only** `fallback_value` is updatable — key and type are fixed) · `DELETE /contact-properties/{id}` → `{ ..., "deleted": true }`.

## Segments — saved filters or manual lists at /segments

A segment resolves to: contacts matching its saved **filter** (if any) **OR** contacts added as manual **members**. Omit `filter` on create for a purely manual segment.

```sh
curl -X POST "$MILLIONSEND_BASE_URL/segments" \
  -H "Authorization: Bearer $MILLIONSEND_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Pro users",
    "filter": {
      "match": "all",
      "conditions": [
        { "field": "property:plan", "op": "equals", "value": "pro" },
        { "field": "unsubscribed", "op": "is_false", "value": null }
      ]
    }
  }'
```

Filter grammar (validated server-side, unknown field/op → 422):

- `match`: `"all"` (AND) or `"any"` (OR).
- Text fields `email`, `first_name`, `last_name`, and `property:<key>` take ops `equals`, `not_equals`, `contains`, `starts_with`, `ends_with`, `is_set`, `is_not_set`. Presence ops (`is_set`/`is_not_set`) take `"value": null`; the rest need a string value.
- `unsubscribed` takes `is_true` / `is_false` (value null).
- `created_at` takes `before` / `after` with an ISO date string value.

Routes: `POST /segments`, `GET /segments` (paginated like contacts), `GET /segments/{id}` (includes live `contact_count`), `PATCH /segments/{id}` (name and/or filter; `"filter": null` clears it, turning the segment manual-only), `DELETE /segments/{id}` (409 `conflict` if a broadcast references it).

Membership (manual members; the contact path accepts UUID or email):

- `POST /contacts/{id}/segments/{segmentId}` — add; idempotent upsert (adding twice is fine) → `{ "id": "<contact uuid>" }`.
- `DELETE /contacts/{id}/segments/{segmentId}` — remove → `{ "id": ..., "audienceId": "<segment uuid>", "deleted": true }`; removing a non-member is a 404.
- `GET /segments/{id}/contacts?limit=&after=` — list the segment's resolved contacts (filter matches ∪ manual members). SDK: `ms.contacts.list({ segmentId })`.

## Topics — subscription preferences

Topics model opt-in/opt-out categories (e.g. "Product news"). `default_subscription` is fixed at creation: `opt_in` = subscribed unless the contact opts out; `opt_out` = unsubscribed unless they opt in.

- `POST /topics` `{ "name": "Product news", "description": "...", "default_subscription": "opt_in", "visibility": "public" }` → `{ id }`. `visibility` (`private` | `public`, MillionSend extension): public topics always show on the hosted unsubscribe/preferences page; private topics show there only when reached through their own topic link.
- `GET /topics` → `{ "data": [...] }` (no pagination); `GET /topics/{id}`; `PATCH /topics/{id}` (name, description, visibility — `default_subscription` is immutable and silently ignored if sent); `DELETE /topics/{id}` (409 `conflict` if a broadcast references it).
- Set a contact's subscriptions — `PATCH /contacts/{id}/topics` with a **bare array** body:

```sh
curl -X PATCH "$MILLIONSEND_BASE_URL/contacts/ana@example.com/topics" \
  -H "Authorization: Bearer $MILLIONSEND_API_KEY" \
  -H "Content-Type: application/json" \
  -d '[{ "id": "<topic-uuid>", "subscription": "opt_out" }]'
```

## Suppressions — /suppressions

The team's do-not-send list. Entries block both broadcasts and transactional sends (`POST /emails` strips suppressed recipients; all-`to`-suppressed → 422 `All recipients are suppressed`). Each entry: `{ id, email, origin, source_id, created_at }` — `origin` is `bounce` | `complaint` | `manual` | `unsubscribe` (the last is a MillionSend superset value: retained one-click opt-outs), `source_id` the email id whose bounce/complaint created it (else null). Same wire as Resend's `suppressions` surface, so the `resend` SDK's `suppressions.add/get/list/remove` and `suppressions.batch.add/remove` work as-is.

```sh
# block one address (origin manual). Idempotent: already suppressed for any origin → same row, its existing id returned
curl -X POST "$MILLIONSEND_BASE_URL/suppressions" \
  -H "Authorization: Bearer $MILLIONSEND_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "email": "bounced@example.com" }'
# → { "object": "suppression", "id": "<uuid>" }

# bulk (up to 1000 per call; Resend caps at 100) — e.g. carrying a bounce list over from another provider
curl -X POST "$MILLIONSEND_BASE_URL/suppressions/batch/add" \
  -H "Authorization: Bearer $MILLIONSEND_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "emails": ["a@example.com", "b@example.com"] }'
# → { "data": [{ "object": "suppression", "id": "..." }, ...] } — one per distinct address, input order
```

- `GET /suppressions?limit=&after=|before=&origin=bounce` — keyset list; unknown `origin` → 422.
- `GET /suppressions/{id}` / `DELETE /suppressions/{id}` — `{id}` is the suppression UUID **or the email address**; 404 `not_found` otherwise. Delete → `{ "object": "suppression", "id": ..., "deleted": true }` and the address can receive mail again (it is re-suppressed automatically on the next bounce/complaint).
- `POST /suppressions/batch/remove` — body `{ "emails": [...] }` **or** `{ "ids": [...] }` (exactly one, 1–1000); returns only the rows actually removed.
- Addresses erased under GDPR/LGPD keep blocking sends but are hidden from the list and from lookups by email; they are reachable by id only (email reads `"[erased]"`), and re-adding the address returns that id without restoring it.

## SDK equivalents

Node: `ms.contacts.create({...})`, `ms.contacts.list({ segmentId? })`, `ms.contacts.get({ id | email })`, `ms.contacts.update(...)`, `ms.contacts.remove(...)`, `ms.contacts.segments.add/remove(...)`, plus `ms.segments.*` and `ms.topics.*` — same shapes as the REST bodies. Python: `millionsend.Contacts.create({...})`, `millionsend.Contacts.get(email="...")`, `millionsend.Segments.*`, `millionsend.Topics.*`. Contact-properties and `POST /contacts/batch` have no SDK surface yet — use REST; suppressions are reachable through the official `resend` SDK pointed at MillionSend (`resend.suppressions.*`). Errors follow `{ statusCode, name, message }` with names like `validation_error`, `not_found`, `conflict`, `restricted_api_key`.
