---
name: millionsend-self-host
description: Stand up a self-hosted MillionSend instance (open-source Resend alternative on AWS SES) — Docker Compose, the npx @millionsend/setup wizard, AWS SES/SNS/SQS event pipeline, domain DNS/DKIM verification, and the SMTP relay. Use when installing, configuring, or troubleshooting a MillionSend deployment.
---

# Self-host MillionSend

MillionSend sends through the user's own AWS SES account. Two containers: Postgres plus one app container running the API (port 3001), worker, and web dashboard (port 3000). Migrations run automatically on boot.

Prerequisites: Docker with Compose; an AWS account with SES access in the chosen region (SES **sandbox** accounts can only send to verified recipients — request production access to send to anyone); a sending domain the user controls.

## Fastest path — the setup wizard

One command in an empty directory (Node 18+):

```sh
mkdir millionsend && cd millionsend
npx @millionsend/setup
```

The wizard creates `.env` with generated secrets, optionally provisions the AWS resources (IAM policy + user + access key, SNS event topic, SES configuration set; adds an SQS queue when there is no public https URL), downloads the compose file, and runs `docker compose up -d`. Every step is skippable and safe to re-run; `--dry-run` prints the plan and touches nothing. Run it wherever AWS **admin** credentials live (laptop is fine) — the server itself never needs admin credentials; `teardown` deletes everything it created. No Node on the server? The same CLI ships inside the image: `docker run --rm -it --user root -v ~/.aws:/root/.aws ghcr.io/millionsend/millionsend:edge setup`.

Dashboard: http://localhost:3000 · API: http://localhost:3001.

## Manual path

```sh
mkdir millionsend && cd millionsend
curl -O https://raw.githubusercontent.com/MillionSend/millionsend/main/deploy/docker-compose.yml
curl -o .env https://raw.githubusercontent.com/MillionSend/millionsend/main/.env.example
# edit .env, then:
docker compose up -d
```

Uses the prebuilt multi-arch image `ghcr.io/millionsend/millionsend:edge` (`:edge` is a moving head — pin a version tag for production). From source instead: clone github.com/MillionSend/millionsend and `docker compose up --build -d`.

Required `.env` values (everything else defaults to a working local setup):

- `MASTER_ENCRYPTION_KEY`, `BETTER_AUTH_SECRET` — `openssl rand -base64 32` each. The master key encrypts email bodies at rest — back it up with the database.
- `APP_BASE_URL` — the exact URL the dashboard is opened at (default `http://localhost:3000`). It is the only origin sign-in accepts and is baked into unsubscribe/tracking links; a mismatch breaks login and blocks broadcast sends.
- `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` — or rely on the default AWS credential chain. SES sandbox accounts must keep `SES_MAX_SEND_RATE=1`.

## SES events (bounces, complaints, deliveries, opens, clicks)

The setup wizard always configures this. Two transports:

- Public https `APP_BASE_URL` → SNS subscription pushing to `https://<your-host>/ses/events` (confirms itself once the app runs with `SNS_TOPIC_ARNS` set; if pending, use "Request confirmation" in the SNS console).
- Otherwise → an SQS queue (`millionsend-events`) the worker long-polls; its URL goes in `.env` as `SQS_QUEUE_URL`.

Manual equivalent: SNS standard topic (same region as SES) → its ARN in `SNS_TOPIC_ARNS` (an allowlist — events from other topics are rejected); an SES **configuration set** with an event destination to that topic (event types: Send, Delivery, Delivery Delay, Bounce, Complaint, Open, Click, Reject, Rendering Failure) → name in `SES_CONFIGURATION_SET`. Restart after setting them. Without `SES_CONFIGURATION_SET`, sends go out but emit **no events** (no delivery tracking, no webhooks, no automatic suppression). The dashboard's Settings → SES page also offers a CloudFormation quick-create link.

## Domain verification (DNS)

Done from the dashboard after boot: Domains → add domain → it shows the DKIM CNAME records (SES-managed DKIM, or bring your own key — BYODKIM) to create at the DNS provider. Sending from a domain is refused until it shows **verified**.

## SMTP relay

For software that only speaks SMTP. Host: the Docker host; port `2587` (`SMTP_PORT` to change); username `millionsend` (fixed); password: an `ms_` API key from the dashboard. Plaintext by default — STARTTLS is offered (and required before AUTH) only when `SMTP_TLS_CERT_PATH`/`SMTP_TLS_KEY_PATH` point at a PEM keypair, so otherwise keep the port off the public internet. Same accept pipeline as `POST /emails`. The service is in the compose files; remove it if unused.

## Operations gotchas

- **First user to register becomes the initial account**; registration then closes (`ALLOW_SIGNUP=true` to reopen). Keep port 3000 private unless signup is deliberately open.
- **Run exactly one worker container** — the SES rate limiter is in-memory, so N replicas send at N× the configured rate. Scale vertically. To split processes across containers set `PROCESS` to `api` | `worker` | `web` (default `all`).
- Send rate and email retention are dashboard settings (Settings → Instance; defaults 14/s and 30 days). `SES_MAX_SEND_RATE` / `EMAIL_RETENTION_DAYS` env vars still work as boot overrides.
- Ports are tunable: `WEB_PORT` (dashboard), `PORT` (API), `SMTP_PORT` (relay). If the dashboard moves, update `APP_BASE_URL` to match.
- Full reference: `SELF_HOSTING.md` in github.com/MillionSend/millionsend.
