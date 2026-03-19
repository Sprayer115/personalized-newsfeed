# Personalized Newsfeed — Self-Hosted

Self-hosted RSS newsfeed stack with push notifications and a native iOS reader.

| Component | Role |
|---|---|
| **Miniflux** | Feed aggregator, web UI, REST API |
| **PostgreSQL** | Miniflux database |
| **ntfy** | Push notifications to iOS |
| **Reeder 5** (iOS) | Native feed reader via Miniflux API |
| **nginx Proxy Manager** | HTTPS termination (running separately) |

---

## Prerequisites

- Docker + Docker Compose installed on the server
- nginx Proxy Manager already running (with a Docker network you can attach to)
- Two DNS records pointing at your server:
  - `miniflux.yourdomain.com`
  - `ntfy.yourdomain.com`

---

## 1. Clone the repo

```bash
git clone <repo-url> newsfeed
cd newsfeed
```

---

## 2. Configure environment

```bash
cp .env.example .env
```

Edit `.env`:

```env
MINIFLUX_DB_PASSWORD=<strong unique password>
MINIFLUX_ADMIN_PASSWORD=<strong unique password>
MINIFLUX_BASE_URL=https://miniflux.yourdomain.com
```

---

## 3. Configure ntfy

Edit `ntfy/server.yml` and set your domain:

```yaml
base-url: https://ntfy.yourdomain.com
```

The other values (`cache-file`, `auth-file`, `auth-default-access`) should stay as-is.

---

## 4. Connect to nginx Proxy Manager's Docker network

NPM needs to reach the `miniflux` and `ntfy` containers. Either:

**Option A — attach this stack to NPM's existing network:**

Find your NPM network name:
```bash
docker network ls
```

Add it as an external network in `docker-compose.yml` under the `ntfy` and `miniflux` services, and declare it at the bottom:

```yaml
networks:
  newsfeed:
    driver: bridge
  npm_network:        # replace with your actual NPM network name
    external: true
```

Then add `npm_network` to both `miniflux` and `ntfy` service network lists.

**Option B — forward by host IP:**

In NPM, use the server's host IP and the internal ports (`8080` for Miniflux, `80` for ntfy) instead of container names.

---

## 5. Start the stack

```bash
docker compose up -d
```

Verify all three containers are running:
```bash
docker compose ps
```

---

## 6. Set up nginx Proxy Manager proxy hosts

In the NPM web UI, create two **Proxy Hosts**:

| Domain | Forward Hostname | Forward Port | SSL |
|---|---|---|---|
| `miniflux.yourdomain.com` | `miniflux` | `8080` | Let's Encrypt |
| `ntfy.yourdomain.com` | `ntfy` | `80` | Let's Encrypt |

Enable "Force SSL" on both. Wait for Let's Encrypt certificates to issue.

---

## 7. Verify Miniflux

Open `https://miniflux.yourdomain.com` — log in with `admin` and the password from `.env`.

---

## 8. Create ntfy tokens

```bash
# Token for Miniflux to publish notifications
docker compose exec ntfy ntfy token add miniflux-publisher

# Token for your iOS device to subscribe
docker compose exec ntfy ntfy token add ios-reader
```

Save both tokens — you'll need them in the next steps.

---

## 9. Configure Miniflux → ntfy integration

In Miniflux: **Settings → Integrations → ntfy**

| Field | Value |
|---|---|
| ntfy URL | `https://ntfy.yourdomain.com` |
| Topic | `newsfeed` |
| Token | `miniflux-publisher` token from step 8 |

Enable the integration and save.

---

## 10. Add feeds

In Miniflux: **Feeds → Add Feed**

**Reddit (no credentials needed):**
```
https://www.reddit.com/r/SUBREDDIT/.rss
```

For full article text: open the feed settings → enable **"Fetch original content"**.

---

## 11. Set up Reeder 5 (iOS)

1. Install **Reeder 5** from the App Store
2. Open → **Accounts → +**  → **Miniflux**
3. Server URL: `https://miniflux.yourdomain.com`
4. API Key: generate one in Miniflux → **Settings → API Keys**

---

## 12. Set up ntfy iOS app

1. Install **ntfy** from the App Store
2. Add server: `https://ntfy.yourdomain.com`
3. Subscribe to topic: `newsfeed`
4. When prompted for a token, use the `ios-reader` token from step 8
5. Enable notifications for the topic

---

## Security checklist

- [ ] `MINIFLUX_DB_PASSWORD` — strong, unique (not shared with admin password)
- [ ] `MINIFLUX_ADMIN_PASSWORD` — strong, unique
- [ ] ntfy `auth-default-access: deny-all` is set in `ntfy/server.yml`
- [ ] NPM SSL certs issued for both domains
- [ ] No ports (`8080`, `5432`, `80`) exposed directly to the internet
- [ ] Reeder 5 uses API key, not the admin password
- [ ] `.env` is in `.gitignore` and never committed

---

## Updating

```bash
docker compose pull
docker compose up -d
```
