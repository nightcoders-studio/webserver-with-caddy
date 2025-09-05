Using This Webserver (Caddy + Docker)

Overview
- Purpose: Provide a simple, production-ready reverse proxy and static file server using Caddy, orchestrated with Docker Compose.
- Files: `compose.yml` (service + volumes), `Caddyfile` (config), `site/` (static files served at `/`).
- TLS: Automatic HTTPS via Let’s Encrypt when using a real domain pointing to this host.

Prerequisites
- Docker: Install Docker Engine and Docker Compose (v2).
- Ports: Open inbound `80` and `443` on the host firewall and any cloud security groups.
- DNS: Point your domain/subdomain A/AAAA record(s) to this server’s IP.

One-Time Setup
- Create the external network used by the proxy: `docker network create caddy_net`
- Ensure folder structure exists next to this repo:
  - `Caddyfile` (provided)
  - `compose.yml` (provided)
  - `site/` (create if serving static files)

Production Profile (Hardened Defaults)
- Image pin: Using `caddy:2.8` for stability.
- Health checks: Admin API enabled on `:2019`, Compose healthcheck probes `/config`.
- Headers: HSTS, NoSniff, Referrer-Policy, Permissions-Policy set by default.
- Logging: JSON access logs to stdout, easy to ship to your log stack.
- Compression: `zstd` + `gzip` enabled.
- Security: `no-new-privileges`, `tmpfs /tmp`, data/config in named volumes.

Start / Stop / Status
- Start (or apply changes): `docker compose up -d`
- Stop: `docker compose down`
- Logs (follow): `docker compose logs -f caddy`
- Health: `docker inspect --format='{{json .State.Health}}' caddy | jq .`

Editing and Reloading Config
- Edit `Caddyfile`, then validate: `docker compose exec caddy caddy validate --config /etc/caddy/Caddyfile`
- Reload without restart: `docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile`

Data & Persistence
- `caddy_data` volume: Stores certificates and ACME account data (keep it between runs).
- `caddy_config` volume: Caddy runtime config state.
- `./site` bind mount to `/srv`: Static site content (if you use file_server).

Backups
- Certificates and ACME account: `docker run --rm -v caddy_data:/data -v "$PWD":/backup busybox tar -czf /backup/caddy_data.tgz -C / data`
- Restore: `docker run --rm -v caddy_data:/data -v "$PWD":/backup busybox sh -c 'rm -rf /data/* && tar -xzf /backup/caddy_data.tgz -C /'`

Caddyfile Basics
- Location: `Caddyfile`
- Compression: Add `encode zstd gzip` for better performance.
- Static file site example (serves `./site`):
  example.com {
    encode zstd gzip
    root * /srv
    file_server
  }
- Reverse proxy example to a container on the same Docker network:
  app.example.com {
    encode zstd gzip
    reverse_proxy app:3000
  }

Best-Production Caddyfile Template
- Global options and reusable headers are preconfigured in `Caddyfile`:
  - Set your ACME email in the top `{ ... }` block.
  - Replace `example.com` and `app.example.com` with your domains.
- Reverse proxy stanza includes timeouts and optional active health checks.
- Optional CSP is present; uncomment and tailor per app.

Serving Static Files
- Put your built static site into `site/`.
- Use the “Static file site example” in your `Caddyfile` replacing `example.com` with your domain.
- Apply changes: `docker compose up -d` then validate/reload as above.

Reverse Proxy to Other Containers
- Join your upstream app to the same external network `caddy_net` so Caddy can resolve it by name.
- In your app’s compose file, add:
  networks:
    - caddy_net
  networks:
    caddy_net:
      external: true
- Then target it in `Caddyfile` via `reverse_proxy app:PORT`, where `app` is the container/service name.
- For multiple apps/subdomains, add additional server blocks in `Caddyfile`.
- If your app exposes a health endpoint (e.g., `/healthz`), set `health_uri /healthz` inside `reverse_proxy`.

Local Development Notes
- Using real domains locally is uncommon; for `localhost` Caddy can issue local certs, but Let’s Encrypt requires a public domain.
- For quick local HTTP only, you can use an entry like:
  localhost {
    root * /srv
    file_server
  }

DNS and Certificates
- Ensure DNS A/AAAA records point to this server before first start to allow automatic TLS issuance.
- If using a CDN/proxy (e.g., Cloudflare), disable proxying (grey cloud) on first issuance, then re-enable if desired.
- Renewal happens automatically; keep ports 80/443 reachable.
- Set a real email in `Caddyfile` global options for ACME notices.

Updating Caddy Image
- Pull new image and recreate: `docker compose pull && docker compose up -d`

Troubleshooting
- Port conflicts: Ensure no other process binds `80/443` (e.g., stop `nginx`, `apache`).
- ACME/HTTP challenge failures: Check DNS points to this host, port 80 open, and CDN proxy temporarily off during first issuance.
- Validate config: `docker compose exec caddy caddy validate --config /etc/caddy/Caddyfile`
- Inspect active config: `docker compose exec caddy caddy adapt --pretty --config /etc/caddy/Caddyfile`
- Healthcheck failing: Ensure `admin :2019` remains enabled in the global block.

Reference
- Compose mounts `./Caddyfile` → `/etc/caddy/Caddyfile`, `./site` → `/srv`.
- External network name: `caddy_net` (must exist: `docker network create caddy_net`).
- Caddy official image docs: `docker pull caddy`
 - Image pinned: `caddy:2.8`
