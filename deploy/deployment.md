# Heliolytics — Deployment Guide

One guide for **local development** (Docker on your machine) and **production** (any VPS + your domain).

| Component | Dev | Production |
|-----------|-----|------------|
| **Go API** | Docker · `localhost:8080` | `https://api.<your-domain>` |
| **Web dashboard** | Docker · `localhost:3000` | `https://dashboard.<your-domain>` |
| **Mobile app** | Debug build on phone (LAN HTTP OK) | Release build (HTTPS only) |

Three repos must exist as **siblings** in the same parent directory:

```
heliolytics/
  Heliolytics/        ← this repo (API + deploy/)
  Heliolytics_Web/    ← Next.js dashboard
  Heliolytics_App/    ← Flutter mobile app
```

---

## Secrets

| Variable | Used by | Purpose |
|----------|---------|---------|
| `HELIOLYTICS_SIGNING_SECRET` | API, Web, mobile app | HMAC auth — the app calls this **API key** in Settings |
| `HELIOLYTICS_WEB_PASSWORD` | Web login page | Dashboard password |
| `POSTGRES_PASSWORD` | PostgreSQL | Database password |

Generate strong values (run three times):

```bash
openssl rand -hex 32
```

Save them in a password manager. **Never commit `deploy/.env`.**

---

## Part 1 — Local development (Docker)

### 1.1 Prerequisites

- [Docker Desktop](https://docs.docker.com/desktop/) installed and running
- All three repos cloned side by side (see layout above)

### 1.2 Configure environment

```bash
cd path/to/Heliolytics/deploy
cp .env.example .env
```

Edit `deploy/.env`:

```env
HELIOLYTICS_SIGNING_SECRET=<openssl rand -hex 32>
HELIOLYTICS_WEB_PASSWORD=<your web login password>
POSTGRES_PASSWORD=<your db password>

API_PORT=8080
WEB_PORT=3000
HTTPS_PORT=443
REPARSE_ENABLED=false
```

### 1.3 Start the stack

```bash
chmod +x install.sh reset-db.sh
./install.sh
```

Wait ~1–2 minutes, then verify:

```bash
curl http://localhost:8080/health   # should print: ok
docker compose ps                  # all services should be "healthy"
```

| Service | URL |
|---------|-----|
| API | http://localhost:8080/health |
| Web dashboard | http://localhost:3000/login |
| HTTPS (Caddy, self-signed) | https://localhost (browser warning is expected in dev) |

### 1.4 Mobile app — dev

1. Connect phone and Mac to the **same Wi-Fi**.
2. Find your Mac's LAN IP: `ipconfig getifaddr en0`
3. Install a debug build on the phone (`flutter run` or `flutter build apk --debug`).
4. In the app: **Settings → Cloud API**
   - API base URL: `http://<LAN-IP>:8080`
   - API key: value of `HELIOLYTICS_SIGNING_SECRET` from `deploy/.env`
5. Tap **Test connection** — should succeed.

> Android emulator: use `http://10.0.2.2:8080` instead of the LAN IP.
> Debug builds allow `http://`; release builds require `https://`.

### 1.5 Useful dev commands

```bash
# View logs
docker compose logs -f api
docker compose logs -f web

# Stop everything
docker compose down

# Wipe DB and re-apply schema (deletes all data)
./reset-db.sh
```

---

## Part 2 — Production (VPS + custom domain)

### 2.1 What you need

1. A **VPS** — 1 vCPU, 2 GB RAM, 20 GB disk is enough to start.
   Providers: DigitalOcean, Hetzner, Linode, Oracle Cloud free tier, etc.
2. **Ubuntu 22.04 or 24.04** on the VPS.
3. **SSH access** as root or a sudo user.
4. A **domain name** with access to DNS settings.

Replace `YOUR_VPS_IP` and `<your-domain>` throughout the steps below with your real values.

---

### 2.2 DNS — point subdomains to the VPS

In your DNS panel, add two A records:

| Type | Name | Value | TTL |
|------|------|-------|-----|
| A | `api.heliolytics` | `YOUR_VPS_IP` | 300 |
| A | `heliolytics-dashboard` | `YOUR_VPS_IP` | 300 |

Wait 5–30 minutes, then verify from your local machine:

```bash
dig +short api.heliolytics.<your-domain>
dig +short heliolytics-dashboard.<your-domain>
# both should print YOUR_VPS_IP
```

Do not continue until DNS resolves correctly.

---

### 2.3 VPS — initial setup

```bash
ssh root@YOUR_VPS_IP

# Update system and install Docker
apt update && apt upgrade -y
apt install -y git curl ca-certificates ufw
curl -fsSL https://get.docker.com | sh
systemctl enable docker && systemctl start docker

# Firewall — allow SSH + web only
ufw allow OpenSSH
ufw allow 80/tcp
ufw allow 443/tcp
ufw enable
```

Do **not** expose ports 8080 or 3000 publicly — Caddy handles all inbound traffic on 80/443.

---

### 2.4 Clone repos on the VPS

```bash
mkdir -p /opt/heliolytics && cd /opt/heliolytics

git clone <your-Heliolytics-repo-url>     Heliolytics
git clone <your-Heliolytics_Web-repo-url> Heliolytics_Web
git clone <your-Heliolytics_App-repo-url> Heliolytics_App
```

---

### 2.5 Production environment file

```bash
cd /opt/heliolytics/Heliolytics/deploy
cp .env.example .env
nano .env
```

Use **new** secrets (different from dev):

```env
HELIOLYTICS_SIGNING_SECRET=<openssl rand -hex 32>
HELIOLYTICS_WEB_PASSWORD=<strong unique password>
POSTGRES_PASSWORD=<strong unique password>

API_PORT=8080
WEB_PORT=3000
HTTPS_PORT=443
REPARSE_ENABLED=false
```

Lock file permissions:

```bash
chmod 600 .env
```

---

### 2.6 Production Caddyfile

Replace the dev `Caddyfile` with your real domain names:

```caddy
api.heliolytics.<your-domain> {
    reverse_proxy api:8080
}

heliolytics-dashboard.<your-domain> {
    reverse_proxy web:3000
}
```

Caddy automatically obtains TLS certificates from Let's Encrypt and redirects HTTP → HTTPS.

**Requirements:** DNS must resolve to this server; ports 80 and 443 must be open.

---

### 2.7 Expose ports 80 and 443 in docker-compose

Edit the `caddy:` service in `docker-compose.yml`:

```yaml
    ports:
      - "80:80"
      - "443:443"
```

Optionally bind the API and web services to localhost only (prevents direct internet access):

```yaml
  api:
    ports:
      - "127.0.0.1:8080:8080"

  web:
    ports:
      - "127.0.0.1:3000:3000"
```

---

### 2.8 Start production stack

```bash
cd /opt/heliolytics/Heliolytics/deploy
chmod +x install.sh
./install.sh
```

Verify:

```bash
docker compose ps
curl -s https://api.heliolytics.<your-domain>/health    # ok
curl -I https://heliolytics-dashboard.<your-domain>/login  # HTTP/2 200
```

If Caddy logs show certificate errors:
- DNS not propagated yet — wait, then `docker compose restart caddy`
- Port 80 blocked — check `ufw` and your VPS provider's firewall/security group settings

---

### 2.9 Mobile app — production

Build a **release** APK on your development machine:

```bash
cd path/to/Heliolytics_App
flutter pub get
flutter build apk --release
# APK: build/app/outputs/flutter-apk/app-release.apk
```

Transfer to your phone and install. In the app: **Settings → Cloud API**

| Field | Value |
|-------|-------|
| API base URL | `https://api.heliolytics.<your-domain>` |
| API key | value of `HELIOLYTICS_SIGNING_SECRET` from VPS `deploy/.env` |

Tap **Test connection** — must succeed over HTTPS.

> Release builds reject `http://` URLs. Always use the `https://` URL.

---

### 2.10 Production checklist

- [ ] DNS A records for both subdomains → VPS IP
- [ ] `dig` returns correct IP for both hostnames
- [ ] UFW: only ports 22, 80, 443 open
- [ ] `deploy/.env` uses unique secrets (not dev defaults), permissions `600`
- [ ] `Caddyfile` uses your production hostnames
- [ ] `docker compose ps` — all services healthy
- [ ] `curl https://api.heliolytics.<your-domain>/health` → `ok`
- [ ] Web login works at the dashboard URL
- [ ] Mobile app: HTTPS URL + API key tested successfully

---

### 2.11 Updating after code changes

```bash
cd /opt/heliolytics/Heliolytics && git pull
cd /opt/heliolytics/Heliolytics_Web && git pull
cd /opt/heliolytics/Heliolytics/deploy
docker compose build
docker compose up -d
```

If `schema.sql` changed:

```bash
./reset-db.sh   # WARNING: deletes all stored data
```

---

### 2.12 Troubleshooting

| Problem | What to check |
|---------|---------------|
| `dig` does not return VPS IP | DNS propagation — wait and retry |
| Caddy certificate failed | Port 80 open; DNS correct; `docker compose logs caddy` |
| Web login fails | `HELIOLYTICS_WEB_PASSWORD` in `.env`; `docker compose restart web` |
| App "Test connection" fails | HTTPS URL correct (no trailing slash); API key matches `.env` |
| App works on Wi-Fi but not mobile data | API must be on public HTTPS, not a LAN IP |
| `401` from API | Wrong API key in app settings |
| Empty dashboard | Sync from the app first; check `docker compose logs api` during upload |
| API unhealthy | `docker compose logs db api`; DB password in `.env` matches `DATABASE_URL` |

View logs:

```bash
docker compose logs -f api
docker compose logs -f web
docker compose logs -f caddy
docker compose logs -f db
```

---

## Architecture

```
Mobile app ──HTTPS──► api.heliolytics.<your-domain>
                              │
                         Caddy :443
                              │
                         Go API :8080
                              │
                         PostgreSQL

Browser ──HTTPS──► heliolytics-dashboard.<your-domain>
                              │
                         Caddy :443
                              │
                         Next.js :3000 ──internal──► Go API :8080
```

Both subdomains point to the same VPS. Caddy routes by hostname to either the API container or the web container.
