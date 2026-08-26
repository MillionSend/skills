---
name: millionsend-mcp
description: Connect an AI agent to MillionSend's built-in MCP server (Streamable HTTP at /mcp, OAuth sign-in, team-scoped tokens) and use its tools to send emails and manage contacts, segments, topics, broadcasts, and domains. Use when adding MillionSend to Claude Code, Claude Desktop, Cursor, or VS Code via MCP, troubleshooting the OAuth connection, or deciding which MCP tool and permission scope a task needs.
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

- The access token is bound to the one team chosen at consent. To act on a different team, sign in again from the client.
- Grants are listed under **Settings → Connected apps** in the dashboard, where they can be revoked. Revocation takes effect at the client's next token refresh, within 15 minutes; a member removed from the team loses access immediately.
- MCP calls share the API's per-minute rate limit (429 with `retry-after` when exceeded).

## Tools

Each tool requires a permission scope granted at consent; clients only see the tools their scopes cover. If an expected tool is missing, reconnect and grant the scope.

| Tool | Scope | Notes |
| --- | --- | --- |
| `list_emails` | `emails:read` | Keyset pagination (`limit`, `after`/`before`). |
| `get_email` | `emails:read` | Delivery status in `last_event`. |
| `list_contacts` | `audience:read` | Pass `segment_id` for one segment's members. |
| `get_contact` | `audience:read` | By contact id **or** email address. |
| `list_segments` | `audience:read` | The targets broadcasts are sent to. |
| `list_topics` | `audience:read` | Subscription topics. |
| `list_domains` | `domains:read` | Verification status; absent on instances without SES configured. |
| `send_email` | `emails:send` | Full `POST /emails` shape incl. `scheduled_at`, `topic_id`, `attachments`, `headers`. |
| `create_contact` | `audience:write` | Inline `segments` and `topics` supported; 409 on duplicate email. |
| `update_contact` | `audience:write` | Name, `properties`, `unsubscribed`; omitted fields unchanged. |
| `add_contact_to_segment` | `audience:write` | Idempotent. |
| `create_broadcast` | `broadcasts:write` | Draft by default; `send: true` sends immediately. |
| `send_broadcast` | `broadcasts:write` | Send a draft now or with `scheduled_at`. |

Errors surface as the REST API's `{ statusCode, name, message }` bodies — an unverified sender domain fails `send_email` exactly as it fails `POST /emails` (422); a suppressed-only recipient list, topic opt-outs, and `sending_paused` behave identically. Anything the tools don't cover (deleting contacts, webhooks, API keys, domain creation) is REST-only — see the other millionsend skills.
