# MillionSend Agent Skills

[Agent Skills](https://skills.sh) for [MillionSend](https://github.com/MillionSend/millionsend) — the open-source, self-hostable, Resend-compatible email platform on AWS SES. Each skill teaches a coding agent (Claude Code, Cursor, and other agents supporting the Agent Skills format) how to work with a real MillionSend instance: exact endpoints, request shapes, error codes, and gotchas, all verified against the API source.

## Install

All skills:

```sh
npx skills add MillionSend/skills
```

One skill:

```sh
npx skills add MillionSend/skills --skill millionsend-send-email
```

## Plugin (Cursor and Grok Bot)

This repo is also a [Cursor plugin](https://cursor.com/docs/plugins): `.cursor-plugin/plugin.json` is the manifest, `mcp.json` adds the hosted MCP server (`https://api.millionsend.com/mcp`), and the skills load from `skills/`.

Cursor, from a local clone:

```sh
git clone https://github.com/MillionSend/skills ~/.cursor/plugins/local/millionsend
```

Then run **Developer: Reload Window** (or restart Cursor) and check **Customize** for the skills and the `millionsend` MCP server. Clone into that folder rather than symlinking it: Cursor skips symlinks that point outside `~/.cursor/plugins/local`.

Grok Bot: in a chat, say "Add this MCP server: https://api.millionsend.com/mcp".

Sign-in is OAuth through app.millionsend.com: on first connect you pick the team (or all teams) and the permissions the agent gets. There is no API key to paste and no local process to run. Revoke access under **Settings → Connected apps** in the MillionSend dashboard.

Self-hosted: point `mcp.json` at your instance's own `/mcp` URL (shown under **Settings → MCP**).

## Skills

| Skill | What it covers |
| --- | --- |
| [millionsend-send-email](skills/millionsend-send-email/SKILL.md) | `POST /emails` incl. attachments, headers, `topic_id`, ISO + relative `scheduled_at`, idempotency, batch (strict/permissive), reschedule/cancel, SDKs, SMTP relay |
| [millionsend-contacts](skills/millionsend-contacts/SKILL.md) | Team-global contacts, bulk `POST /contacts/batch` (skip/upsert), typed `/contact-properties`, segments (filters and manual membership), topics + subscriptions, unsubscribe semantics, `/suppressions` (batch add/remove) |
| [millionsend-broadcasts](skills/millionsend-broadcasts/SKILL.md) | Campaigns: draft / `send: true` / schedule, `preview_text`, segment ∩ topic targeting, `{{{FIRST_NAME\|fallback}}}` merge fields, `{{{UNSUBSCRIBE_URL}}}`, `/templates` (aliases, no publish cycle) |
| [millionsend-domains](skills/millionsend-domains/SKILL.md) | `/domains` create → DNS records → verify, tracking toggles, the read-only shared inbox subdomain, plus `/api-keys` (permissions, domain-scoped keys) |
| [millionsend-webhooks](skills/millionsend-webhooks/SKILL.md) | `/webhooks` CRUD, event types incl. the `email.failed` / `email.suppressed` reasons, bring-your-own `whsec_` secret, Standard Webhooks signature verification (`webhook-*` and `svix-*` headers), retries, receiver checklist |
| [millionsend-self-host](skills/millionsend-self-host/SKILL.md) | Stand up an instance: `npx @millionsend/setup`, Docker Compose, AWS SES/SNS/SQS events, DNS, SMTP relay |
| [millionsend-migrate-from-resend](skills/millionsend-migrate-from-resend/SKILL.md) | `npx @millionsend/cli migrate --from resend` (read-only against Resend; plan / apply / status / rollback; re-run to sync), then zero-code cutover via `RESEND_BASE_URL`/`RESEND_API_URL`, SDK swap table for 9 languages, SMTP swap, the deliberate deltas |
| [millionsend-mcp](skills/millionsend-mcp/SKILL.md) | Connect Claude Code / Claude Desktop / Cursor / VS Code to the built-in MCP server (`/mcp`, OAuth, team-scoped) and use its tools, incl. an agent setting up its own inbox on the shared subdomain |

## License

MIT — see [LICENSE](LICENSE). (The MillionSend application itself is AGPL-3.0; these skills are documentation and are deliberately permissive.)
