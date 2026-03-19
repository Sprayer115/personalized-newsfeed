# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this project is

A self-hosted personalized newsfeed stack. No application code — infrastructure only. The deliverables are configuration files deployed via Docker Compose.

## Stack

| Service | Image | Role |
|---|---|---|
| FreshRSS | `freshrss/freshrss:latest` | RSS/Atom aggregator, web UI, Google Reader API backend |
| PostgreSQL | `postgres:15-alpine` | FreshRSS database |
| ntfy | `binwiederhier/ntfy:latest` | Push notification broker |

SSL termination and public exposure are handled by **nginx Proxy Manager** (running separately, not in this compose file). Neither FreshRSS nor ntfy publish ports directly.

## Repository layout

```
docker-compose.yml   — all three services, volumes, network
ntfy/server.yml      — ntfy server config (bind-mounted into container at /etc/ntfy)
.env.example         — required environment variable template
feeds.md             — all RSS feed URLs organized by category
```

## Deploying

```bash
cp .env.example .env
# Fill in FRESHRSS_DB_PASSWORD and TZ
docker compose up -d
```

FreshRSS database connection is configured through the **web installer UI** (not env vars) — host `freshrss-db`, port `5432`, db/user `freshrss`.

## iOS client

**Reeder Classic** connects via the Google Reader compatible API:
- URL: `https://freshrss.yourdomain.com`
- Auth: FreshRSS username + password
- API must be enabled in FreshRSS Settings → Authentication → Allow API access

## Push notifications

FreshRSS has **no native ntfy integration**. Options:
- Community extension: [freshrss-notify](https://github.com/vert-fr/FreshRSS-Notify-Ext) (mount into `freshrss-extensions` volume)
- Custom cron/webhook sidecar (not in v1)

## Key constraints

- ntfy `auth-default-access: deny-all` must remain in `ntfy/server.yml` — server is public-facing.
- Do not add application code or extraction services in v1.
- Feed URLs are documented in `feeds.md` — update there when adding new sources.
