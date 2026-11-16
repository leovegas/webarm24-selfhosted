# 02 — Install

About 20 minutes, most of it spent on the first image pull. Every step shows what you should see. If it differs, the step links to the fix.

> Using an AI agent? Give it `rollout/ROLLOUT.md` instead: the same steps with machine-checkable results.

## 1. Prepare the server

```bash
# Ubuntu/Debian: Docker Engine + Compose plugin (skip if `docker compose version` already works)
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker "$USER" && newgrp docker
docker compose version          # → Docker Compose version v2.x
```

Create a DNS record `automation.example.com → <server public IP>`. Then open ports 80 and 443 (for example `sudo ufw allow 80,443/tcp`).

## 2. Get the install kit

Clone it from GitHub (the `stable` branch is the latest release; `vX.Y.Z` tags match image versions):
```bash
sudo mkdir -p /opt/webarm && sudo chown "$USER" /opt/webarm
git clone --branch stable https://github.com/leovegas/webarm24-selfhosted.git /opt/webarm
```
Or unpack the bundle tarball from your purchase e-mail or the GitHub Release. Use this on servers without GitHub access:
```bash
sudo mkdir -p /opt/webarm && sudo chown "$USER" /opt/webarm
tar -xzf webarm-selfhost-bundle-<version>.tgz -C /opt/webarm --strip-components=1
```
Then:
```bash
cd /opt/webarm
cp ~/Downloads/license.key ./license.key       # from your purchase e-mail
ls
# → AGENTS.md  CLAUDE.md  Caddyfile.template  README.md  VERSION  docker-compose.yml  docs  rollout  skills  webarmctl  .env.example
```

## 3. Preflight

```bash
./webarmctl preflight
```
Expected: `OK` for docker, compose, architecture, memory, disk, user namespaces and ports, and `OK` (or `WARN`, before DNS propagates) for DNS. Fix every `FAIL` using the `fix:` line printed under it.

## 4. Install

```bash
export WEBARM_REGISTRY_USER=lic_xxxxxxxxxxxxxxxx      # license id
export WEBARM_REGISTRY_PASSWORD=wreg_xxxxxxxxxxxxxxxx # registry password from the e-mail
PUBLIC_HOST=automation.example.com ADMIN_EMAILS=you@example.com LICENSE_FILE=./license.key ./webarmctl install
```

What happens:
1. `.env` is created from `.env.example` (mode 600). `POSTGRES_PASSWORD`, `JWT_SECRET` and `PIPELINE_FIELD_ENCRYPTION_KEY` are generated.
2. The license key is copied to `license/webarm_license`.
3. `Caddyfile` is rendered.
4. The tool logs in to the registry and pulls the image (~6 GiB).
5. `db`, `app` and `caddy` start, and the tool waits until the app is healthy.

Expected ending:
```
  OK   app healthy
  OK   license — valid
Next: open https://automation.example.com/setup — sign up with you@example.com, activate the license.
```

Did you leave out `LICENSE_FILE`? Then the license line shows `WARN license — missing (WA-LIC-001)`. That is fine, you will paste the key in step 6.

## 5. Add an AI provider key

```bash
nano .env          # set ANTHROPIC_API_KEY=... (or OPENAI_API_KEY / OPENROUTER_API_KEY), optionally SMTP_*
./webarmctl restart
```

## 6. Create the admin account

1. Open `https://automation.example.com/setup`.
2. Enter the e-mail from `ADMIN_EMAILS` and a password, then click **Create admin account**. Only `ADMIN_EMAILS` addresses can sign up before the license is active.
3. If the status pill is not **valid**, paste the content of `license.key` and click **Activate**.
4. Click **Open the app**.

## 7. Verify

```bash
./webarmctl doctor
SMOKE_ADMIN_EMAIL=you@example.com SMOKE_ADMIN_PASSWORD='…' ./webarmctl smoke
```
`doctor` should be all `OK`. A `WARN` for SMTP or backups is expected at this point. `smoke` creates and runs a JS pipeline plus a sandboxed Python step, and checks MCP and the API schema. It ends with `N/N checks passed`.

## 8. Back up, and schedule it

```bash
./webarmctl backup
( crontab -l 2>/dev/null; echo '17 3 * * * cd /opt/webarm && ./webarmctl backup --json >> backups/backup.log 2>&1' ) | crontab -
```
Copy the backups off the server ([09](09-backup-restore.md)). **Store `.env` safely**: without `PIPELINE_FIELD_ENCRYPTION_KEY`, stored secrets cannot be decrypted.

## 9. Invite your team

Users sign up at `https://automation.example.com/pipelines` until all licensed seats are used. Admin tools are at `/admin`. For coding agents, each user creates an MCP key from the **MCP** button. The endpoint is `https://automation.example.com/mcp`.

## Go-live checklist
See `rollout/checklists/go-live.md`.
