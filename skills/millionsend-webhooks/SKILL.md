---
name: millionsend-webhooks
description: Subscribe to and verify MillionSend webhook events (email.sent, delivered, bounced, complained, opened, clicked, delivery_delayed, plus team-level deliverability.* and quota.*) — /webhooks CRUD via the REST API or the dashboard, bring-your-own whsec_ signing_secret, Standard Webhooks signature verification (webhook-signature v1 HMAC, also sent as svix-*), retries, and the receiver checklist. Use when creating webhook endpoints, migrating a Resend/Svix receiver, building a webhook receiver for a MillionSend instance, or debugging failed deliveries.
---

# MillionSend webhooks

MillionSend signs webhooks with the **Standard Webhooks** spec — the same scheme behind Resend/Svix webhooks, so `standardwebhooks` and `svix` verification libraries work unchanged.

## Subscribe — POST /webhooks (or the dashboard)

```sh
curl -X POST "$MILLIONSEND_BASE_URL/webhooks" \
  -H "Authorization: Bearer $MILLIONSEND_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "endpoint": "https://example.com/webhooks/millionsend",
    "events": ["email.bounced", "email.complained"]
  }'
# → { "object": "webhook", "id": "<uuid>", "signing_secret": "whsec_..." }
```

- `endpoint` must be **https** (a signed customer-event payload must not travel plaintext).
- `events` — at least one of the types actually emitted: `email.sent` · `email.delivered` · `email.delivery_delayed` · `email.bounced` · `email.complained` · `email.opened` · `email.clicked`; the team-level `deliverability.warning` · `deliverability.paused` · `quota.warning` · `quota.reached` · `quota.paused` (no email in `data`; they describe the team's standing); and the audience events `contact.created` · `contact.updated` · `contact.deleted` · `contact.unsubscribed` · `contact.resubscribed` · `contact.topic_opt_in` · `contact.topic_opt_out` · `suppression.added` · `suppression.removed`. Contact events carry the contact in Resend's shape (`id`, `email`, `first_name`, `last_name`, `unsubscribed`, `created_at`, `updated_at`) plus `source` (`api` | `dashboard` | `hosted_page` | `one_click`); the topic pair adds `topic_id`/`topic_name`; suppression events carry `{ id, email, origin, source, created_at }` (`source: null` for SES-written bounces/complaints). `contact.deleted` has `email: "[erased]"` (deleting is an erasure) — key on `id`. Any other name (e.g. Resend's `domain.created`) is a loud 422, not a subscription that never fires.
- `signing_secret` (optional on create) — bring your own: `whsec_` + standard base64 (padded, `+`/`/` alphabet) of 24–64 bytes, the format Resend/Svix issue, so a receiver that already verifies with that secret needs no redeploy. Omit it and one is generated. Anything else → 422 `validation_error` with message `signing_secret must be whsec_ followed by base64 of 24-64 bytes`.
- `signing_secret` is returned on **create and get only**, never in list rows. Rotate it with `POST /webhooks/{id}/rotate` (body `{}` to mint, or `{ "signing_secret": "whsec_..." }` to bring your own; `overlap_hours` 0–72, default 24) → `{ "object": "webhook", "id", "signing_secret", "previous_secret_expires_at" }`. During the overlap every delivery carries **two** space-separated `v1,...` signatures (new first, then previous), so a receiver holding either verifies; switch at any point, after the window only the new one signs. `GET /webhooks/{id}` also reports `previous_secret_expires_at` (null when no window is open). MCP: `rotate_webhook_secret`.

Other routes (full_access key required): `GET /webhooks` (keyset list; a row's `events: null` means "all events" — dashboard-created endpoints can be wired that way) · `GET /webhooks/{id}` (includes `signing_secret`) · `PATCH /webhooks/{id}` (any of `endpoint`, `events`, `status`: `enabled` | `disabled`) · `DELETE /webhooks/{id}` → `{ ..., "deleted": true }`. The dashboard's Webhooks page manages the same endpoints.

Opens/clicks and bounce/complaint events require the SES event pipeline on the instance (`SES_CONFIGURATION_SET` + SNS/SQS — see the millionsend-self-host skill); cloud has it wired already.

## Delivery format

Each delivery is an HTTP POST with a JSON body and three signature headers:

- `webhook-id` — unique message id (`msg_<uuid>`); stable across retries of the same event → use it for dedupe.
- `webhook-timestamp` — unix **seconds**.
- `webhook-signature` — `v1,<base64 HMAC-SHA256>`; may contain several space-separated `v1,...` candidates (accept if any matches).

The same three values are also sent as `svix-id`, `svix-timestamp` and `svix-signature` (the names Resend's docs use) — one signature, two header names, so a receiver reading either family by literal name works unchanged, as do the `svix`/`standardwebhooks` libraries.

Payload (mirrors Resend's webhook event shape):

```json
{
  "type": "email.bounced",
  "created_at": "2026-08-17T12:00:00.000Z",
  "data": {
    "email_id": "<email uuid>",
    "from": "Acme <onboarding@acme.dev>",
    "to": ["user@example.com"],
    "subject": "Welcome",
    "created_at": "2026-08-17T12:00:00.000Z"
  }
}
```

Some events add extras under `data` (e.g. click URL, bounce info).

## Verify the signature (always, before trusting the body)

Signed content is `` `${webhook-id}.${webhook-timestamp}.${rawBody}` ``; the HMAC key is the **base64-decoded** part of the secret after `whsec_`. Reject timestamps older/newer than 5 minutes. Use the raw request bytes — re-serializing the JSON breaks the signature.

Node (no dependency):

```js
import { createHmac, timingSafeEqual } from "node:crypto";

function verify(secret, headers, rawBody) {
  const ts = Number(headers["webhook-timestamp"]);
  if (!Number.isFinite(ts) || Math.abs(Date.now() / 1000 - ts) > 300) return false;
  const key = Buffer.from(secret.slice("whsec_".length), "base64");
  const expected = createHmac("sha256", key)
    .update(`${headers["webhook-id"]}.${ts}.${rawBody}`)
    .digest();
  return headers["webhook-signature"].split(" ").some((cand) => {
    const [version, sig] = cand.split(",", 2);
    if (version !== "v1" || !sig) return false;
    const given = Buffer.from(sig, "base64");
    return given.length === expected.length && timingSafeEqual(given, expected);
  });
}
```

Or with a library (identical wire format):

```python
# pip install standardwebhooks
from standardwebhooks import Webhook

wh = Webhook("whsec_...")
payload = wh.verify(raw_body, headers)  # raises on bad signature/timestamp
```

## Respond and retries

Return any **2xx** quickly; anything else (or a timeout) counts as a failure. Failed deliveries retry with Svix-style backoff: 5s, 5m, 30m, 2h, 5h, 10h — **6 attempts total**, then the delivery is marked exhausted. Each attempt re-signs with a fresh timestamp but keeps the same `webhook-id`, so idempotent handling keyed on `webhook-id` is safe. The dashboard's webhook detail page shows the per-delivery log (status code, response snippet, attempts) for debugging.

## Receiver checklist

1. Read the raw body before any JSON parsing middleware touches it.
2. Verify signature + timestamp; 401/400 on failure.
3. Dedupe on `webhook-id`.
4. Enqueue heavy work and return 2xx immediately.
5. Branch on `type`; treat unknown types as ignorable (new ones may be added).
