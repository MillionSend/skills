---
name: millionsend-domains
description: Verify sending domains and manage API keys on MillionSend (Resend-compatible email API) — /domains create → DNS records → verify, open/click tracking toggles, custom return path, and /api-keys CRUD with full/sending permissions and domain-scoped keys. Use when setting up everything needed before the first send, adding a domain, fetching its DNS/DKIM records, checking verification, or creating/rotating API keys via the REST API.
---

# MillionSend domains and API keys

Everything a team needs before its first send: a verified sending domain and an API key. All routes require a **full_access** `ms_` key (`Authorization: Bearer ms_...`). Base URL: `https://api.millionsend.com` (cloud) or the instance's API origin (default `http://localhost:3001`).

## Add a domain — POST /domains

```sh
curl -X POST "$MILLIONSEND_BASE_URL/domains" \
  -H "Authorization: Bearer $MILLIONSEND_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "name": "acme.dev", "custom_return_path": "send" }'
```

- `name` — **lowercase** registrable hostname (uppercase is a 422, not normalized: SES registers the identity exactly as typed). Already added → 409 with name `conflict`.
- `region` — optional. Each instance serves one SES region (MillionSend Cloud: `sa-east-1`) and uses it when the field is omitted; any other value is a 422 naming the served region. Don't copy a region from another provider.
- `custom_return_path` — the MAIL FROM subdomain (Resend's name for it), default `"send"`.

Response: the domain object (`id`, `name`, `status`, `region`, `open_tracking`, `click_tracking`, `tracking_subdomain`, `capabilities`) plus `records[]` — the DNS rows to create at the DNS provider, in the Resend SDK shape `{ record, name, type, ttl, status, value, priority? }`: DKIM CNAMEs, the MAIL FROM MX/TXT pair, and a recommended DMARC TXT row (a MillionSend superset — Resend doesn't include it).

## Check verification — POST /domains/{id}/verify

```sh
curl -X POST "$MILLIONSEND_BASE_URL/domains/4ef9a417-02e9-4d39-ad75-9611e0fcc33c/verify" \
  -H "Authorization: Bearer $MILLIONSEND_API_KEY"
# → { "object": "domain", "id": "..." }
```

Triggers a live SES + DNS check and updates the stored `status`; read the result back with `GET /domains/{id}` (which also returns `records[]` with per-record status). A worker cron re-checks periodically too. Sending from the domain is refused until `status` is `verified`.

Other routes: `GET /domains` (keyset list: `limit` 1–100, `after`/`before` cursors) · `GET /domains/{id}` · `DELETE /domains/{id}` → `{ "object": "domain", "id": ..., "deleted": true }`.

## Tracking — PATCH /domains/{id}

```sh
curl -X PATCH "$MILLIONSEND_BASE_URL/domains/$ID" \
  -H "Authorization: Bearer $MILLIONSEND_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "open_tracking": true, "click_tracking": true, "tracking_subdomain": "track" }'
```

- `open_tracking` / `click_tracking` — per-domain toggles for pixel/link-rewrite tracking.
- `tracking_subdomain` — branded tracking host label (`track` → `track.acme.dev`); `""` or `null` clears it.
- `tls` and `capabilities` (accepted by Resend's SDK types) are **not supported**: 422, never silently ignored.

## API keys — /api-keys

```sh
curl -X POST "$MILLIONSEND_BASE_URL/api-keys" \
  -H "Authorization: Bearer $MILLIONSEND_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "name": "ci-sender", "permission": "sending_access", "domain_id": "4ef9a417-02e9-4d39-ad75-9611e0fcc33c" }'
# → { "id": "<uuid>", "token": "ms_..." }
```

- `permission` — `full_access` (default; everything) or `sending_access` (confined to `/emails*`; any management route is 403 `restricted_api_key`).
- `domain_id` — optional; scopes the key to sending from that one domain. Must be a **verified** domain of the team (422 otherwise).
- The `token` (`ms_` prefix) is returned **only on create** — store it immediately. `GET /api-keys` lists metadata only (`id`, `name`, `created_at`, `last_used_at`; never tokens). `DELETE /api-keys/{id}` revokes → `{ "object": "api_key", "id": ..., "deleted": true }`.

Rate limit: 600 requests/minute per key by default → 429 `rate_limit_exceeded` with a `retry-after` header (seconds).

## SDK / errors

Every native MillionSend SDK from 0.4.0 wraps these routes with the Resend SDK's names — Node `ms.domains.create/list/get/verify/update/remove` and `ms.apiKeys.create/list/remove`, Python `millionsend.Domains.*` and `millionsend.ApiKeys.*`, PHP `$ms->domains->*` and `$ms->apiKeys->*`, and the same shape in Ruby, Go, Rust, Java, .NET and Elixir. On an older SDK, or with the `resend` SDK pointed at MillionSend (`RESEND_BASE_URL`), the upstream `domains.*` / `apiKeys.*` methods hit these same paths. Errors are `{ statusCode, name, message }`: `not_found` (404), `conflict` (409, duplicate domain), `validation_error` (422), `restricted_api_key` (403).
