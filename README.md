# Personalized Newsfeed — Self-Hosted

Self-hosted RSS newsfeed stack with push notifications and a native iOS reader.

| Component | Role |
|---|---|
| **FreshRSS** | Feed aggregator, web UI, Google Reader API |
| **PostgreSQL** | FreshRSS database |
| **ntfy** | Push notifications to iOS |
| **Reeder Classic** (iOS) | Native feed reader via Google Reader API |
| **nginx Proxy Manager** | HTTPS termination (running separately) |

Feed list: see [feeds.md](feeds.md)

---

## Prerequisites

- Docker + Docker Compose installed on the server
- nginx Proxy Manager already running (with a Docker network you can attach to)
- Two DNS records pointing at your server:
  - `freshrss.yourdomain.com`
  - `ntfy.yourdomain.com`

---

## 1. Clone the repo

```bash
git clone git@github.com:Sprayer115/personalized-newsfeed.git newsfeed
cd newsfeed
```

---

## 2. Configure environment

```bash
cp .env.example .env
```

Edit `.env`:

```env
FRESHRSS_DB_PASSWORD=<strong unique password>
TZ=Europe/Berlin
```

---

## 3. Configure ntfy

Edit `ntfy/server.yml` and set your domain:

```yaml
base-url: https://ntfy.yourdomain.com
```

---

## 4. Connect to nginx Proxy Manager's Docker network

NPM needs to reach the `freshrss` and `ntfy` containers. Either:

**Option A — attach this stack to NPM's existing network:**

Find your NPM network name:
```bash
docker network ls
```

Add it as an external network to both the `freshrss` and `ntfy` services and declare it at the bottom of `docker-compose.yml`:

```yaml
networks:
  newsfeed:
    driver: bridge
  npm_network:        # replace with your actual NPM network name
    external: true
```

Then add `npm_network` to both service network lists.

**Option B — forward by host IP in NPM**, using the server's host IP and internal ports (`80` for both FreshRSS and ntfy).

---

## 5. Start the stack

```bash
docker compose up -d
docker compose ps
```

---

## 6. Set up nginx Proxy Manager proxy hosts

In the NPM web UI, create two **Proxy Hosts**:

| Domain | Forward Hostname | Forward Port | SSL |
|---|---|---|---|
| `freshrss.yourdomain.com` | `freshrss` | `80` | Let's Encrypt |
| `ntfy.yourdomain.com` | `ntfy` | `80` | Let's Encrypt |

Enable "Force SSL" on both.

---

## 7. FreshRSS initial setup

Open `https://freshrss.yourdomain.com` and complete the web installer:

- **Database:** PostgreSQL
  - Host: `freshrss-db`
  - Port: `5432`
  - Database: `freshrss`
  - User: `freshrss`
  - Password: value from `.env`
- Create your admin account
- Enable the **API** under Settings → Authentication → Allow API access (required for Reeder Classic)

---

## 8. Add feeds

Go to **Subscription Management → Add a feed** and paste URLs from [feeds.md](feeds.md).

---

## 9. Set up ntfy tokens

```bash
# Token for a notifier script/extension to publish
docker compose exec ntfy ntfy token add newsfeed-publisher

# Token for your iOS device to subscribe
docker compose exec ntfy ntfy token add ios-reader
```

> **Note:** FreshRSS has no native ntfy integration. Options:
> - Install the community extension [freshrss-notify](https://github.com/vert-fr/FreshRSS-Notify-Ext) via the extensions volume
> - Or use a lightweight cron/webhook sidecar that polls the FreshRSS API and sends to ntfy

---

## 10. Connect Reeder Classic (iOS)

1. Open Reeder Classic → **+** → **Self-Hosted → FreshRSS**
2. URL: `https://freshrss.yourdomain.com`
3. Username + password: your FreshRSS admin credentials

Reeder connects via the Google Reader compatible API at `/api/greader.php`.

---

## 11. Set up ntfy iOS app

1. Install **ntfy** from the App Store
2. Add server: `https://ntfy.yourdomain.com`
3. Subscribe to topic: `newsfeed`
4. Use the `ios-reader` token when prompted
5. Enable notifications for the topic

---

## Updating

```bash
docker compose pull
docker compose up -d
```

---

## Security checklist

- [ ] `FRESHRSS_DB_PASSWORD` — strong, unique
- [ ] FreshRSS admin password — strong, unique
- [ ] ntfy `auth-default-access: deny-all` set in `ntfy/server.yml`
- [ ] NPM SSL certs issued for both domains
- [ ] No ports exposed directly to the internet
- [ ] `.env` is never committed
