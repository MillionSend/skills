---
name: millionsend-webhooks
description: Subscribe to and verify MillionSend webhook events (email.sent, delivered, bounced, complained, opened, clicked, delivery_delayed) — /webhooks CRUD via the REST API or the dashboard, Standard Webhooks signature verification (webhook-signature v1 HMAC), retries, and the receiver checklist. Use when creating webhook endpoints, building a webhook receiver for a MillionSend instance, or debugging failed deliveries.
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
- `events` — at least one of the 7 types actually emitted: `email.sent` · `email.delivered` · `email.delivery_delayed` · `email.bounced` · `email.complained` · `email.opened` · `email.clicked`. Any other name (e.g. Resend's `contact.created`) is a loud 422, not a subscription that never fires.
- `signing_secret` is returned on **create and get only**, never in list rows.

Other routes (full_access key required): `GET /webhooks` (keyset list; a row's `events: null` means "all events" — dashboard-created endpoints can be wired that way) · `GET /webhooks/{id}` (includes `signing_secret`) · `PATCH /webhooks/{id}` (any of `endpoint`, `events`, `status`: `enabled` | `disabled`) · `DELETE /webhooks/{id}` → `{ ..., "deleted": true }`. The dashboard's Webhooks page manages the same endpoints.

Opens/clicks and bounce/complaint events require the SES event pipeline on the instance (`SES_CONFIGURATION_SET` + SNS/SQS — see the millionsend-self-host skill); cloud has it wired already.

## Delivery format

Each delivery is an HTTP POST with a JSON body and three headers:

- `webhook-id` — unique message id (`msg_<uuid>`); stable across retries of the same event → use it for dedupe.
- `webhook-timestamp` — unix **seconds**.
- `webhook-signature` — `v1,<base64 HMAC-SHA256>`; may contain several space-separated `v1,...` candidates (accept if any matches).

(The header names are the Standard Webhooks defaults, **not** `svix-*` — code that reads `svix-signature` by name must switch; the svix/standardwebhooks libraries handle both.)

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
