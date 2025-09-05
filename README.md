# Caddy Webserver via Docker Compose

Simple, production-ready reverse proxy and static file server powered by Caddy, orchestrated with Docker Compose. Ships with hardened defaults, health checks, access logging, and easy config reloads.

For a deeper, step-by-step guide, see `guide.md`.

## Features
- Automatic HTTPS via Let’s Encrypt (with a real domain)
- Reverse proxy and/or static file hosting
- Security headers, compression, JSON access logs
- Health-checked, reloadable config through Caddy Admin API
- Persistent volumes for certs and runtime config

## Prerequisites
- Docker Engine + Docker Compose v2
- Open inbound ports `80` and `443`
- DNS A/AAAA record pointing your domain to this server

## Quick Start
1. Create the external Docker network once:
   ```bash
   docker network create caddy_net
   ```
2. Adjust `Caddyfile`:
   - Set a real email in the global options block
   - Replace `example.com` (and `app.example.com` if using reverse proxy) with your domain(s)
3. Optional: create a `site/` directory if serving static files.
4. Start the stack:
   ```bash
   docker compose up -d
   ```

## Repository Layout
- `compose.yml`: Service, volumes, healthcheck, and network
- `Caddyfile`: Caddy configuration (global options, headers, examples)
- `site/`: Static files served at `/` when using the static site block
- `guide.md`: In-depth instructions, tips, and troubleshooting

## Common Operations
- Apply changes / start: `docker compose up -d`
- Stop stack: `docker compose down`
- Tail logs: `docker compose logs -f caddy`
- Validate config: `docker compose exec caddy caddy validate --config /etc/caddy/Caddyfile`
- Reload config: `docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile`
- Health status: `docker inspect --format='{{json .State.Health}}' caddy | jq .`

## Static File Hosting
Example server block (already present in `Caddyfile`):
```caddy
example.com {
  import common_headers
  root * /srv
  file_server
}
```
- Put your built static site into `./site`.
- Replace `example.com` with your domain.

## Reverse Proxy
Example server block (already present in `Caddyfile`):
```caddy
app.example.com {
  import common_headers

  @health path /healthz
  handle @health {
    respond 200
  }

  reverse_proxy app:3000 {
    # health_uri /healthz
    lb_policy least_conn
    fail_duration 10s
    max_fails 3
    transport http {
      read_timeout 30s
      write_timeout 30s
    }
  }
}
```
- Join your upstream container to the same external network `caddy_net`.
- Target it by service name and port (e.g., `app:3000`).

In your upstream compose file:
```yaml
networks:
  - caddy_net

networks:
  caddy_net:
    external: true
```

## Volumes & Persistence
- `caddy_data`: Certificates and ACME account data (persist across restarts)
- `caddy_config`: Caddy runtime config state
- `./site` → `/srv`: Static content bind mount

### Back up certificates
```bash
docker run --rm -v caddy_data:/data -v "$PWD":/backup busybox tar -czf /backup/caddy_data.tgz -C / data
```

### Restore certificates
```bash
docker run --rm -v caddy_data:/data -v "$PWD":/backup busybox sh -c 'rm -rf /data/* && tar -xzf /backup/caddy_data.tgz -C /'
```

## Updating Caddy
```bash
docker compose pull && docker compose up -d
```

## Troubleshooting
- Port conflicts: Ensure no other service binds `80/443` (stop nginx/apache)
- ACME challenges: Confirm DNS points here and port 80 is reachable; disable CDN proxying for first issuance
- Validate config: `docker compose exec caddy caddy validate --config /etc/caddy/Caddyfile`
- Inspect adapted config: `docker compose exec caddy caddy adapt --pretty --config /etc/caddy/Caddyfile`
- Keep `admin :2019` enabled for health checks and safe reloads

---
See `guide.md` for additional details, recommendations, and templates.

