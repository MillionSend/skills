---
name: millionsend-send-email
description: Send transactional email through MillionSend (Resend-compatible email API on AWS SES) — POST /emails with attachments, custom headers, topic-scoped sends, ISO or relative scheduling, idempotency, batch (strict or permissive), reschedule/cancel, the SDKs, and the SMTP relay. Use when sending, scheduling, batching, fetching, rescheduling, or canceling emails on a MillionSend cloud or self-hosted instance.
---

# Send email with MillionSend

Base URL: `https://api.millionsend.com` (cloud) or the instance's own API origin (self-hosted; `http://localhost:3001` on a local compose setup). Authenticate with `Authorization: Bearer ms_...` — API keys are created in the dashboard or via `POST /api-keys` and start with `ms_`.

## Prerequisites

- The `from` address's domain must be a **verified domain** of the team (dashboard → Domains, or the `/domains` API — see the millionsend-domains skill). Otherwise: 422 `The <domain> domain is not verified for this team`.
- A key with `sending_access` permission can only use `/emails*` routes. A domain-scoped key can only send from its assigned domain (403 `restricted_api_key` otherwise).

## Send one email — POST /emails

```sh
curl -X POST "$MILLIONSEND_BASE_URL/emails" \
  -H "Authorization: Bearer $MILLIONSEND_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "from": "Acme <onboarding@acme.dev>",
    "to": ["user@example.com"],
    "subject": "Welcome",
    "html": "<p>It works!</p>"
  }'
# → { "id": "<email uuid>" }
```

Request fields (Resend-shaped, snake_case):

- `from` (required) — exactly one mailbox; display name allowed (`"Acme <x@acme.dev>"`).
- `to` (required) — string or array, max 50 recipients. `cc`, `bcc`, `reply_to` same shape.
- `subject` (required); at least one of `html` / `text` (required).
- `scheduled_at` — ISO 8601 **with offset** (`2026-09-01T10:00:00-03:00`) or a relative time: `in N min(s)/minute(s)/hour(s)/day(s)` ("in 5 mins", "in 2 hours", "in 1 day"). Max 30 days ahead.
- `tags` — `[{ "name": "...", "value": "..." }]`.
- `topic_id` — a topic UUID: recipients opted out of that topic are dropped at accept (like suppression hits), an RFC 8058 `List-Unsubscribe` header is added, and the literal tokens `{{{UNSUBSCRIBE_URL}}}` / `{{{RESEND_UNSUBSCRIBE_URL}}}` in the body are replaced with a per-recipient unsubscribe link.
- `attachments` — `[{ "filename": "invoice.pdf", "content": "<base64>", "content_type": "application/pdf" }]`. Only inline base64 `content` is accepted: a `path` URL is a 422 (never fetched — SSRF), and `content_id` (inline images) is a 422.
- `headers` — flat string map of extra message headers. Transport-owned names are rejected with 422 (case-insensitive): `From`, `To`, `Cc`, `Bcc`, `Reply-To`, `Subject`, `Content-Type`, `Return-Path`, `List-Unsubscribe`, anything `x-ses-*`, etc. Values must not contain control characters.

```sh
curl -X POST "$MILLIONSEND_BASE_URL/emails" \
  -H "Authorization: Bearer $MILLIONSEND_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "from": "Acme <billing@acme.dev>",
    "to": "user@example.com",
    "subject": "Your invoice",
    "text": "Attached.",
    "scheduled_at": "in 2 hours",
    "headers": { "X-Entity-Ref-ID": "inv_42" },
    "attachments": [{ "filename": "invoice.pdf", "content": "JVBERi0xLjQK...", "content_type": "application/pdf" }]
  }'
```

## Idempotency

Pass an `Idempotency-Key` header to make retries safe:

- Same key + same payload → replays the original response (no second send).
- Same key + different payload → 409 `invalid_idempotent_request`.
- Still processing → 409 `concurrent_idempotent_requests` (retry later).

A relative `scheduled_at` is stored as the raw string, so retries hash identically.

## Batch — POST /emails/batch

Body is a **bare JSON array** of send objects, 1–100 items. Items take the full single-send shape, including `attachments` and `headers`. `Idempotency-Key` works here too. Two validation modes:

- **Strict (default)** — all-or-nothing: any invalid item (bad field, unverified domain, all recipients suppressed) fails the whole batch with `emails.<index>: <reason>` and nothing is sent. Response: `{ "data": [{ "id": "..." }, ...] }` in input order.
- **Permissive** — send `x-batch-validation: permissive`: the valid subset is accepted and each failure is reported by input index: `{ "data": [...], "errors": [{ "index": 2, "message": "..." }] }` (`errors` is present, possibly empty, in permissive mode).

## Get, reschedule, cancel

- `GET /emails?limit=&after=|before=` — keyset pagination (`limit` 1–100 default 20, cursors are item ids, `after`/`before` mutually exclusive) → `{ object: "list", data: [...], has_more }`.
- `GET /emails/{id}` → `{ object: "email", id, from, to, subject, html, text, last_event, scheduled_at, ... }`. `last_event` is the delivery status (`queued`, `sent`, `delivered`, `bounced`, `complained`, ...).
- `PATCH /emails/{id}` with `{ "scheduled_at": "..." }` (ISO or relative) — reschedules a **not-yet-sent scheduled** email → `{ object: "email", id }`.
- `POST /emails/{id}/cancel` — cancels a scheduled, not-yet-sent email only (immediate sends cannot be canceled). Returns `{ object: "email", id }`.

## SDKs

Node (`npm install millionsend`, mirrors the `resend` package):

```ts
import { MillionSend } from "millionsend";

const ms = new MillionSend("ms_123", { baseUrl: "https://api.millionsend.com" });
const { data, error } = await ms.emails.send({
  from: "Acme <onboarding@acme.dev>",
  to: "user@example.com",
  subject: "Welcome",
  html: "<p>It works!</p>",
});
if (error) console.error(error.name, error.message); // never throws for API errors
else console.log(data.id);
```

Batch: `await ms.batch.send([{...}, {...}])`.

Python (`pip install millionsend`, mirrors the `resend` package; raises on error):

```python
import millionsend

millionsend.api_key = "ms_123"
millionsend.base_url = "https://api.millionsend.com"

email = millionsend.Emails.send({
    "from": "Acme <onboarding@acme.dev>",
    "to": "user@example.com",
    "subject": "Welcome",
    "html": "<p>It works!</p>",
})
print(email.id)
```

Both SDKs read `MILLIONSEND_API_KEY` and `MILLIONSEND_BASE_URL` from the environment. SDKs also exist for Go, Ruby, PHP, Java, .NET, Rust, and Elixir under github.com/MillionSend.

## Errors

Every error body is `{ "statusCode": <int>, "name": "<code>", "message": "<human text>" }`. Codes to branch on: `missing_api_key` / `invalid_api_key` (401), `restricted_api_key` (403), `sending_paused` (403, bounce/complaint rate crossed the SES pause line), `not_found` (404), `validation_error` (422 — also used for 409 "Contact already exists"), `invalid_idempotent_request` / `concurrent_idempotent_requests` (409), `all_recipients_suppressed` (422 — every `to` recipient is on the suppression list or opted out of the send's `topic_id`; message `All recipients are suppressed`), `rate_limit_exceeded` (429 — per-key limit, default 600 requests/minute; honor the `retry-after` header, in seconds), `daily_quota_exceeded` (429 — the day's quota is spent and the parked backlog is full; retry after the UTC day rolls over), `internal_server_error` (500). Hard-bounced or complained addresses are auto-suppressed.

## SMTP relay (for software that only speaks SMTP)

Host: the instance host, port `2587`. Username: `millionsend` (fixed). Password: an `ms_` API key. STARTTLS is offered (and required before AUTH) when the server has a TLS keypair configured; without one the relay only runs if the operator explicitly allowed insecure auth for a private network. Same pipeline as `POST /emails` (domain verification, suppression, events).
