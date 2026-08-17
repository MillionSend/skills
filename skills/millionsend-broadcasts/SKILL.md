---
name: millionsend-broadcasts
description: Compose, schedule, send, and cancel marketing broadcasts on a MillionSend instance (self-hosted, Resend-compatible email API). Use when sending campaigns to all contacts, a segment, or a topic, personalizing with merge fields like {{{FIRST_NAME|there}}}, or wiring unsubscribe links.
---

# MillionSend broadcasts

A broadcast is a campaign fanned out to the team's contacts. Lifecycle: **draft → queued (scheduled/sending) → sent**, or **canceled** while still queued. Only drafts can be edited/deleted; only scheduled broadcasts can be canceled. All routes require a **full_access** `ms_` key (`Authorization: Bearer ms_...`) against the instance base URL (default `http://localhost:3001`).

## Create a draft — POST /broadcasts

```sh
curl -X POST "$MILLIONSEND_BASE_URL/broadcasts" \
  -H "Authorization: Bearer $MILLIONSEND_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "August launch",
    "from": "Acme <news@acme.dev>",
    "subject": "New: dark mode",
    "html": "<p>Hi {{{FIRST_NAME|there}}}, dark mode is live.</p><p><a href=\"{{{UNSUBSCRIBE_URL}}}\">Unsubscribe</a></p>",
    "segment_id": "<segment-uuid>",
    "topic_id": "<topic-uuid>"
  }'
# → { "id": "<broadcast uuid>" }
```

- `from` (single mailbox, display name ok) and `subject` required; at least one of `html`/`text`. Optional: `name`, `reply_to`.
- **Targeting**: omit both `segment_id` and `topic_id` → every non-unsubscribed contact. `segment_id` → contacts matching that saved filter. `topic_id` → only contacts subscribed to that topic (default subscription + per-contact overrides). Both together intersect.
- Rejected loudly with 422 (not silently dropped): `preview_text`, `send: true` (send-on-create), `scheduled_at` on create — scheduling happens on the send call.

`PATCH /broadcasts/{id}` updates any of the same fields (draft only, else 400 `Only draft broadcasts can be updated`). `DELETE /broadcasts/{id}` deletes a draft. `GET /broadcasts` lists (keyset pagination: `limit` 1–100 default 20, `after`/`before` id cursors); `GET /broadcasts/{id}` returns the full body with wire `status` of `draft` | `queued` | `sent` | `canceled`.

## Merge fields (personalization)

Resend-compatible triple-brace syntax, resolved per recipient at fan-out:

- `{{{FIRST_NAME}}}`, `{{{LAST_NAME}}}`, `{{{EMAIL}}}` — builtins from the contact record.
- `{{{plan}}}` — any custom contact property, matched by exact key (builtins win over a same-named property).
- `{{{FIRST_NAME|there}}}` — fallback after `|` used when the value is missing/empty; without a fallback an empty value renders as `""` (a raw token never reaches an inbox).
- Values are HTML-escaped in the `html` body (not in `text`); fallback text is the author's own and stays literal.

## Unsubscribe link

Put the literal token `{{{UNSUBSCRIBE_URL}}}` wherever the unsubscribe link belongs — it is replaced per recipient with a signed hosted unsubscribe URL (`<APP_BASE_URL>/unsubscribe/<token>`, topic-scoped when the broadcast targets a topic). Independently of the in-body link, every broadcast email gets RFC 8058 `List-Unsubscribe` + `List-Unsubscribe-Post: List-Unsubscribe=One-Click` headers automatically. Unsubscribed and suppressed (bounced/complained) contacts are skipped automatically at fan-out.

## Send or schedule — POST /broadcasts/{id}/send

```sh
# immediately
curl -X POST "$MILLIONSEND_BASE_URL/broadcasts/$ID/send" \
  -H "Authorization: Bearer $MILLIONSEND_API_KEY" \
  -H "Content-Type: application/json" -d '{}'

# or scheduled (ISO 8601 with offset, max 30 days ahead; no natural language)
curl -X POST "$MILLIONSEND_BASE_URL/broadcasts/$ID/send" \
  -H "Authorization: Bearer $MILLIONSEND_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "scheduled_at": "2026-09-01T10:00:00-03:00" }'
```

Preconditions checked at send time (all return the standard `{ statusCode, name, message }` error body):

- The instance must have `APP_BASE_URL` set — unsubscribe links are built from it (422 otherwise).
- The `from` domain must be a verified team domain (422 `The <domain> domain is not verified for this team`). A domain-scoped key must match it (403 `restricted_api_key`).
- If the team's bounce/complaint rate is at the SES pause line: 403 `sending_paused` — lower it before new campaigns.

## Cancel — POST /broadcasts/{id}/cancel

Works only while status is scheduled (`queued` on the wire, before fan-out starts); after that, 400 `Only scheduled broadcasts can be canceled`.

## SDK equivalents

Node: `ms.broadcasts.create({...})`, `.update(id, {...})`, `.list()`, `.get(id)`, `.send(id, { scheduledAt })`, `.cancel(id)`, `.remove(id)`. Python: `millionsend.Broadcasts.create({...})`, `.send(id, {"scheduled_at": ...})`, etc. Same field names and error codes as REST.
