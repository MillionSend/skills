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
| [millionsend-send-email](skills/millionsend-send-email/SKILL.md) | Send transactional email: `POST /emails`, batch, scheduling, idempotency, error shapes, SDKs, SMTP relay |
| [millionsend-contacts](skills/millionsend-contacts/SKILL.md) | Team-global contacts, segments (saved filters at `/segments`), topics, unsubscribe semantics |
| [millionsend-broadcasts](skills/millionsend-broadcasts/SKILL.md) | Campaigns: draft/send/schedule/cancel, targeting, `{{{FIRST_NAME\|fallback}}}` merge fields, `{{{UNSUBSCRIBE_URL}}}` |
| [millionsend-self-host](skills/millionsend-self-host/SKILL.md) | Stand up an instance: `npx @millionsend/setup`, Docker Compose, AWS SES/SNS/SQS events, DNS, SMTP relay |
| [millionsend-webhooks](skills/millionsend-webhooks/SKILL.md) | Subscribe to events and verify Standard Webhooks signatures (`webhook-signature: v1,<hmac>`) |

## License

MIT — see [LICENSE](LICENSE). (The MillionSend application itself is AGPL-3.0; these skills are documentation and are deliberately permissive.)
