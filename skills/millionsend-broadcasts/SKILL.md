---
name: millionsend-broadcasts
description: Compose, schedule, send, and cancel marketing broadcasts on MillionSend (Resend-compatible email API), and manage reusable templates via /templates. Use when sending campaigns to all contacts, a segment, or a topic, sending or scheduling on create, personalizing with merge fields like {{{FIRST_NAME|there}}}, setting preview (preheader) text, wiring unsubscribe links, or creating/updating/duplicating templates (with aliases) through the API.
---

# MillionSend broadcasts

A broadcast is a campaign fanned out to the team's contacts. Lifecycle: **draft → queued (scheduled/sending) → sent**, or **canceled** while still queued. Only drafts can be edited/deleted; only scheduled broadcasts can be canceled. All routes require a **full_access** `ms_` key (`Authorization: Bearer ms_...`). Base URL: `https://api.millionsend.com` (cloud) or the instance's API origin (default `http://localhost:3001`).

## Create — POST /broadcasts

```sh
curl -X POST "$MILLIONSEND_BASE_URL/broadcasts" \
  -H "Authorization: Bearer $MILLIONSEND_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "August launch",
    "from": "Acme <news@acme.dev>",
    "subject": "New: dark mode",
    "preview_text": "Dark mode is live for every plan",
    "html": "<p>Hi {{{FIRST_NAME|there}}}, dark mode is live.</p><p><a href=\"{{{UNSUBSCRIBE_URL}}}\">Unsubscribe</a></p>",
    "segment_id": "<segment-uuid>",
    "topic_id": "<topic-uuid>"
  }'
# → { "id": "<broadcast uuid>" }
```

- `from` (single mailbox, display name ok) and `subject` required; at least one of `html`/`text`. Optional: `name`, `reply_to`, `preview_text` (inbox preview — injected as a hidden preheader at fan-out).
- **Targeting**: omit both `segment_id` and `topic_id` → every non-unsubscribed contact. `segment_id` → contacts the segment resolves to (saved filter matches ∪ manual members). `topic_id` → only contacts subscribed to that topic (default subscription + per-contact overrides). Both together intersect.
- **Send on create**: `"send": true` sends immediately; add `"scheduled_at"` (requires `send: true`) to schedule instead — ISO 8601 with offset or a relative time (`"in 1 hour"`, `"in 2 days"`), max 30 days ahead.

`PATCH /broadcasts/{id}` updates any of the content/targeting fields (draft only, else 400 `invalid_parameter` "Only draft broadcasts can be updated"). `DELETE /broadcasts/{id}` deletes a draft. `GET /broadcasts` lists (keyset pagination: `limit` 1–100 default 20, `after`/`before` id cursors); `GET /broadcasts/{id}` returns the full body (`from`, `subject`, `preview_text`, `topic_id`, `html`, `text`, ...) with wire `status` of `draft` | `queued` | `sent` | `canceled` (internal scheduled/sending both surface as `queued`).

## Merge fields (personalization)

Resend-compatible triple-brace syntax, resolved per recipient at fan-out:

- `{{{FIRST_NAME}}}`, `{{{LAST_NAME}}}`, `{{{EMAIL}}}` — builtins from the contact record.
- `{{{plan}}}` — any custom contact property, matched by exact key (builtins win over a same-named property; a property definition's `fallback_value` fills missing keys).
- `{{{FIRST_NAME|there}}}` — fallback after `|` used when the value is missing/empty; without a fallback an empty value renders as `""` (a raw token never reaches an inbox).
- Values are HTML-escaped in the `html` body (not in `text`); fallback text is the author's own and stays literal.

## Unsubscribe link

Put the literal token `{{{UNSUBSCRIBE_URL}}}` wherever the unsubscribe link belongs — it is replaced per recipient with a signed hosted unsubscribe URL (`<APP_BASE_URL>/unsubscribe/<token>`, topic-scoped when the broadcast targets a topic). Independently of the in-body link, every broadcast email gets RFC 8058 `List-Unsubscribe` + `List-Unsubscribe-Post: List-Unsubscribe=One-Click` headers automatically. Unsubscribed and suppressed (bounced/complained) contacts are skipped automatically at fan-out.

## Send or schedule a draft — POST /broadcasts/{id}/send

```sh
# immediately
curl -X POST "$MILLIONSEND_BASE_URL/broadcasts/$ID/send" \
  -H "Authorization: Bearer $MILLIONSEND_API_KEY" \
  -H "Content-Type: application/json" -d '{}'

# or scheduled — ISO 8601 with offset, or relative ("in 1 hour"), max 30 days ahead
curl -X POST "$MILLIONSEND_BASE_URL/broadcasts/$ID/send" \
  -H "Authorization: Bearer $MILLIONSEND_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "scheduled_at": "2026-09-01T10:00:00-03:00" }'
```

Only drafts can be sent (400 `invalid_parameter` otherwise). Preconditions checked at send time (all return the standard `{ statusCode, name, message }` error body):

- The instance must have `APP_BASE_URL` set — unsubscribe links are built from it (422 otherwise).
- The `from` domain must be a verified team domain (422 `The <domain> domain is not verified for this team`). A domain-scoped key must match it (403 `restricted_api_key`).
- If the team's bounce/complaint rate is at the SES pause line: 403 `sending_paused` — lower it before new campaigns.

## Cancel — POST /broadcasts/{id}/cancel

Works only while status is scheduled (`queued` on the wire, before fan-out starts); after that, 400 `invalid_parameter` "Only scheduled broadcasts can be canceled".

## Templates — /templates

Reusable content (`subject`, `html`, optional `text`) for broadcasts. The dashboard composer's template picker **copies** a template's content into a broadcast as a starting snapshot — no link back, later template edits change nothing. Wire-compatible with Resend's `templates` surface (the `resend` SDK's `templates.create/get/list/update/publish/duplicate/remove` work as-is), with these deltas:

- **No draft/publish cycle, no versions**: every save is live. `status` is always `published`, `published_at` = `created_at`, `current_version_id` = `id`, `has_unpublished_versions` = `false`; `POST /templates/{id}/publish` is an idempotent no-op (404 if unknown).
- **Not supported yet — loud, not dropped**: `from`, `reply_to`, `variables` with a value → 422 `validation_error` "`<field>` is not supported on templates yet"; reads return `null`, `null`, `[]`. Merge fields (`{{{FIRST_NAME}}}`, `{{{plan}}}`) work without declaring variables.
- **No send-by-template yet**: `POST /emails` and `POST /broadcasts` take no template reference — pass `html`/`text` (fetch them from `GET /templates/{id}` if needed).

```sh
curl -X POST "$MILLIONSEND_BASE_URL/templates" \
  -H "Authorization: Bearer $MILLIONSEND_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "name": "Welcome", "alias": "welcome-v1", "subject": "Hi {{{FIRST_NAME|there}}}", "html": "<p>Welcome aboard.</p>" }'
# → { "object": "template", "id": "<uuid>" }
```

- `name` 1–200 chars (trimmed), `html` required (≤ 500k, stored as sent), `subject` ≤ 998, `text` ≤ 500k; `""` for `subject`/`text` clears it.
- `alias` (optional): `^[A-Za-z0-9][A-Za-z0-9._-]*$`, ≤ 100 chars, case-sensitive, unique per team, must not look like a UUID (422). Taken → 409 with name `validation_error`, message `Template alias already exists`.
- Every single-template route takes **id or alias**: `GET /templates/{id|alias}` (full body), `PATCH /templates/{id|alias}` (any of `name`, `alias` — `null` clears —, `subject`, `html`, `text`; empty body is a no-op), `DELETE /templates/{id|alias}` → `{ ..., "deleted": true }` (broadcasts keep their copied content), `POST /templates/{id|alias}/publish`, `POST /templates/{id|alias}/duplicate` → new id, named `<name> (copy)`, no alias.
- `GET /templates?limit=&after=|before=` — keyset list of `{ id, name, alias, status, published_at, created_at, updated_at }`.

## SDK equivalents

Node: `ms.broadcasts.create({...})` (incl. `previewText`, `send`, `scheduledAt`), `.update(id, {...})`, `.list()`, `.get(id)`, `.send(id, { scheduledAt })`, `.cancel(id)`, `.remove(id)`. Python: `millionsend.Broadcasts.create({...})`, `.send(id, {"scheduled_at": ...})`, etc. Same field names and error codes as REST. Templates: the official `resend` SDK's `templates.*` pointed at MillionSend (`resend.templates.create({...}).publish()` chains fine — publish is a no-op).
