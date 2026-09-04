---
name: millionsend-migrate-from-resend
description: Migrate from Resend to MillionSend (wire-compatible Resend alternative) — move the account data with `npx @millionsend/cli migrate --from resend` (read-only against Resend, plan/apply/status/rollback, re-run to sync), then point the official Resend SDK at MillionSend via RESEND_BASE_URL/RESEND_API_URL, swap to a native MillionSend SDK, or change two SMTP settings. Use when moving an account or codebase off Resend, scripting the migration in CI, debugging a Resend SDK pointed at a MillionSend instance, or checking which Resend features differ.
---

# Migrate from Resend to MillionSend

MillionSend's REST API is wire-compatible with Resend's documented v1 API: same paths, field names, response shapes, error format, and `Authorization: Bearer` auth. CI runs the official `resend` npm package against MillionSend to keep it that way. A migration has two halves: the account data (one CLI command) and the code (three paths, fastest first).

The target base URL is `https://api.millionsend.com` (cloud) or the self-hosted instance's API origin. Create an `ms_` API key with full access first (dashboard → API keys, or `POST /api-keys`); the sending domain must be re-verified on MillionSend (DKIM keys are per-provider — see the millionsend-domains skill for the DNS records).

## Step 1 — move the account data with the CLI

`@millionsend/cli` (npm, Node ≥ 18, zero dependencies) reads the Resend account and recreates it on MillionSend. Resend is only ever read (`GET` to documented endpoints, 8 req/s by default — Resend's per-team limit is 10, shared with production sending); keys live in memory, are never written to a file and are redacted from logs; there is no telemetry.

```sh
npx @millionsend/cli migrate --from resend                          # interactive: connect → choose resources → plan → confirm → apply → summary
npx @millionsend/cli migrate plan --from resend [--out plan.json]   # read-only; exit 0 nothing to do, 2 changes, 1 error
npx @millionsend/cli migrate apply [plan.json] [--yes]              # apply a saved plan, or plan+apply in one go
npx @millionsend/cli migrate status                                 # what the last run created, what is left
npx @millionsend/cli migrate rollback [--yes]                       # deletes ONLY ids the CLI created, reverse dependency order
```

Credentials: `RESEND_API_KEY` (full access), `MILLIONSEND_API_KEY` (full access), `MILLIONSEND_BASE_URL` or `--to-url <url>` (self-hosted; Cloud is offered in a terminal). Each key also accepts `--from-key-stdin` / `--to-key-stdin` (first line source, second line target). Non-interactive is automatic when stdin is not a TTY (or with `--non-interactive` / `--json`): a missing input is exit 1 naming the env var to set; never pass keys as `--from-key` / `--to-key` arguments in CI (process lists). Exit codes: 0 ok, 1 error, 2 plan has changes (`plan` only), 3 partial.

What moves and how (re-run safe — the same command right before cutover syncs what changed):

- **Contacts** — `POST /contacts/batch` with `on_conflict=upsert`: email, names, `unsubscribed`, then, in a second pass (only when the account uses contact properties or topics), properties and topic subscriptions. Opt-outs are preserved; nobody is re-subscribed. `--skip enrichment` skips pass 2; it is resumable from `.millionsend/migrate-state.json`.
- **Topics / segments / properties / webhooks / templates / domains** — matched by name / name / key / endpoint / alias-then-name / name: missing → `+ create`, differing → `~ update` (PATCH), equal → `= unchanged`. Segment memberships follow the contacts.
- **Webhooks** — signing secrets are copied (`signing_secret` on `POST /webhooks`), so receivers keep verifying; `--fresh-webhook-secrets` mints new ones (shown once in the report). Events outside the 7 `email.*` types are dropped per webhook and listed as manual.
- **Suppressions** — `POST /suppressions/batch/add` with `origin` (bounce / complaint / manual) so history survives.
- **Domains** — created with region, custom return path and tracking toggles; the report prints a copy-ready table of MillionSend's DNS records per domain (the one unavoidable manual step). Both providers stay verified side by side.
- **Broadcasts** — drafts/scheduled import as drafts; sent ones only with `--include-sent`.
- **Templates** — name, alias, subject, html, text; `from` / `reply_to` / `variables` cannot be stored → `! manual` items.
- **Not moved** — API keys (Resend exposes names only → listed as a to-do), DKIM/DNS records, sent email history; audiences (deprecated in Resend) are skipped, segments cover them.

Flags that change what happens: `--only a,b` / `--skip a,b` (`domains, properties, topics, segments, contacts, enrichment, broadcasts, templates, webhooks, suppressions, api-keys`), `--rps 1..10`, `--on-conflict upsert|skip|error`, `--fresh` (ignore the state file), `--json` (JSON on stdout, progress on stderr), `--verbose`, `--report <file>`. Files land in `.millionsend/` (mode 0600, never a key; appended to `.gitignore` when one exists).

Before any write the plan checks the target's `GET /usage` (plan, `emails_per_day` / `domains` limits, cloud flag) and refuses or warns precisely ("7 domains to create; the Free plan allows 3").

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

1. **Webhook endpoints must be re-created on MillionSend** (`POST /webhooks`, dashboard, or the MCP's `create_webhook`), but the receiver does not change: every delivery carries the Standard Webhooks `webhook-id` / `webhook-timestamp` / `webhook-signature` headers **and** the same values as `svix-id` / `svix-timestamp` / `svix-signature`, and you can pass the existing `whsec_` as `signing_secret` on create (Resend returns it on `GET /webhooks/{id}`; malformed → 422) so the secret does not change either. Only the 7 `email.*` event types exist — subscribing to `contact.created` etc. is a 422.
2. **Attachments are inline base64 only**: an attachment `path` URL is a 422 and is never fetched (SSRF); `content_id` (inline/cid images) is also a 422.
3. **Broadcast statuses**: internal scheduled/sending surface as `queued` (matching Resend's union), but `canceled` is emitted as-is — a superset value; treat unknown statuses as terminal. `POST /broadcasts/{id}/send` can also return 403 with name `sending_paused` (bounce/complaint rate hit the SES enforcement line) — a name outside Resend's error union, standard `{statusCode, name, message}` shape.
4. **Domains**: adding a duplicate domain is a 409 with name `conflict` (outside Resend's error-name union); domain-update `tls` and `capabilities` are 422 (unsupported, not ignored); domain `records[]` include an extra recommended DMARC row.
5. **Not implemented**: `contacts.segments.list` (read segment membership through `GET /segments/{id}/contacts` instead; `GET /contacts/{id}/topics` does exist and returns every topic with the contact's effective `subscription` and an `explicit` flag); sending with a template (`template: { id, variables }` on `POST /emails` and `/emails/batch` is a 422, not silently dropped — pass `html`/`text`); automations and Resend's other undocumented surfaces. Audiences are pure aliases of segments (`/audiences/{id}/contacts[...]` works; the "audience id" is a segment id).
6. **Templates have no draft/publish cycle or versions**: every save is live, `status` is always `published`, `POST /templates/{id}/publish` is a no-op; `from`, `reply_to` and `variables` are rejected with 422 (not dropped) and read back as `null`/`null`/`[]`; every single-template route takes an id **or alias**.
7. **Suppressions**: same `/suppressions` wire as Resend, plus `origin: "unsubscribe"` (retained one-click opt-outs — a value outside the SDK's `SuppressionOrigin` union), batch bodies of up to 1000 (Resend: 100), an optional `origin: "bounce" | "complaint" | "manual"` on `POST /suppressions` and `batch/add` (default `manual`; `unsubscribe` → 422; absent from the SDK's `AddSuppressionOptions`, send it raw), and an idempotent `POST /suppressions` (an already-suppressed address returns its existing id, origin unchanged). The CLI carries Resend's bounce/complaint lists over with their origin; by hand, `POST /suppressions/batch/add` with `origin`.
8. **Delivery events**: SES's `temporary_failure` maps to `pending` on the wire; hard bounces/complaints auto-suppress the address, and a send where every recipient is suppressed (or opted out of its `topic_id`) fails with `422 all_recipients_suppressed` (message `All recipients are suppressed`).

Superset features (safe to ignore; use when wanted): `POST /contacts/batch?on_conflict=error|skip|upsert` (1–1000 contacts per call; never re-subscribes an unsubscribed contact), typed `/contact-properties`, segment `filter` expressions, topic `visibility`, natural-language `scheduled_at` (Resend-compatible anyway), batch `x-batch-validation: permissive` (emails and contacts), `GET /openapi.json` (the full OpenAPI document of the instance).

## Verify the migration

```sh
curl -X POST "$MILLIONSEND_BASE_URL/emails" \
  -H "Authorization: Bearer $MILLIONSEND_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "from": "Acme <onboarding@acme.dev>", "to": "you@example.com", "subject": "Migrated", "text": "Hello from MillionSend" }'
```

A 422 `The <domain> domain is not verified for this team` means the DNS re-verification step is still pending — everything else is already working.
