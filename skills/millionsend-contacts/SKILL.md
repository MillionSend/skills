---
name: millionsend-contacts
description: Manage contacts, segments, and topics on a MillionSend instance (self-hosted, Resend-compatible email API). Use when creating, listing, updating, or deleting contacts, handling unsubscribes, building segments (saved filters), or managing topic subscriptions via the MillionSend REST API or SDKs.
---

# MillionSend contacts, segments, and topics

Contacts are **team-global**: one list per team, one row per `(team, lowercased email)`. There is no "audiences" concept — `POST /contacts` creates directly, and segments (saved filters) replace audience lists. All routes need `Authorization: Bearer ms_...` with a **full_access** key (a sending-only key gets 403 `restricted_api_key`). Base URL is the user's instance (default `http://localhost:3001`).

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
    "properties": { "plan": "pro", "seats": 4 }
  }'
```

- `email` must be a bare address (no display name).
- `properties` is a flat map; scalar values are coerced to strings, nested objects/arrays are a 422.

Read/update/delete — the `{id}` path segment accepts **either the contact UUID or its email** (case-insensitive):

- `GET /contacts/{id}` → full contact incl. `properties`.
- `PATCH /contacts/{id}` — any of `first_name`/`last_name` (nullable to clear), `unsubscribed`, `properties`.
- `DELETE /contacts/{id}` → `{ "object": "contact", "contact": "<uuid>", "deleted": true }`.
- `GET /contacts?limit=&after=|before=` — keyset pagination: `limit` 1–100 (default 20), cursors are item ids, `after`/`before` mutually exclusive; response `{ object: "list", data: [...], has_more }`.

Unsubscribe semantics: `"unsubscribed": true` records the timestamp and excludes the contact from **all broadcasts** (transactional `POST /emails` sends are unaffected). Separate from this, hard bounces and complaints land on the team suppression list, which blocks both broadcasts and transactional sends. Broadcast emails carry RFC 8058 one-click unsubscribe links/headers automatically; recipients who click get `unsubscribed: true` set for them.

CSV import exists in the dashboard (Contacts page) — there is no import API endpoint; for bulk API loading, loop `POST /contacts`.

## Segments — saved filters over the team's contacts

Live at `/segments`. A segment is not a static list: it is a filter evaluated at send time.

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

Routes: `POST /segments`, `GET /segments` (paginated like contacts), `GET /segments/{id}` (includes live `contact_count`), `PATCH /segments/{id}` (name and/or filter), `DELETE /segments/{id}`. Ids are UUIDs.

## Topics — subscription preferences

Topics model opt-in/opt-out categories (e.g. "Product news"). `default_subscription` is fixed at creation: `opt_in` = subscribed unless the contact opts out; `opt_out` = unsubscribed unless they opt in.

- `POST /topics` `{ "name": "Product news", "description": "...", "default_subscription": "opt_in" }` → `{ id }`.
- `GET /topics` → `{ "data": [...] }` (no pagination); `GET /topics/{id}`; `DELETE /topics/{id}`.
- Set a contact's subscriptions — `PATCH /contacts/{id}/topics` with a **bare array** body:

```sh
curl -X PATCH "$MILLIONSEND_BASE_URL/contacts/ana@example.com/topics" \
  -H "Authorization: Bearer $MILLIONSEND_API_KEY" \
  -H "Content-Type: application/json" \
  -d '[{ "id": "<topic-uuid>", "subscription": "opt_out" }]'
```

## SDK equivalents

Node: `ms.contacts.create({...})`, `ms.contacts.list()`, `ms.contacts.get({ id | email })`, `ms.contacts.update(...)`, `ms.contacts.remove(...)`, plus `ms.topics.*` — same shapes as the REST bodies. Python: `millionsend.Contacts.create({...})`, `millionsend.Contacts.get(email="...")`, `millionsend.Topics.*`. Errors follow `{ statusCode, name, message }` with names like `validation_error`, `not_found`, `restricted_api_key`.
