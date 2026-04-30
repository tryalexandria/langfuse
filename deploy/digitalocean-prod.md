# Self-hosted Langfuse on DigitalOcean

Production deployment notes for `langfuse.tryalexandria.fr`. One Docker droplet,
host-level Caddy reverse proxy, Cloudflare-only ingress.

## At a glance

| | |
|---|---|
| URL | https://langfuse.tryalexandria.fr |
| Droplet | `langfuse-prod` (id `568181235`), `s-4vcpu-8gb`, fra1, Docker on Ubuntu 22.04 |
| Public IPv4 | `165.22.80.137` |
| Cloud Firewall | `langfuse-prod-fw` (id `753f6e86-edbb-4f00-8bf7-b94f63c5333e`) |
| Cost | ≈ $48 droplet + ≈ $10 backups ≈ **$58/mo** |
| Backups | DO weekly snapshots: enabled |
| Monitoring | DO agent: enabled |

## Architecture

```
                 ┌─────────────────┐
                 │     Browser     │
                 └────────┬────────┘
                  HTTPS (Cloudflare cert)
                          │
                 ┌────────▼────────┐
                 │   Cloudflare    │  proxied A record, SSL/TLS = Flexible or Full
                 │  (orange-cloud) │  (Caddy serves both schemes — either works)
                 └────────┬────────┘
                          │  HTTP:80  or  HTTPS:443
              ────────────┼──────────────────────────────────
              DO Cloud Firewall: only Cloudflare CIDRs on 80/443
                          │       SSH:22 from anywhere
                          │       (everything else blocked)
              ────────────┼──────────────────────────────────
                          │
            ┌─────────────▼────────────────┐
            │  Droplet  165.22.80.137       │
            │                                │
            │  ┌──────────────────────────┐  │
            │  │ Caddy (host-level)       │  │   /etc/caddy/Caddyfile
            │  │  :80  -> reverse_proxy   │  │   internal CA cert (auto-renewed)
            │  │  :443 -> reverse_proxy   │  │   logs: /var/log/caddy/access.log
            │  └────────────┬─────────────┘  │
            │               │ http://localhost:3000
            │  ┌────────────▼─────────────┐  │
            │  │ docker compose stack     │  │   /opt/langfuse/
            │  │   langfuse-web           │  │   docker-compose.yml (upstream, do not edit)
            │  │   langfuse-worker        │  │   docker-compose.override.yml (local edits)
            │  │   postgres               │  │
            │  │   clickhouse             │  │
            │  │   redis                  │  │   env: /etc/langfuse/.env (mode 600)
            │  │   minio  (internal only) │  │
            │  └──────────────────────────┘  │
            └────────────────────────────────┘
```

Why Caddy in front: Cloudflare can only reach a free-plan origin on a small
fixed set of ports (default 80/443). Langfuse listens on 3000. Caddy bridges
the gap *and* terminates TLS so origin traffic is encrypted end-to-end.

## Files & their owners

| Path | Owner | Edit how |
|---|---|---|
| `/etc/langfuse/.env` (mode 600) | secrets + `NEXTAUTH_URL`, `AUTH_DISABLE_SIGNUP`, etc. | DO Console → `sudo nano` |
| `/opt/langfuse/docker-compose.yml` | **upstream** Langfuse compose file | do not hand-edit; refetch from langfuse/langfuse on upgrade |
| `/opt/langfuse/docker-compose.override.yml` | local env passthroughs (e.g. `AUTH_DISABLE_SIGNUP`) | edit on droplet; survives upstream upgrades |
| `/opt/langfuse/.env` | symlink → `/etc/langfuse/.env` | n/a |
| `/etc/caddy/Caddyfile` | reverse-proxy config | edit on droplet, then `systemctl reload caddy` |
| `/var/log/caddy/access.log` | Caddy request log | tail for debugging |
| `/var/log/langfuse-bootstrap.log` | first-boot script log | append-only, useful if recreating |
| DO Cloud Firewall `langfuse-prod-fw` | edge ACL | `doctl compute firewall …` from local |

## Cloudflare requirements

- DNS: `A langfuse.tryalexandria.fr → 165.22.80.137`, **proxied (orange cloud)**.
- SSL/TLS mode: **Flexible** *or* **Full**. `Full (strict)` will reject Caddy's
  self-signed cert — to use strict, swap in a Cloudflare Origin Cert (see
  *Future / optional* below).
- No Origin Rules / Page Rules / Workers required.

## Common operations

### SSH access
```bash
ssh root@165.22.80.137
# both `FP @ macBook` and `FP @ macStudio` SSH keys are authorized.
```

### Edit secrets / config
From DO dashboard → **Droplets → langfuse-prod → Console**:
```bash
sudo nano /etc/langfuse/.env
cd /opt/langfuse && docker compose up -d
```
The only value you should be editing day-to-day is `NEXTAUTH_URL` (after a
domain change) or feature flags like `AUTH_DISABLE_SIGNUP`. **Do not change**
`NEXTAUTH_SECRET`, `SALT`, `ENCRYPTION_KEY`, or any DB/Redis/MinIO password
after first boot — existing data becomes unreadable.

### Apply changes after editing `.env` or `docker-compose.override.yml`
```bash
cd /opt/langfuse && docker compose up -d
```
Compose only recreates containers whose env or config changed. Healthy DB
volumes are untouched.

### Upgrade Langfuse
```bash
cd /opt/langfuse
docker compose pull          # pulls new image tags (langfuse:3, langfuse-worker:3)
docker compose up -d
```
ClickHouse/Postgres migrations run automatically on `langfuse-web` startup.
Rolling-window upgrade is *not* supported (single-node deployment), expect
~30 s downtime.

### Reload Caddy after editing the Caddyfile
```bash
caddy validate --config /etc/caddy/Caddyfile  # syntax-check first
systemctl reload caddy
```

### View logs
```bash
docker compose -f /opt/langfuse/docker-compose.yml logs -f langfuse-web
docker compose -f /opt/langfuse/docker-compose.yml logs -f langfuse-worker
tail -F /var/log/caddy/access.log     # all requests through Caddy
journalctl -u caddy --since "10 min ago"
```

### Disk usage
ClickHouse + Postgres + MinIO grow over time; check periodically:
```bash
docker system df -v
df -h /var/lib/docker
```

### Resize the droplet
DO dashboard → Droplets → `langfuse-prod` → **Resize**. CPU/RAM resize is a
power-cycle (~1 min downtime). Disk-only resize is online. The account
currently exposes only Basic-tier sizes (max `s-4vcpu-8gb`) — open a DO
support ticket to unlock General-Purpose / Memory-Optimized tiers if needed.

### Rotate a non-data secret (e.g. SMTP password, an SSO client secret)
1. Edit `/etc/langfuse/.env` via DO Console.
2. `cd /opt/langfuse && docker compose up -d langfuse-web langfuse-worker`.
**Never rotate** `NEXTAUTH_SECRET`/`SALT`/`ENCRYPTION_KEY` or DB passwords —
those would brick the existing data.

## Maintenance gotchas

- **UFW is disabled** on the droplet. The Docker 1-Click image ships with UFW
  enabled and a permissive rule for ports 2375/2376 (Docker remote API!) but a
  deny on 80/443. We rely on the DO Cloud Firewall as the single source of
  truth. Don't re-enable UFW without also opening 80 + 443.
- **Caddy uses an internal-CA self-signed cert.** Cloudflare SSL/TLS must
  therefore be `Flexible` or `Full` — not `Full (strict)`. The cert auto-rotates;
  no maintenance needed.
- **Port 9090 (MinIO) is firewalled off** to keep the attack surface small.
  Side effect: Langfuse multimodal direct uploads aren't reachable from
  browsers. To enable, add an inbound 9090 rule to the firewall and update
  `LANGFUSE_S3_MEDIA_UPLOAD_ENDPOINT` to a public hostname.
- **DigitalOcean apt repo signature warning** appears in cloud-init logs
  (`NO_PUBKEY 35696F43FC7DB4C2`). Cosmetic; can be ignored or fixed by
  reimporting the DO apt key.
- **First-boot bootstrap is idempotent** — re-running it never regenerates
  secrets or recreates volumes (guarded by `[ ! -f /etc/langfuse/.env ]`).

## Backups

DO snapshots run weekly. To restore:
1. Dashboard → Backups → choose snapshot → **Restore Droplet**.
2. The restored droplet boots with the same `/etc/langfuse/.env`, volumes,
   and Caddy config. DNS A record will need to be repointed if the IP changed.

For an extra layer (offsite Postgres + ClickHouse dumps), see *Future / optional*.

## Bootstrap script (reference)

The droplet was provisioned via DO cloud-init `--user-data-file` with the
script below. It is idempotent and can be re-run on a fresh droplet to
recreate this exact setup. Stored on the droplet at
`/root/langfuse-bootstrap.sh`.

```bash
#!/bin/bash
set -uo pipefail
LOG=/var/log/langfuse-bootstrap.log
{ echo; echo "=== Langfuse bootstrap: $(date -u) ==="; } >> "$LOG"
exec >>"$LOG" 2>&1

# 1. Wait for docker
for i in $(seq 1 60); do
  docker info >/dev/null 2>&1 && break
  sleep 2
done

# 2. Layout
mkdir -p /etc/langfuse /opt/langfuse
chmod 750 /etc/langfuse

# 3. Fetch upstream compose file
[ -f /opt/langfuse/docker-compose.yml ] || curl -fsSL -o /opt/langfuse/docker-compose.yml \
  https://raw.githubusercontent.com/langfuse/langfuse/main/docker-compose.yml

# 4. Generate secrets once (hex => no shell-quoting / URL-encoding hazards)
if [ ! -f /etc/langfuse/.env ]; then
  ENC_KEY=$(openssl rand -hex 32)
  NEXTAUTH_SECRET=$(openssl rand -hex 32)
  SALT=$(openssl rand -hex 32)
  PG_PW=$(openssl rand -hex 24); CH_PW=$(openssl rand -hex 24)
  REDIS_PW=$(openssl rand -hex 24); MINIO_PW=$(openssl rand -hex 24)
  IP=$(curl -fsSL http://169.254.169.254/metadata/v1/interfaces/public/0/ipv4/address || echo 127.0.0.1)
  umask 077
  cat > /etc/langfuse/.env <<EOF
NEXTAUTH_URL=http://${IP}:3000
NEXTAUTH_SECRET=${NEXTAUTH_SECRET}
SALT=${SALT}
ENCRYPTION_KEY=${ENC_KEY}
POSTGRES_USER=postgres
POSTGRES_PASSWORD=${PG_PW}
POSTGRES_DB=postgres
DATABASE_URL=postgresql://postgres:${PG_PW}@postgres:5432/postgres
CLICKHOUSE_USER=clickhouse
CLICKHOUSE_PASSWORD=${CH_PW}
REDIS_AUTH=${REDIS_PW}
MINIO_ROOT_USER=minio
MINIO_ROOT_PASSWORD=${MINIO_PW}
LANGFUSE_S3_EVENT_UPLOAD_ACCESS_KEY_ID=minio
LANGFUSE_S3_EVENT_UPLOAD_SECRET_ACCESS_KEY=${MINIO_PW}
LANGFUSE_S3_MEDIA_UPLOAD_ACCESS_KEY_ID=minio
LANGFUSE_S3_MEDIA_UPLOAD_SECRET_ACCESS_KEY=${MINIO_PW}
LANGFUSE_S3_BATCH_EXPORT_ACCESS_KEY_ID=minio
LANGFUSE_S3_BATCH_EXPORT_SECRET_ACCESS_KEY=${MINIO_PW}
TELEMETRY_ENABLED=true
EOF
  chmod 600 /etc/langfuse/.env
fi

# 5. Compose finds .env next to its yaml
ln -sf /etc/langfuse/.env /opt/langfuse/.env

# 6. 4 GB swap insurance for ClickHouse + Postgres on 8 GB host
if ! swapon --show | grep -q .; then
  fallocate -l 4G /swapfile && chmod 600 /swapfile && mkswap /swapfile && swapon /swapfile
  echo '/swapfile none swap sw 0 0' >> /etc/fstab
fi

# 7. Cap docker container log size
mkdir -p /etc/docker
[ -f /etc/docker/daemon.json ] || cat > /etc/docker/daemon.json <<'EOF'
{ "log-driver": "json-file", "log-opts": { "max-size": "50m", "max-file": "3" } }
EOF

# 8. Bring up the stack
cd /opt/langfuse && docker compose pull && docker compose up -d
```

Post-bootstrap manual steps (done once at provisioning):

1. `ufw --force disable` (DO Cloud Firewall is the source of truth).
2. Install Caddy and write `/etc/caddy/Caddyfile`:
   ```
   (langfuse_proxy) {
       encode zstd gzip
       log { output file /var/log/caddy/access.log { roll_size 50mb roll_keep 5 } format console }
       reverse_proxy 127.0.0.1:3000 {
           header_up Host {host}
           header_up X-Forwarded-Proto https
       }
   }
   http://langfuse.tryalexandria.fr { import langfuse_proxy }
   langfuse.tryalexandria.fr        { tls internal; import langfuse_proxy }
   ```
3. Create the DO Cloud Firewall (rules: SSH:22 anywhere, ICMP anywhere,
   80 + 443 from Cloudflare CIDRs only).
4. Cloudflare: proxied A record + SSL/TLS = Flexible or Full.
5. Update `NEXTAUTH_URL` in `.env` to `https://langfuse.tryalexandria.fr` and
   `docker compose up -d`.

## Auth configuration

`AUTH_DISABLE_SIGNUP=true` is set in `/etc/langfuse/.env` and passed through
via `docker-compose.override.yml`. Existing users keep working; new signups
return `HTTP 422 {"message":"Sign up is disabled."}`. To re-enable, flip to
`false` and `docker compose up -d langfuse-web`.

To add another user with signup disabled: log in as an org owner →
**Settings → Members → Invite member by email**.

## Troubleshooting

| Symptom | First check |
|---|---|
| Cloudflare 522 *Connection timed out* | DO Cloud Firewall has `80` and `443` from Cloudflare CIDRs; UFW is disabled (`ufw status`). Tcpdump on the droplet: `tcpdump -nn -i any 'tcp port 80 or tcp port 443'` should show CF egress IPs hitting it. |
| Cloudflare 525 *SSL handshake failed* | SSL/TLS mode is `Full (strict)` but Caddy is serving its internal cert. Drop to `Full`, or upgrade to a Cloudflare Origin Cert. |
| ERR_TOO_MANY_REDIRECTS | NextAuth thinks it's HTTP. Check `NEXTAUTH_URL=https://...` in `.env` and that Caddy is sending `X-Forwarded-Proto: https`. |
| Login works but cookies don't persist | Same root cause as above — secure-cookie / scheme mismatch. |
| `langfuse-web` won't become healthy after upgrade | Check `docker logs langfuse-langfuse-web-1` for ClickHouse migration errors. Migrations are forward-only. |
| ClickHouse OOM-kills | RAM-tight on 8 GB host. Resize droplet or trim retention via the Langfuse UI. The 4 GB swap absorbs short spikes. |
| Disk usage climbing | `docker system df -v`. Mostly ClickHouse + MinIO. Resize the droplet's disk online from DO dashboard. |
| Caddy access log empty | File ownership; should be `caddy:caddy`. After a fresh restart, recreate with `install -o caddy -g caddy -m 0644 /dev/null /var/log/caddy/access.log`. |

## Future / optional improvements

- **Cloudflare Origin Certificate + `Full (strict)`** — paste the cert+key into
  `/etc/caddy/origin.{crt,key}`, change `tls internal` to `tls /etc/caddy/origin.crt /etc/caddy/origin.key`,
  reload Caddy, switch SSL/TLS to `Full (strict)`. 5-min change, 15-year cert,
  no auto-renew.
- **Offsite DB backups** — `docker exec langfuse-postgres-1 pg_dumpall -U postgres`
  + `clickhouse-backup` on a daily cron, push to a DO Space (~$5/mo).
- **SMTP for invite emails** — set `EMAIL_FROM_ADDRESS` and `SMTP_CONNECTION_URL`
  in `/etc/langfuse/.env`. They're already declared in the upstream compose
  env block; no override-file change needed.
- **Resize to 16 GB RAM** — once the DO account exposes the larger tiers; useful
  if ClickHouse cardinality grows.
- **Multimodal direct uploads** — open port 9090 in the firewall, set
  `LANGFUSE_S3_MEDIA_UPLOAD_ENDPOINT` to a publicly-resolvable URL (would need
  a separate `media.langfuse.tryalexandria.fr` Cloudflare record + Caddy site).
