---
name: millionsend-send-email
description: Send transactional email through MillionSend (Resend-compatible email API on AWS SES) — POST /emails with attachments, custom headers, topic-scoped sends, ISO or relative scheduling, idempotency, batch (strict or permissive), reschedule/cancel, the SDKs, and the SMTP relay. Use when sending, scheduling, batching, fetching, rescheduling, or canceling emails on a MillionSend cloud or self-hosted instance.
---

# Send email with MillionSend

Base URL: `https://api.millionsend.com` (cloud) or the instance's own API origin (self-hosted; `http://localhost:3001` on a local compose setup). Authenticate with `Authorization: Bearer ms_...` — API keys are created in the dashboard or via `POST /api-keys` and start with `ms_`.

## Prerequisites

- The `from` address's domain must be a **verified domain** of the team (dashboard → Domains, or the `/domains` API — see the millionsend-domains skill). Otherwise: 422 `The <domain> domain is not verified for this team`.
- A `GET /domains` row with `shared: true` (the team's shared inbox subdomain, v0.14.0+) reads `verified`, yet sends only from its inboxes' own addresses: `from` must be one inbox's address exactly (no `+tag`; a display name only if it is the inbox's `from_name`), without `scheduled_at`, `reply_to`, `attachments` or `headers` (422 `validation_error` otherwise), and only where that inbox may start conversations (403 `inbox_send_policy` otherwise). `POST /emails/batch`, broadcasts and the SMTP relay always refuse it (403 `inbox_send_policy`; SMTP `550 5.7.1`). Send everything else from a `shared: false` row.
- A key with `sending_access` permission can only use `/emails*` routes. A domain-scoped key can only send from its assigned domain (403 `restricted_api_key` otherwise).

## Send one email — POST /emails

```sh
curl -X POST "$MILLIONSEND_BASE_URL/emails" \
  -H "Authorization: Bearer $MILLIONSEND_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "from": "Acme <onboarding@acme.dev>",
    "to": ["success@simulator.amazonses.com"],
    "subject": "Welcome",
    "html": "<p>It works!</p>"
  }'
# → { "id": "<email uuid>" }
```

Request fields (Resend-shaped, snake_case):

- `from` (required) — exactly one mailbox; display name allowed (`"Acme <x@acme.dev>"`).
- `to` (required) — string or array, max 50 recipients. `cc`, `bcc`, `reply_to` same shape. A `to`/`cc`/`bcc` address on a domain that can never receive mail — a reserved name (`.test`, `.invalid`, `.local`, …), `example.com`/`.net`/`.org`, or a TLD that doesn't exist (`gmail.con`) — is a 422 `validation_error` naming it. To test without emailing anyone, send to `success@simulator.amazonses.com` (the SES mailbox simulator; the email shows as `delivered`).
- `subject` (required, up to 998 characters; longer is a 422); at least one of `html` / `text` (required).
- `scheduled_at` — ISO 8601 **with offset** (`2026-09-01T10:00:00-03:00`) or a relative time: `in N min(s)/minute(s)/hour(s)/day(s)` ("in 5 mins", "in 2 hours", "in 1 day"). Max 30 days ahead.
- `tags` — `[{ "name": "...", "value": "..." }]`.
- `topic_id` — a topic UUID: recipients opted out of that topic are dropped like suppression hits, at accept and again when a scheduled or parked email goes out (see below). When the send goes to a single address that is a contact, an RFC 8058 `List-Unsubscribe` header is added and the literal tokens `{{{UNSUBSCRIBE_URL}}}` / `{{{RESEND_UNSUBSCRIBE_URL}}}` in the body are replaced with that contact's unsubscribe link. Any other topic send (more than one `to`/`cc`/`bcc` address, or a non-contact) gets no generated header and the tokens are removed: use a broadcast, or one send per recipient, when each person needs their own link.
- `attachments` — `[{ "filename": "invoice.pdf", "content": "<base64>", "content_type": "application/pdf" }]`. Only inline base64 `content` is accepted: a `path` URL is a 422 (never fetched — SSRF). `content_id` makes the part inline: the HTML shows it with `<img src="cid:<content_id>">`.
- `headers` — flat string map of extra message headers, allowlisted (case-insensitive): any `X-*` name except `X-SES-*` and `X-MillionSend-*`, plus `In-Reply-To`, `References`, `Importance`, `Priority`, `Comments`, `Keywords`, `Organization`, `OTP-Token`, and the one-click pair `List-Unsubscribe` (needs an `<https://…>` target) + `List-Unsubscribe-Post: List-Unsubscribe=One-Click`, sent together. Anything else, a vetted name given twice in different cases, or a value with control characters is a 422. Printable-ASCII values go out as given, quotes included, so a structured value like `OTP-Token: "123456"; origin="https://acme.dev"` arrives intact.

```sh
curl -X POST "$MILLIONSEND_BASE_URL/emails" \
  -H "Authorization: Bearer $MILLIONSEND_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "from": "Acme <billing@acme.dev>",
    "to": "success@simulator.amazonses.com",
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

## Suppressed and opted-out recipients

- At accept, `to`/`cc`/`bcc` recipients on the team's suppression list are dropped (bounce, complaint and manual entries always; `unsubscribe` entries only on a send with a `topic_id`), and so are recipients opted out of the send's `topic_id`.
- No `to` recipient left → 422 `all_recipients_suppressed`. When the suppression list alone emptied `to`, `POST /emails` still keeps a metadata-only email with status `suppressed` (no content, not counted toward the quota) and fires `email.suppressed`; each retry keeps another, even with the same `Idempotency-Key` (a refused send doesn't hold its key). Topic opt-outs, batch items and the SMTP relay keep nothing.
- An email that waits before it goes out, such as a scheduled or parked one, is checked again when it does: recipients suppressed or opted out of its `topic_id` in the meantime are dropped, and if no `to` recipient is left it ends `suppressed` without being sent and fires `email.suppressed` (reason `recipient_suppressed` or `topic_unsubscribed`; see the millionsend-webhooks skill).
- Dropping only some recipients, at accept or when the email goes out, fires nothing: the email goes to the rest.
- The all-suppressed record, the send-time topic re-check and `email.suppressed` need MillionSend v0.15.1 or later.

## Get, reschedule, cancel

- `GET /emails?limit=&after=|before=` — keyset pagination (`limit` 1–100 default 20, cursors are item ids, `after`/`before` mutually exclusive) → `{ object: "list", data: [...], has_more }`. Newest first since v0.13.0: `after` takes a page's last id and pages toward older emails; to sync, pass the newest id you stored as `before`, then each page's first id while `has_more` is `true`.
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
  to: "success@simulator.amazonses.com",
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
    "to": "success@simulator.amazonses.com",
    "subject": "Welcome",
    "html": "<p>It works!</p>",
})
print(email.id)
```

Both SDKs read `MILLIONSEND_API_KEY` and `MILLIONSEND_BASE_URL` from the environment. SDKs also exist for Go, Ruby, PHP, Java, .NET, Rust, and Elixir under github.com/MillionSend.

## Errors

Every error body is `{ "statusCode": <int>, "name": "<code>", "message": "<human text>" }`. Codes to branch on: `missing_api_key` / `invalid_api_key` (401), `restricted_api_key` (403), `sending_paused` (403, bounce/complaint rate crossed the SES pause line), `not_found` (404), `validation_error` (422 — also used for 409 "Contact already exists"), `invalid_idempotent_request` / `concurrent_idempotent_requests` (409), `all_recipients_suppressed` (422 — every `to` recipient is on the suppression list or opted out of the send's `topic_id`; message `All recipients are suppressed`), `rate_limit_exceeded` (429 — per-key limit, default 600 requests/minute; honor the `retry-after` header, in seconds), `daily_quota_exceeded` (429 — the day's quota is spent and the parked backlog is full; retry after the UTC day rolls over), `internal_server_error` (500). Hard-bounced or complained addresses are auto-suppressed.

## SMTP relay (for software that only speaks SMTP)

Host: the instance host, port `2587`. Username: `millionsend` (fixed). Password: an `ms_` API key. STARTTLS is offered (and required before AUTH) when the server has a TLS keypair configured; without one the relay only runs if the operator explicitly allowed insecure auth for a private network. Same pipeline as `POST /emails` (domain verification, suppression, events); a recipient on a domain that can never receive mail is refused at `RCPT TO` with `550 5.1.2`, and the other recipients are still accepted. Each message spends one unit of the key's per-minute budget, the same one its API calls spend (default 600), and a team sends at most 3,000 a minute: past either, the relay answers `451 4.7.1` at `MAIL FROM` with the seconds left, before the message is sent, so the client keeps it and retries. A subject over 998 characters is a `554`.
