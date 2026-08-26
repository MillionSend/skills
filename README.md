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

## Skills

| Skill | What it covers |
| --- | --- |
| [millionsend-send-email](skills/millionsend-send-email/SKILL.md) | `POST /emails` incl. attachments, headers, `topic_id`, ISO + relative `scheduled_at`, idempotency, batch (strict/permissive), reschedule/cancel, SDKs, SMTP relay |
| [millionsend-contacts](skills/millionsend-contacts/SKILL.md) | Team-global contacts, typed `/contact-properties`, segments (filters and manual membership), topics + subscriptions, unsubscribe semantics |
| [millionsend-broadcasts](skills/millionsend-broadcasts/SKILL.md) | Campaigns: draft / `send: true` / schedule, `preview_text`, segment ∩ topic targeting, `{{{FIRST_NAME\|fallback}}}` merge fields, `{{{UNSUBSCRIBE_URL}}}` |
| [millionsend-domains](skills/millionsend-domains/SKILL.md) | `/domains` create → DNS records → verify, tracking toggles, plus `/api-keys` (permissions, domain-scoped keys) |
| [millionsend-webhooks](skills/millionsend-webhooks/SKILL.md) | `/webhooks` CRUD, Standard Webhooks signature verification (`webhook-signature: v1,<hmac>`), retries, receiver checklist |
| [millionsend-self-host](skills/millionsend-self-host/SKILL.md) | Stand up an instance: `npx @millionsend/setup`, Docker Compose, AWS SES/SNS/SQS events, DNS, SMTP relay |
| [millionsend-migrate-from-resend](skills/millionsend-migrate-from-resend/SKILL.md) | Zero-code migration via `RESEND_BASE_URL`/`RESEND_API_URL`, SDK swap table for 9 languages, SMTP swap, the deliberate deltas |
| [millionsend-mcp](skills/millionsend-mcp/SKILL.md) | Connect Claude Code / Claude Desktop / Cursor / VS Code to the built-in MCP server (`/mcp`, OAuth, team-scoped) and use its tools |

## License

MIT — see [LICENSE](LICENSE). (The MillionSend application itself is AGPL-3.0; these skills are documentation and are deliberately permissive.)
