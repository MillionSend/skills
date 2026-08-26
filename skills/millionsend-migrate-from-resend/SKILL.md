---
name: millionsend-migrate-from-resend
description: Migrate an app from Resend to MillionSend (wire-compatible Resend alternative) with zero code changes — point the official Resend SDK at MillionSend via RESEND_BASE_URL/RESEND_API_URL, or swap to a native MillionSend SDK, or change two SMTP settings. Use when moving a codebase off Resend, debugging a Resend SDK pointed at a MillionSend instance, or checking which Resend features differ.
---

# Migrate from Resend to MillionSend

MillionSend's REST API is wire-compatible with Resend's documented v1 API: same paths, field names, response shapes, error format, and `Authorization: Bearer` auth. CI runs the official `resend` npm package against MillionSend to keep it that way. Three migration paths, fastest first.

The target base URL is `https://api.millionsend.com` (cloud) or the self-hosted instance's API origin. Create an `ms_` API key first (dashboard → API keys, or `POST /api-keys`); the sending domain must be re-verified on MillionSend (DKIM keys are per-provider — see the millionsend-domains skill for the DNS records).

## Path 1 — keep the Resend SDK, change env vars only

The official Resend SDKs accept a base-URL override. Resend SDKs don't validate the key prefix, so `ms_` keys pass through:

```sh
# Node (resend npm package)
RESEND_BASE_URL=https://api.millionsend.com
RESEND_API_KEY=ms_xxxxxxxxx

# Python (resend PyPI package)
RESEND_API_URL=https://api.millionsend.com
RESEND_API_KEY=ms_xxxxxxxxx
```

Node also takes a constructor option: `new Resend(apiKey, { baseUrl })`. For other languages' Resend SDKs, check whether they expose a base-URL override; if not, use Path 2.

## Path 2 — swap to the native MillionSend SDK

Each SDK mirrors the matching Resend SDK's method surface (`emails.send`, `batch.send`, `contacts`, `broadcasts`, `domains`-where-implemented, ...), so the migration is the import + client line:

| Language | Install | Client |
| --- | --- | --- |
| Node | `npm install millionsend` | `new MillionSend("ms_123", { baseUrl: "https://api.millionsend.com" })` |
| Python | `pip install millionsend` | `millionsend.api_key = "ms_123"; millionsend.base_url = "https://api.millionsend.com"` |
| Go | `go get github.com/MillionSend/millionsend-go` | `millionsend.NewClient("ms_123")` + `client.BaseURL = ...` |
| Ruby | `gem install millionsend` | `Millionsend.api_key = "ms_123"; Millionsend.base_url = ...` |
| PHP | `composer require millionsend/millionsend-php` | `MillionSend\MillionSend::client('ms_123', 'https://api.millionsend.com')` |
| Rust | `millionsend = "0.2"` (crates.io) | `MillionSend::with_base_url("ms_123", "https://api.millionsend.com")` |
| Java | `com.millionsend:millionsend-java` (Maven Central) | `new MillionSend("ms_123", "https://api.millionsend.com")` |
| .NET | `dotnet add package MillionSend` | `new MillionSendClient("ms_123", "https://api.millionsend.com")` |
| Elixir | `{:millionsend, "~> 0.2"}` (Hex) | `config :millionsend, MillionSend.Client, api_key: ..., base_url: ...` |

Node and Python also read `MILLIONSEND_API_KEY` / `MILLIONSEND_BASE_URL` from the environment.

## Path 3 — SMTP

Two settings change: host → the MillionSend instance host, port `2587`; username → `millionsend`; password → the `ms_` API key (same password-is-the-key convention as Resend).

## The gotchas (deliberate deltas — all loud, never silent)

1. **Webhook headers are `webhook-id` / `webhook-timestamp` / `webhook-signature`**, the Standard Webhooks names — not `svix-*`. The `svix` and `standardwebhooks` verification libraries handle both, so library-based receivers port unchanged; only code reading `svix-signature` by literal header name must be updated. Webhook endpoints must be re-created on MillionSend (new `whsec_` secrets), and only the 7 `email.*` event types exist — subscribing to `contact.created` etc. is a 422.
2. **Attachments are inline base64 only**: an attachment `path` URL is a 422 and is never fetched (SSRF); `content_id` (inline/cid images) is also a 422.
3. **Broadcast statuses**: internal scheduled/sending surface as `queued` (matching Resend's union), but `canceled` is emitted as-is — a superset value; treat unknown statuses as terminal. `POST /broadcasts/{id}/send` can also return 403 with name `sending_paused` (bounce/complaint rate hit the SES enforcement line) — a name outside Resend's error union, standard `{statusCode, name, message}` shape.
4. **Domains**: adding a duplicate domain is a 409 with name `conflict` (outside Resend's error-name union); domain-update `tls` and `capabilities` are 422 (unsupported, not ignored); domain `records[]` include an extra recommended DMARC row.
5. **Not implemented**: `contacts.topics.list` / `contacts.segments.list` (the nested GET lists — read a contact's data through `GET /contacts/{id}` and segment membership through `GET /segments/{id}/contacts` instead); templates, automations, and Resend's other undocumented surfaces. Audiences are pure aliases of segments (`/audiences/{id}/contacts[...]` works; the "audience id" is a segment id).
6. **Delivery events**: SES's `temporary_failure` maps to `pending` on the wire; hard bounces/complaints auto-suppress the address, and a send where every recipient is suppressed fails with 422 `All recipients are suppressed`.

Superset features (safe to ignore; use when wanted): typed `/contact-properties`, segment `filter` expressions, topic `visibility`, natural-language `scheduled_at` (Resend-compatible anyway), batch `x-batch-validation: permissive`, `GET /openapi.json` (the full OpenAPI document of the instance).

## Verify the migration

```sh
curl -X POST "$MILLIONSEND_BASE_URL/emails" \
  -H "Authorization: Bearer $MILLIONSEND_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "from": "Acme <onboarding@acme.dev>", "to": "you@example.com", "subject": "Migrated", "text": "Hello from MillionSend" }'
```

A 422 `The <domain> domain is not verified for this team` means the DNS re-verification step is still pending — everything else is already working.
