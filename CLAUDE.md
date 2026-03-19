# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this project is

A self-hosted personalized newsfeed stack. No application code — infrastructure only. The deliverables are configuration files deployed via Docker Compose.

## Stack

| Service | Image | Role |
|---|---|---|
| Miniflux | `miniflux/miniflux:latest` | RSS/Atom aggregator, web UI, API backend |
| PostgreSQL | `postgres:15-alpine` | Miniflux database (SQLite not supported by Miniflux) |
| ntfy | `binwiederhier/ntfy:latest` | Push notification broker |

SSL termination and public exposure are handled by **nginx Proxy Manager** (running separately, not in this compose file). Neither Miniflux nor ntfy publish ports directly.

## Repository layout

```
docker-compose.yml   — all three services, volumes, network
ntfy/server.yml      — ntfy server config (bind-mounted into container at /etc/ntfy)
.env.example         — required environment variable template
```

## Deploying

```bash
cp .env.example .env
# Fill in MINIFLUX_DB_PASSWORD, MINIFLUX_ADMIN_PASSWORD, MINIFLUX_BASE_URL
docker compose up -d
```

## Post-deploy configuration (manual, via web UIs)

**Miniflux** (`https://miniflux.yourdomain.com`):
- Settings → Integrations → ntfy: URL `https://ntfy.yourdomain.com`, topic `newsfeed`, publish token
- Settings → API Keys: generate a key for Reeder 5
- Feeds: add Reddit sources as `https://www.reddit.com/r/SUBREDDIT/.rss` — no credentials needed
- Per-feed: enable "Fetch original content" for full-text extraction via built-in scraper

**ntfy** (CLI, after container starts):
```bash
# Create publish-only token for Miniflux
docker exec -it <ntfy-container> ntfy token add --role=write miniflux-publisher
# Create read token for iOS app
docker exec -it <ntfy-container> ntfy token add --role=read ios-reader
```

**nginx Proxy Manager** (web UI):
- Proxy host: `miniflux.yourdomain.com` → `miniflux:8080`, Let's Encrypt SSL
- Proxy host: `ntfy.yourdomain.com` → `ntfy:80`, Let's Encrypt SSL
- NPM must be on the same Docker network as the `newsfeed` network, or use the host IP

**Reeder 5** (iOS): Accounts → Add → Miniflux → server URL + API key (not username/password)

## Key constraints (from the spec)

- ntfy is a **temporary notification bridge**. The Miniflux REST API at `/v1` is the integration point for the future custom iOS app.
- Single ntfy topic `newsfeed` — no per-source topic splitting in v1.
- ntfy `auth-default-access: deny-all` must remain in `ntfy/server.yml` — server is public-facing.
- Do not add application code, custom webhooks, or extraction services — the spec explicitly excludes them from v1.
