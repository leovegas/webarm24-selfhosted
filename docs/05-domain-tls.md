# 05 — Domain & TLS

## Default: automatic Let's Encrypt

1. Create a DNS record: `PUBLIC_HOST` → the server's public IPv4 (A) and/or IPv6 (AAAA).
2. Ports 80 and 443 must be reachable from the internet.
3. Run `./webarmctl start`. Caddy obtains the certificate on first request and renews it automatically.

Check: `./webarmctl doctor` → `OK tls certificate — expires …`. If issuance fails, run `./webarmctl logs caddy`. Look for `challenge failed`, which usually means DNS isn't propagated yet or port 80 is blocked.

## LAN or intranet without public DNS

Set `TLS_MODE=internal` in `.env` and run `./webarmctl restart`. Caddy then issues certificates from its own CA. Browsers will warn until you distribute the root certificate:
```bash
docker compose -p webarm cp caddy:/data/caddy/pki/authorities/local/root.crt ./webarm-root.crt
```
Install `webarm-root.crt` into your company trust store (via GPO or MDM).

## Your own certificate

Mount it and replace the `@@TLS@@` line in `Caddyfile.template` with `tls /certs/fullchain.pem /certs/privkey.pem`. Then add the volume `- ./certs:/certs:ro` to the `caddy` service with a `docker-compose.override.yml`:
```yaml
services:
  caddy:
    volumes:
      - ./certs:/certs:ro
```
Run `./webarmctl restart`.

## Behind your own reverse proxy or load balancer

Turn off the bundled Caddy, or keep it on other ports, and proxy to the app.

**Option A: keep Caddy on internal ports.** Set `HTTP_PORT=8080`, `HTTPS_PORT=8443`, `TLS_MODE=internal` and proxy to `https://server:8443`. This is the simplest option.

**Option B: expose the app directly.** Use a `docker-compose.override.yml`:
```yaml
services:
  app:
    ports: ["127.0.0.1:8000:8000"]
  caddy:
    profiles: ["disabled"]
```

Requirements for any proxy:
- Pass `Host` and `X-Forwarded-Proto: https`. The app trusts forwarded headers, so OAuth and e-mail links come out as `https://PUBLIC_HOST`.
- **WebSockets** on `/ws/…` (the chat), and **no buffering** for `/mcp` and SSE responses.
- Body size ≥ 250 MB (stored files) and read timeout ≥ 15 min (long agent calls).

nginx:
```nginx
location / {
    proxy_pass http://127.0.0.1:8000;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_buffering off;
    proxy_read_timeout 900s;
    client_max_body_size 250m;
}
```

Traefik (labels on `app`):
```yaml
labels:
  - traefik.enable=true
  - traefik.http.routers.webarm.rule=Host(`automation.example.com`)
  - traefik.http.routers.webarm.tls.certresolver=le
  - traefik.http.services.webarm.loadbalancer.server.port=8000
  - traefik.http.services.webarm.loadbalancer.healthcheck.path=/health
```

Health endpoints for load balancers:
- `/health` returns 200 whenever the process is up.
- `/ready` returns 200 when the instance accepts work, and 503 while starting or draining. Its body includes `license` and `version`.

## Changing the domain

Set the new `PUBLIC_HOST` and run `./webarmctl restart`. Then update DNS, the Google OAuth redirect URI and any inbound webhook URLs you gave out. Webhook tokens stay the same; only the host changes.
