---
name: millionsend-webhooks
description: Subscribe to and verify MillionSend webhook events (email.scheduled, sent, delivered, delivery_delayed, bounced, complained, failed, suppressed, opened, clicked, plus team-level deliverability.* and quota.*) — /webhooks CRUD via the REST API or the dashboard, bring-your-own whsec_ signing_secret, Standard Webhooks signature verification (webhook-signature v1 HMAC, also sent as svix-*), retries, and the receiver checklist. Use when creating webhook endpoints, migrating a Resend/Svix receiver, building a webhook receiver for a MillionSend instance, or debugging failed deliveries.
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
- `events` — at least one of the types actually emitted: `email.scheduled` · `email.sent` · `email.delivered` · `email.delivery_delayed` · `email.bounced` · `email.complained` · `email.failed` · `email.suppressed` · `email.opened` · `email.clicked` · `email.received`; the team-level `deliverability.warning` · `deliverability.paused` · `quota.warning` · `quota.reached` · `quota.paused` (no email in `data`; they describe the team's standing); the audience events `contact.created` · `contact.updated` · `contact.deleted` · `contact.unsubscribed` · `contact.resubscribed` · `contact.topic_opt_in` · `contact.topic_opt_out` · `suppression.added` · `suppression.removed`; and the opt-in events `email.prefetched` · `inbox.created` · `inbox.updated` · `inbox.deleted` · `inbox.expired` · `inbox.thread.created` · `inbox.thread.folder.updated` · `inbox.thread.labels.updated` · `inbox.email.received` · `inbox.email.sent` · `inbox.email.quarantined` · `inbox.draft.created` · `inbox.draft.updated` · `inbox.draft.sent` · `inbox.draft.deleted`, which reach only endpoints that name them; every other type above also reaches an "all events" endpoint (`events: null`, below). `email.scheduled`, `email.failed` and `email.suppressed` need MillionSend v0.15.1 or later: an older server refuses a webhook naming them with 422 `validation_error`. Contact events carry the contact in Resend's shape (`id`, `email`, `first_name`, `last_name`, `unsubscribed`, `created_at`, `updated_at`) plus `source` (`api` | `dashboard` | `hosted_page` | `one_click`); the topic pair adds `topic_id`/`topic_name`; suppression events carry `{ id, email, origin, source, source_id, created_at }` (`source: null` for SES-written bounces/complaints; `source_id` is the email whose bounce or complaint created the entry, else `null`). `contact.deleted` carries the address; after an erasure (`DELETE /contacts/{id}?erase=true`, batch `erase: true`, or the dashboard's erase action) it reads `"[erased]"` — key on `id`. Any other name (e.g. Resend's `domain.created`) is a loud 422, not a subscription that never fires.
- `signing_secret` (optional on create) — bring your own: `whsec_` + standard base64 (padded, `+`/`/` alphabet) of 24–64 bytes, the format Resend/Svix issue, so a receiver that already verifies with that secret needs no redeploy. Omit it and one is generated. Anything else → 422 `validation_error` with message `signing_secret must be whsec_ followed by base64 of 24-64 bytes`.
- `signing_secret` is returned on **create and get only**, never in list rows. Rotate it with `POST /webhooks/{id}/rotate` (body `{}` to mint, or `{ "signing_secret": "whsec_..." }` to bring your own; `overlap_hours` 0–72, default 24) → `{ "object": "webhook", "id", "signing_secret", "previous_secret_expires_at" }`. During the overlap every delivery carries **two** space-separated `v1,...` signatures (new first, then previous), so a receiver holding either verifies; switch at any point, after the window only the new one signs. `GET /webhooks/{id}` also reports `previous_secret_expires_at` (null when no window is open). The MCP tools never return, accept or rotate a signing secret: use the dashboard or the REST routes here.

Other routes (full_access key required): `GET /webhooks` (keyset list; a row's `events: null` means "all events" — dashboard-created endpoints can be wired that way) · `GET /webhooks/{id}` (includes `signing_secret`) · `PATCH /webhooks/{id}` (any of `endpoint`, `events`, `status`: `enabled` | `disabled`) · `DELETE /webhooks/{id}` → `{ ..., "deleted": true }`. The dashboard's Webhooks page manages the same endpoints. `GET /webhooks` runs newest first since v0.13.0: `after` takes a page's last id and pages toward older endpoints; to sync, pass the newest id you stored as `before`, then each page's first id while `has_more` is `true`.

`email.delivered`, `email.delivery_delayed`, `email.bounced` and `email.complained`, and `email.failed` for a message SES rejects after `email.sent`, require the SES event pipeline on the instance (`SES_CONFIGURATION_SET` + SNS/SQS — see the millionsend-self-host skill); cloud has it wired already. Opens and clicks are tracked by MillionSend itself.

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

Every email event's `data` carries those five fields (`created_at` is when the event happened); some add extras (e.g. click URL, bounce info), including:

- `email.scheduled` — none. Fired when an email is accepted with a future `scheduled_at` (read the time with `GET /emails/{id}`); rescheduling with `PATCH /emails/{id}` fires nothing.
- `email.failed` — `failed: { reason }`. The email failed for good: it won't be sent, or SES refused it after `email.sent`. `reason`: `domain_not_verified` · `domain_deleted` · `body_missing` · `body_unreadable` · `message_rejected` · `retries_exhausted` · `send_interrupted` · `provider_error` (the catch-all).
- `email.suppressed` — `suppressed: { type, message, reason }`, always-present strings. The email won't be sent: the suppression list, an unsubscribe or a topic opt-out stopped it, or its contact, broadcast or topic was deleted or changed before it went out. `type`: `OnAccountSuppressionList` (Resend's value) · `Unsubscribed` · `Ineligible`; `reason`: `recipient_suppressed` · `contact_unsubscribed` · `topic_unsubscribed` · `contact_missing` · `contact_recipient_mismatch` · `broadcast_missing` · `topic_missing` · `ineligible` (the fallback). A `POST /emails` that the suppression list alone refuses with 422 `all_recipients_suppressed` fires it too, for the metadata-only `suppressed` email it keeps (see the millionsend-send-email skill). Dropping only some recipients, at accept or at send time, fires nothing, and neither does canceling a broadcast.

Treat `failed.reason` and all three `suppressed` values as open sets.

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
