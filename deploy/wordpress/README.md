# WordPress (blog.postia.me) — Hostinger deployment

Test WordPress install on the same VPS as Postiz, used to validate the Postiz → WordPress integration end-to-end. Free (no extra hosting cost), runs alongside the other tenants.

## DNS

Cloudflare A record:

| Type | Name | Content | Proxy |
|---|---|---|---|
| A | `blog` | `31.220.17.97` | DNS only (gray cloud) |

DNS-only is required because Traefik on the VPS issues the TLS cert via Let's Encrypt TLS-ALPN-01 — Cloudflare proxy mode would intercept the handshake.

## First-time deployment

Run on the VPS as root.

### 1. Pull the repo (already cloned at `/docker/postia-repo` from the postia setup)

```bash
cd /docker/postia-repo && git pull
```

### 2. Stage the wordpress stack

```bash
mkdir -p /docker/wordpress
cd /docker/wordpress
ln -sf /docker/postia-repo/deploy/wordpress/docker-compose.yml docker-compose.yml
cp /docker/postia-repo/deploy/wordpress/.env.example .env
```

### 3. Generate DB secrets

```bash
ROOT=$(openssl rand -base64 32 | tr -d '=+/\n' | head -c 32)
DB=$(openssl rand -base64 32 | tr -d '=+/\n' | head -c 32)
sed -i "s|^DB_ROOT_PASSWORD=.*|DB_ROOT_PASSWORD=${ROOT}|" .env
sed -i "s|^DB_PASSWORD=.*|DB_PASSWORD=${DB}|" .env
echo "DB credentials written. Save in 1Password as well."
grep -E '^(DB_ROOT_PASSWORD|DB_PASSWORD)=' .env | awk -F= '{print $1, "=", "<set, "length($2)" chars>"}'
```

### 4. Bring up

```bash
docker compose up -d
docker compose ps
```

Wait ~30 sec for MariaDB health, then WordPress comes up.

### 5. Run the WordPress install wizard

Open `https://blog.postia.me` in a browser. WordPress's setup wizard appears:
- Pick language → continue
- **Site title:** anything (e.g. "Postia Test")
- **Username:** an admin name (don't reuse `admin`, pick something like `vixiwp`)
- **Password:** generate a strong one in 1Password — DO NOT use a common one. Save in 1Password.
- **Email:** `admin@vixi.agency`
- **Search engine visibility:** check the "Discourage" box (it's a test site, no need for it to be indexed)
- Click **Install WordPress**

### 6. Generate Application Password (for Postiz to use)

In wp-admin:
- Users → Profile (or "Edit Profile") → scroll all the way down to **Application Passwords**
- Name: `Postia`
- Click **Add New Application Password**
- Copy the 24-char value (e.g. `XXXX XXXX XXXX XXXX XXXX XXXX`) — WP only shows this once
- Save in 1Password

### 7. Connect from Postiz

In postia.me → Connect WordPress → fill the form:

| Field | Value |
|---|---|
| Domain URL | `https://blog.postia.me` |
| Username | the WP admin user from step 5 |
| Password | the Application Password from step 6 (with spaces) |

Submit. Channel should land.

## Updating

```bash
cd /docker/postia-repo && git pull   # if compose changed
cd /docker/wordpress
docker compose pull                  # latest WordPress + MariaDB images
docker compose up -d
```

WordPress core also self-updates from inside wp-admin → Updates. The Docker image bump is for security patches to PHP / Apache / underlying OS.

## Troubleshooting

- **Cert not issued for blog.postia.me**: check Cloudflare proxy is OFF (gray cloud). Check Traefik logs: `docker logs root-traefik-1 | grep -i blog.postia.me`.
- **WP install wizard says "database connection error"**: MariaDB hadn't finished initializing when WP started. `docker compose restart wordpress`. If that doesn't fix it, check `.env` matches the env vars in compose.
- **Postiz channel won't connect**: verify the Application Password is correct (re-paste with spaces). Ensure the user has `publish_posts` capability — Administrator role does by default.

## Resource cost

Roughly ~500 MB RAM and ~1 GB disk while idle. WordPress + MariaDB images add ~700 MB combined to disk.
