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
CLOUDFLARE_TUNNEL_TOKEN=
REPARSE_ENABLED=false
```

Leave `CLOUDFLARE_TUNNEL_TOKEN` empty for local dev (skip tunnel; use localhost ports).

### 1.3 Start the stack

```bash
chmod +x install.sh reset-db.sh
./install.sh
```

Wait ~1–2 minutes, then verify:

```bash
curl http://127.0.0.1:8080/health   # should print: ok
docker compose ps                     # api, web, db healthy; cloudflared may restart if no token
```

| Service | URL |
|---------|-----|
| API | http://127.0.0.1:8080/health |
| Web dashboard | http://127.0.0.1:3000/login |

> API and web bind to **127.0.0.1** only — not exposed on the public internet. Production HTTPS uses Cloudflare Tunnel (Part 2).

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

# Redeploy ONLY the web dashboard (db/api stay up, no API downtime)
./deploy-web.sh            # build from the current Heliolytics_Web checkout
PULL=1 ./deploy-web.sh     # git pull the web repo first, then rebuild
```

---

## Part 2 — Production (VPS + Cloudflare Tunnel)

HTTPS terminates at **Cloudflare** (orange proxy). The VPS runs `cloudflared` — **no Caddy**, **no open ports 80/443** on Oracle.

Example domain: **anshuman115.in** on Cloudflare DNS.

---

### 2.1 What you need

1. **VPS** — 1 vCPU, 2 GB RAM (Oracle Cloud free tier works).
2. **Ubuntu 22.04 or 24.04** + SSH.
3. Domain on **Cloudflare** (nameservers pointed to Cloudflare).
4. **Cloudflare Zero Trust** (free tier) for tunnels.

---

### 2.2 Create the tunnel (Cloudflare dashboard)

1. [Cloudflare Zero Trust](https://one.dash.cloudflare.com/) → **Networks** → **Tunnels** → **Create a tunnel**.
2. Connector type: **Cloudflared**.
3. Name: `heliolytics` → Save.
4. **Public Hostnames** — add one row per service:

| Public hostname | Service type | URL (Docker network) |
|-----------------|--------------|----------------------|
| `api.heliolytics.anshuman115.in` | HTTP | `http://api:8080` |
| `dashboard.heliolytics.anshuman115.in` | HTTP | `http://web:3000` |

Cloudflare creates DNS records (CNAME to tunnel) automatically — **no manual A records**, no grey/orange choice needed (proxied by default).

5. Copy the **tunnel token** (long string starting with `eyJ...`).

**Many services?** Add more public hostname rows — each maps a subdomain to `http://<service>:<port>` on the Docker network. Ten services = ten rows. Non-Docker apps on the VM: use `http://host.docker.internal:PORT` (Linux: add `extra_hosts: ["host.docker.internal:host-gateway"]` on the `cloudflared` service).

---

### 2.3 VPS setup

```bash
ssh ubuntu@YOUR_VPS_IP

sudo apt update && sudo apt upgrade -y
sudo apt install -y git curl ca-certificates ufw
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
# log out and back in

sudo ufw allow OpenSSH
sudo ufw enable
```

**Do not open 80, 443, 8080, or 3000** on Oracle security list or UFW. Only **22** (SSH). Tunnel is outbound from VM → Cloudflare.

---

### 2.4 Clone repos (home directory example)

```bash
mkdir -p ~/heliolytics && cd ~/heliolytics

git clone -b v2 <Heliolytics-repo-url>     Heliolytics
git clone -b v2 <Heliolytics_Web-repo-url> Heliolytics_Web
```

---

### 2.5 Environment file

```bash
cd ~/heliolytics/Heliolytics/deploy
cp .env.example .env
nano .env
chmod 600 .env
```

```env
HELIOLYTICS_SIGNING_SECRET=<openssl rand -hex 32>
HELIOLYTICS_WEB_PASSWORD=<strong password>
POSTGRES_PASSWORD=<openssl rand -hex 32>
CLOUDFLARE_TUNNEL_TOKEN=<paste token from step 2.2>
REPARSE_ENABLED=false
```

---

### 2.6 Start stack

```bash
chmod +x install.sh
./install.sh
```

Verify:

```bash
docker compose ps                    # api, web, db, cloudflared all up
docker compose logs cloudflared      # "Registered tunnel connection"
curl -s https://api.heliolytics.anshuman115.in/health    # ok
```

Web: `https://dashboard.heliolytics.anshuman115.in/login`

---

### 2.7 Mobile app — production

On your Mac:

```bash
cd Heliolytics_App && git checkout v2
flutter build apk --release \
  --dart-define=API_URL=https://api.heliolytics.anshuman115.in \
  --dart-define=API_SIGNING_SECRET=<HELIOLYTICS_SIGNING_SECRET>
```

Install APK → **Settings → Cloud API** → Test connection → sync strap.

---

### 2.8 Production checklist

- [ ] Tunnel public hostnames match your real domain
- [ ] `CLOUDFLARE_TUNNEL_TOKEN` in `.env`, permissions `600`
- [ ] Oracle security list: **SSH only** (no inbound 80/443)
- [ ] `docker compose ps` — all healthy, cloudflared connected
- [ ] `curl https://api.heliolytics.anshuman115.in/health` → `ok`
- [ ] Dashboard login works
- [ ] Release APK uses **https://** API URL

---

### 2.9 Updating after code changes

```bash
cd ~/heliolytics/Heliolytics && git pull origin v2
cd ~/heliolytics/Heliolytics_Web && git pull origin v2
cd ~/heliolytics/Heliolytics/deploy
docker compose build && docker compose up -d
```

---

### 2.10 Troubleshooting

| Problem | Fix |
|---------|-----|
| `cloudflared` restart loop | Token missing/wrong in `.env` |
| 502 from Cloudflare | `docker compose ps` — api/web unhealthy; check logs |
| Wrong service on hostname | Fix public hostname URL in Zero Trust (must match Docker service name) |
| App fails off Wi-Fi | API URL must be public `https://` hostname, not LAN IP |
| `401` from API | API key in app ≠ `HELIOLYTICS_SIGNING_SECRET` |

```bash
docker compose logs -f cloudflared
docker compose logs -f api
docker compose logs -f web
```

---

## Architecture

```
Mobile app ──HTTPS──► Cloudflare edge
                              │
                         cloudflared (Docker)
                         ╱         ╲
                   api:8080      web:3000
                         ╲         ╱
                         PostgreSQL

Browser ──HTTPS──► Cloudflare edge ──► cloudflared ──► web:3000 ──► api:8080 (internal)
```

No inbound web ports on the VPS. Each public hostname in Zero Trust maps to a different internal `service:port`.
