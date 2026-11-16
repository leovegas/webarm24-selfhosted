# 12 — Troubleshooting

Start with:
```bash
./webarmctl doctor          # every check prints a fix
./webarmctl logs app | tail -100
```
Still stuck? Run `./webarmctl support-bundle` and send the archive to support. Secrets are redacted.

## By symptom

| Symptom | Likely cause | Fix |
|---|---|---|
| Every page redirects to `/setup`; API answers 503 `WA-LIC-00x` | license not active | [04 — error codes](04-license.md#error-codes) |
| `/setup` says "Only an address listed in ADMIN_EMAILS can sign up" | not activated yet and the e-mail isn't an admin | add it to `ADMIN_EMAILS` in `.env`, `./webarmctl restart` |
| Signed in but "not an administrator" on `/setup` | the account existed before you added it to `ADMIN_EMAILS` | `./webarmctl restart` (admins are applied on start) |
| Sign-up fails with `WA-LIC-005` | all seats used | remove a user at `/admin` or buy more seats |
| API returns 401 "Sign in required" | self-hosted requires an account for `/api` | sign in; for scripts use a user JWT, an MCP key on `/mcp`, or webhook URLs |
| `upgrade` stops with `WA-LIC-002` | the new image is newer than your update window | renew Updates and refresh the key, or upgrade to an older tag |
| App starts, then logs `WA-LIC-002` | someone pulled a newer `stable` | `./webarmctl rollback` |
| Browser warns about the certificate | `TLS_MODE=internal`, or Let's Encrypt failed | [05](05-domain-tls.md); `./webarmctl logs caddy` |
| `challenge failed` in caddy logs | DNS not pointing here, or port 80 blocked | fix DNS/firewall, then `./webarmctl restart` |
| UI loads without styling or the code editor is missing | the browser cannot reach Google Fonts / cdnjs | allow them in your proxy, or see [13](13-air-gapped.md) |
| WebSocket chat never answers | the proxy drops WebSocket upgrades | [05 — own proxy](05-domain-tls.md#behind-your-own-reverse-proxy-or-load-balancer) |
| Agent / AI steps: `401` / `invalid x-api-key` | wrong or missing provider key | [06](06-ai-providers.md) |
| E-mail steps fail | SMTP not configured | [07](07-email.md) |
| `pull access denied` / `unauthorized` | registry login missing or token rotated | `docker login registry.webarm24.online -u lic_… -p <token>` |
| App unhealthy right after install, logs show `database … does not exist` or `password authentication failed` | `POSTGRES_PASSWORD` changed after the db volume was created | restore the old value in `.env` (the db keeps its first password) |
| After restore, secrets show as garbage / decryption errors | backup came from a different `PIPELINE_FIELD_ENCRYPTION_KEY` | restore that backup's `.env` (`webarmctl restore` does it automatically) |

## <a id="sandbox"></a>Python steps fail: sandbox errors

Symptoms:
- `bwrap: No permissions to create new namespace`, `setting up uid map: Permission denied`, or `Creating new namespace failed`;
- `./webarmctl doctor` → `FAIL python sandbox (bubblewrap)`.

Work through these in order:
1. **Compose security options are intact.** The `app` service must keep `cap_add: [SYS_ADMIN, NET_ADMIN]` and `security_opt: [seccomp=unconfined, apparmor=unconfined]`. Run `docker inspect webarm-app-1 | grep -A3 CapAdd` to confirm.
2. **Kernel allows user namespaces:**
   ```bash
   sudo sysctl -w user.max_user_namespaces=15000
   sudo sysctl -w kernel.unprivileged_userns_clone=1        # Debian/Ubuntu kernels only
   echo -e "user.max_user_namespaces=15000\nkernel.unprivileged_userns_clone=1" | sudo tee /etc/sysctl.d/90-webarm.conf
   ```
3. **Ubuntu 24.04 AppArmor.** `apparmor=unconfined` on the container is required. Don't remove it. As a last resort on the host: `sudo sysctl -w kernel.apparmor_restrict_unprivileged_userns=0`.
4. **Inside a VM or LXC.** Nested containers need nesting enabled: Proxmox LXC `features: nesting=1,keyctl=1`, or use a full VM.
5. **Rootless Docker or Podman** is not supported. Use rootful Docker Engine.

Python steps that need network fail with `ProxyError` or `Connection refused`:
- `PIPELINE_PYTHON_NETWORK=none`: there is no network by design. Use `proxy`.
- Is the target a private IP, or on a port other than 80/443? Then the proxy refuses it on purpose. The step error lists blocked requests.
- The client ignores proxy env vars (raw sockets, `aiohttp` without `trust_env=True`). Use `requests`, `httpx` or `urllib`.

## Collecting evidence

```bash
./webarmctl status --json
./webarmctl doctor --json
./webarmctl logs app --since 30m > app.log
./webarmctl support-bundle
```
