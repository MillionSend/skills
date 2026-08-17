---
name: millionsend-send-email
description: Send transactional email through a MillionSend instance (self-hosted, Resend-compatible email API on AWS SES). Use when the task is to send, schedule, batch, fetch, or cancel emails via the MillionSend REST API, the millionsend Node/Python SDKs, or its SMTP relay.
---

# Send email with MillionSend

MillionSend is self-hosted: every request goes to the user's own instance, not a shared cloud URL. The API listens on port 3001 by default (`http://localhost:3001` locally; e.g. `https://mail.acme.dev` in production). Authenticate with `Authorization: Bearer ms_...` — API keys are created in the dashboard and start with `ms_`.

## Prerequisites

- The `from` address's domain must be a **verified domain** of the team (dashboard → Domains, DNS/DKIM verification). Otherwise: 422 `The <domain> domain is not verified for this team`.
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
- `scheduled_at` — ISO 8601 **with offset** (e.g. `2026-09-01T10:00:00-03:00`), max 30 days ahead. Natural language ("in 2 days") is rejected.
- `tags` — `[{ "name": "...", "value": "..." }]`.
- NOT yet supported (422, rejected loudly, never silently dropped): `attachments`, `headers`, `topic_id`.

## Idempotency

Pass an `Idempotency-Key` header to make retries safe:

- Same key + same payload → replays the original response (no second send).
- Same key + different payload → 409 `invalid_idempotent_request`.
- Still processing → 409 `concurrent_idempotent_requests` (retry later).

```sh
curl -X POST "$MILLIONSEND_BASE_URL/emails" \
  -H "Authorization: Bearer $MILLIONSEND_API_KEY" \
  -H "Idempotency-Key: signup-42" \
  -H "Content-Type: application/json" \
  -d '{"from":"Acme <onboarding@acme.dev>","to":"user@example.com","subject":"Welcome","text":"hi"}'
```

## Batch — POST /emails/batch

Body is a **bare JSON array** of send objects, 1–100 items, no attachments. Validation is strict and the batch is all-or-nothing: any invalid item (bad field, unverified domain, all recipients suppressed) fails the whole batch with `emails.<index>: <reason>` and nothing is sent. Response: `{ "data": [{ "id": "..." }, ...] }` in input order. `Idempotency-Key` works here too.

## Get and cancel

- `GET /emails/{id}` → `{ object: "email", id, from, to, subject, html, text, last_event, scheduled_at, ... }`. `last_event` is the delivery status (`queued`, `sent`, `delivered`, `bounced`, ...).
- `POST /emails/{id}/cancel` → cancels a **scheduled, not-yet-sent** email only (immediate sends cannot be canceled). Returns `{ object: "email", id }`.

## SDKs

Node (`npm install millionsend`, mirrors the `resend` package):

```ts
import { MillionSend } from "millionsend";

const ms = new MillionSend("ms_123", { baseUrl: "https://mail.acme.dev" });
const { data, error } = await ms.emails.send({
  from: "Acme <onboarding@acme.dev>",
  to: "user@example.com",
  subject: "Welcome",
  html: "<p>It works!</p>",
});
if (error) console.error(error.name, error.message); // never throws for API errors
else console.log(data.id);
```

Python (`pip install millionsend`, mirrors the `resend` package; raises on error):

```python
import millionsend

millionsend.api_key = "ms_123"
millionsend.base_url = "https://mail.acme.dev"

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

Every error body is `{ "statusCode": <int>, "name": "<code>", "message": "<human text>" }`. Codes to branch on: `missing_api_key` / `invalid_api_key` (401), `restricted_api_key` (403), `sending_paused` (403, bounce/complaint rate crossed the SES pause line), `not_found` (404), `validation_error` (422 — also used for 409 "Contact already exists"), `invalid_idempotent_request` / `concurrent_idempotent_requests` (409), `internal_server_error` (500). Hard-bounced or complained addresses are auto-suppressed; a send where every `to` is suppressed fails with 422 `All recipients are suppressed`.

## SMTP relay (for software that only speaks SMTP)

Host: the instance host, port `2587`. Username: `millionsend` (fixed). Password: an `ms_` API key. STARTTLS offered when the server has a TLS keypair configured. Same pipeline as `POST /emails` (domain verification, suppression, events).
