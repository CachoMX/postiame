# Postia — Hostinger deployment

Production deployment artifacts for `postia.me` running on the Hostinger VPS at `31.220.17.97` (`srv855026`).

## Architecture

```
┌─────────────────┐
│  Cloudflare DNS │  postia.me  → 31.220.17.97  (DNS only, gray cloud)
│   (DNS only)    │  postiame.com → 31.220.17.97 (DNS only, gray cloud)
└────────┬────────┘
         │
         ▼
┌─────────────────────── Hostinger VPS (Ubuntu 24.04) ───────────────────────┐
│                                                                            │
│   /root/docker-compose.yml          ◄── Traefik (:80, :443) + n8n          │
│       └─ Traefik with Let's Encrypt (TLS-ALPN-01 challenge)                │
│       └─ Watches Docker network: root_default                              │
│                                                                            │
│   /docker/postia/docker-compose.yml ◄── THIS STACK                         │
│       ├─ postiz   (ghcr.io/gitroomhq/postiz-app:v2.25.1, port 5000)        │
│       ├─ postgres (postgres:17-alpine)                                     │
│       ├─ redis    (redis:7-alpine)                                         │
│       ├─ temporal (temporalio/auto-setup:1.28.1)                           │
│       └─ elasticsearch (7.17.27)                                           │
│                                                                            │
│   /docker/postiz-fbb0/docker-compose.yml ◄── existing postiz.vixi.agency   │
│       (decommissioned after postia.me cutover verified)                    │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

Routing (handled by Traefik labels in `docker-compose.yml`):

| Hostname           | Action                              |
|--------------------|-------------------------------------|
| `postia.me`        | Serve Postiz app (TLS via LE)       |
| `www.postia.me`    | 301 → `https://postia.me`           |
| `postiame.com`     | 301 → `https://postia.me`           |
| `www.postiame.com` | 301 → `https://postia.me`           |

## First-time deployment

Run on the VPS as root.

### 1. Clone the fork into a sibling location

```bash
cd /docker
git clone --branch main --depth 50 https://github.com/CachoMX/postiame.git postia-repo
```

### 2. Stage the postia stack directory

```bash
mkdir -p /docker/postia
cd /docker/postia
ln -sf /docker/postia-repo/deploy/hostinger/docker-compose.yml docker-compose.yml
cp /docker/postia-repo/deploy/hostinger/.env.example .env
```

### 3. Generate secrets and write them into `.env`

```bash
JWT=$(openssl rand -base64 48 | tr -d '=+/\n' | head -c 48)
DB=$(openssl rand -base64 32 | tr -d '=+/\n' | head -c 32)
sed -i "s|^JWT_SECRET=.*|JWT_SECRET=${JWT}|" .env
sed -i "s|^DB_PASSWORD=.*|DB_PASSWORD=${DB}|" .env
echo "JWT_SECRET and DB_PASSWORD written to /docker/postia/.env — also save in 1Password."
grep -E '^(JWT_SECRET|DB_PASSWORD)=' .env
```

Copy both values into 1Password (Vixi vault → "Postia production secrets").

### 4. Verify DNS resolves to this VPS

```bash
dig +short postia.me
dig +short www.postia.me
# Both should return 31.220.17.97. If not, wait for Cloudflare propagation
# (typically 5–60 min after the NS switch in Squarespace).
```

### 5. Bring the stack up

```bash
cd /docker/postia
docker compose up -d
docker compose ps
docker compose logs -f postiz   # Ctrl-C to exit log tail
```

Traefik will request a Let's Encrypt cert for `postia.me` (and for the redirect hostnames) on the first inbound HTTPS request.

### 6. Verify

```bash
curl -sI https://postia.me                  # expect HTTP/2 200 or 307
curl -sI https://www.postia.me              # expect HTTP/2 308 -> location: https://postia.me/
curl -sI https://postiame.com               # expect HTTP/2 308 -> location: https://postia.me/
curl -sI https://www.postiame.com           # expect HTTP/2 308 -> location: https://postia.me/
```

Open https://postia.me in a browser — Postiz login page should load.

## Updating to a new Postiz version

```bash
cd /docker/postia-repo
git fetch upstream
git checkout <new-tag-or-sha>
# (compose file points at a pinned image — bump the v2.x.y in docker-compose.yml)
cd /docker/postia
docker compose pull
docker compose up -d
```

## Troubleshooting

- **Cert not issued:** check Traefik logs (`docker logs root-traefik-1`). Most common cause: Cloudflare proxy turned ON for `postia.me` (must be DNS-only / gray cloud).
- **Postiz crashes on boot:** `docker compose logs postiz` — usually missing `JWT_SECRET` or DB connection issue.
- **Out-of-memory while old + new stacks both running:** stop the old `postiz` stack (`docker compose -f /docker/postiz-fbb0/docker-compose.yml stop`) — it's getting decommissioned anyway.

## Related

- Deployment baseline tag: `vixi-deploy-2026-05-04` → upstream `v2.21.7`
- Reverse proxy / Traefik config: `/root/docker-compose.yml` on the VPS (do not modify without coordination — also serves `n8n`)
- Migration plan: `SOW-Postia-Migration.md` (1Password, Vixi vault — not committed)
