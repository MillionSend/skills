---
name: millionsend-mcp
description: Connect an AI agent to MillionSend's built-in MCP server (Streamable HTTP at /mcp, OAuth sign-in, team-scoped tokens) and use its tools to send emails and manage contacts (one at a time or in batches), suppressions, segments, topics, broadcasts, templates, webhooks, API keys, and domains. Use when adding MillionSend to Claude Code, Claude Desktop, Cursor, or VS Code via MCP, troubleshooting the OAuth connection, or deciding which MCP tool and permission scope a task needs.
---

# MillionSend MCP server

Every MillionSend deployment ships an MCP (Model Context Protocol) server. Tool calls run through the exact same pipeline as the REST API — verified domains, suppression, topic opt-outs, quotas, and team scoping all apply unchanged, and tool results are the REST API's JSON responses.

## Server URL

Streamable HTTP at `/mcp` on the **API origin** (not the dashboard origin):

- Cloud: `https://api.millionsend.com/mcp`.
- Self-hosted: the instance's API origin plus `/mcp`, e.g. `https://api.acme.dev/mcp` or `http://localhost:3001/mcp`. The dashboard shows the exact URL under **Settings → MCP**.

## Connect a client

Claude Code:

```sh
claude mcp add --transport http millionsend https://api.millionsend.com/mcp
```

Then run `/mcp` inside Claude Code and pick `millionsend` to sign in.

Cursor (`.cursor/mcp.json`, or `~/.cursor/mcp.json` for every project):

```json
{ "mcpServers": { "millionsend": { "url": "https://api.millionsend.com/mcp" } } }
```

VS Code (`.vscode/mcp.json`):

```json
{ "servers": { "millionsend": { "type": "http", "url": "https://api.millionsend.com/mcp" } } }
```

Claude Desktop: add a custom connector (**Settings → Connectors → Add custom connector**) with the server URL; or in `claude_desktop_config.json`, which only launches stdio servers, bridge with `mcp-remote`:

```json
{
  "mcpServers": {
    "millionsend": { "command": "npx", "args": ["-y", "mcp-remote", "https://api.millionsend.com/mcp"] }
  }
}
```

Self-hosted: replace the URL with the instance's own `/mcp` URL.

## Authentication (OAuth, not API keys)

On first connect the client opens the browser: sign in to MillionSend, pick the **team** the client may act on, and choose the **permissions** (scopes) it gets. No secret ever lands in the client's config. The `/mcp` endpoint does not accept `ms_` API keys — headless/CI automation should call the REST API directly instead.

- A grant bound to one team only ever acts on that team. An **All teams** grant covers every team you belong to: every tool gains an optional `team_id` argument (default: your oldest team) and a `list_teams` tool lists the ids; admin-only tools are refused in a team where you are a plain member.
- Grants are listed under **Settings → Connected apps** in the dashboard, where they can be revoked. Revocation takes effect at the client's next token refresh, within 15 minutes; a member removed from the team loses access immediately.
- MCP calls share the API's per-minute rate limit (429 with `retry-after` when exceeded).

## Tools

Each tool requires a permission scope granted at consent; clients only see the tools their scopes cover. If an expected tool is missing, reconnect and grant the scope.

| Tool | Scope | Notes |
| --- | --- | --- |
| `list_emails` | `emails:read` | Keyset pagination (`limit`, `after`/`before`). |
| `get_email` | `emails:read` | Delivery status in `last_event`. |
| `get_usage` | `emails:read` | Plan, daily send/domain limits and today's accepted count — check before bulk work. |
| `list_contacts` | `audience:read` | Pass `segment_id` for one segment's members. |
| `get_contact` | `audience:read` | By contact id **or** email address. |
| `get_contact_topics` | `audience:read` | Every topic with the contact's effective subscription and an `explicit` flag; by id or email. |
| `list_segments` | `audience:read` | The targets broadcasts are sent to. |
| `list_topics` | `audience:read` | Subscription topics. |
| `list_suppressions` / `get_suppression` | `audience:read` | Blocked addresses; filter by `origin` (bounce, complaint, manual, unsubscribe). |
| `list_templates` / `get_template` | `templates:read` | Templates by id or alias. |
| `list_api_keys` | `api-keys:write` | Active keys, never their tokens. |
| `list_domains` | `domains:read` | Verification status; absent on instances without SES configured. |
| `send_email` | `emails:send` | Full `POST /emails` shape incl. `scheduled_at`, `topic_id`, `attachments`, `headers`. |
| `create_contact` | `audience:write` | Inline `segments` and `topics` supported; 409 on duplicate email. |
| `delete_contacts` | `audience:write` | Up to 1,000 contacts per call by `ids` or `emails`; an imported list can be undone in a few calls instead of thousands. |
| `create_contact_preferences_link` | `audience:write` | The contact's hosted preference-center URL (no expiry; hand it only to the contact). |
| `rotate_webhook_secret` | `webhooks:write` | **admin.** New `whsec_` secret; the previous one keeps signing for `overlap_hours` (default 24) so receivers switch without a gap. |
| `create_contact_batch` | `audience:write` | Up to 1,000 contacts per call — use it for imports. `on_conflict: skip\|upsert`, `validation: permissive` writes the valid subset and lists failures in `errors`. |
| `add_suppressions` / `remove_suppressions` / `delete_suppression` | `audience:write` | Batch block/unblock up to 1,000 addresses; `origin` on add keeps an import's reason (`unsubscribe` allowed). |
| `update_contact` | `audience:write` | Name, `properties`, `unsubscribed`; omitted fields unchanged. |
| `add_contact_to_segment` | `audience:write` | Idempotent. |
| `create_broadcast` | `broadcasts:write` | Draft by default; `send: true` sends immediately. |
| `send_broadcast` | `broadcasts:write` | Send a draft now or with `scheduled_at`. |
| `create_template` / `update_template` / `delete_template` | `templates:write` | Every save is live; no draft/publish cycle. |
| `create_api_key` / `revoke_api_key` | `api-keys:write` | **Owner/admin only.** The token is returned only by `create_api_key`, once — store it immediately. Lets an MCP-only onboarding mint the key the REST calls need. |
| `create_domain` / `update_domain` / `verify_domain` / `delete_domain` | `domains:write` | **Owner/admin only.** `region` is optional and must be the one region the instance serves. |

Errors surface as the REST API's `{ statusCode, name, message }` bodies — an unverified sender domain fails `send_email` exactly as it fails `POST /emails` (422); a suppressed-only recipient list, topic opt-outs, and `sending_paused` behave identically. The full tool list (including webhooks, contact properties and deletes) is under **Settings → MCP** in the dashboard and in the docs; tools that manage domains, webhooks and API keys are offered only to owners and admins.
