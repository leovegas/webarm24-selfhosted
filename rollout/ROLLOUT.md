# WebArm24 Self-hosted — rollout runbook (agent-ready)

This runbook is for an AI coding agent (Claude Code, Codex, …) or a human operator with a shell on the target server. Each step is deterministic: a **precondition**, an exact **command**, an **expected** machine-checkable result, what to do **on failure**, and how to **roll back**. `rollout.yaml` holds the same steps in machine-readable form.

## Rules for agents (read first)

1. Work from the bundle directory (default `/opt/webarm`). Run every command with `--json` when the step says so, and parse the JSON. Don't scrape the coloured text.
2. **STOP and ask the human** at every step marked 🛑. Those steps involve DNS, purchases, secrets the human must supply, destructive actions, or exposing the server.
3. Never print, log, commit or paste the values of `.env`, `license/webarm_license`, registry passwords or API keys. To refer to a value, name the variable.
4. Never run `docker compose down -v`, `./webarmctl uninstall --purge`, `docker volume rm`, or `./webarmctl restore` without explicit human approval in this conversation.
5. Don't edit `docker-compose.yml`. Configuration goes into `.env`; structural changes go into `docker-compose.override.yml`, with human approval.
6. If a step fails twice with the same error, stop. Report the step id, the command, the JSON output and the relevant `./webarmctl logs app | tail -80`.
7. Exit codes: `0` ok · `1` check failed · `2` usage error · `3` a human decision is needed (re-run with `--yes` only if the human agreed).

## Inputs to collect before starting (🛑 ask the human for each missing one)

| Input | Used in | Example |
|---|---|---|
| `PUBLIC_HOST` | R03, R05 | `automation.acme.com` |
| `ADMIN_EMAILS` | R05, R07 | `it-admin@acme.com` |
| license file path | R05 | `./license.key` |
| registry user / password | R05 | `lic_…` / `wreg_…` (purchase e-mail) |
| AI provider + key | R06 | `ANTHROPIC_API_KEY` |
| SMTP (optional) | R06 | host, port, user, password, from |
| TLS mode | R03, R05 | public Let's Encrypt (default) or `internal` |
| admin password for the smoke test | R08 | the human types it, or supplies it via env |

---

### R01 — Server prerequisites
- **Precondition:** shell on the target host. The agent has root or a docker-group account.
- **Command:** `uname -m && docker version -f '{{.Server.Version}}' && docker compose version --short`
- **Expected:** `x86_64`, Docker ≥ 24, Compose ≥ 2.20.
- **On failure:** if Docker is missing, 🛑 ask before installing it (`curl -fsSL https://get.docker.com | sh`). If the architecture is not x86_64, stop: unsupported.
- **Rollback:** none.

### R02 — Get the install kit
- **Precondition:** the server can reach github.com, **or** `webarm-selfhost-bundle-<v>.tgz` is on the server. Skip this step if you are already inside a clone or unpacked bundle (`./webarmctl` and `VERSION` exist).
- **Command (git):** `sudo mkdir -p /opt/webarm && sudo chown "$USER" /opt/webarm && git clone --branch stable https://github.com/leovegas/webarm24-selfhosted.git /opt/webarm && cd /opt/webarm && cat VERSION`
- **Command (tarball):** `sudo mkdir -p /opt/webarm && sudo chown "$USER" /opt/webarm && tar -xzf webarm-selfhost-bundle-*.tgz -C /opt/webarm --strip-components=1 && cd /opt/webarm && cat VERSION`
- **Expected:** prints a version `X.Y.Z`, and `./webarmctl` is executable.
- **On failure:** if `/opt/webarm` is not empty, 🛑 ask the human (it may be an existing install). For the tarball, check the archive with `tar -tzf`.
- **Rollback:** `rm -rf /opt/webarm` (only if nothing else is there).

### R03 — Preflight 🛑 (DNS and firewall are human decisions)
- **Command:** `PUBLIC_HOST=<host> ./webarmctl preflight --json`
- **Expected:** `.ok == true`. A `warning: true` on `dns …` is acceptable only if the human confirms the record was just created.
- **On failure:** for each check with `.ok == false`, show the human `.check`, `.detail` and `.fix`. Don't change sysctl, firewall or DNS without approval. Typical fixes: user namespaces (sysctl), ports in use (another web server), DNS not pointing here.
- **Rollback:** none.

### R04 — Registry access
- **Precondition:** registry credentials available as `WEBARM_REGISTRY_USER` / `WEBARM_REGISTRY_PASSWORD` env vars. Don't write them to disk.
- **Command:** `printf '%s' "$WEBARM_REGISTRY_PASSWORD" | docker login registry.webarm24.online -u "$WEBARM_REGISTRY_USER" --password-stdin`
- **Expected:** `Login Succeeded`.
- **On failure:** `unauthorized` means a wrong or rotated token. 🛑 Ask the human to create a new one under **Your licenses** on webarm24.online/self-hosted. Air-gapped: skip this step and `docker load` the image tarball (docs/13).
- **Rollback:** `docker logout registry.webarm24.online`.

### R05 — Install
- **Command:** `PUBLIC_HOST=<host> ADMIN_EMAILS=<emails> LICENSE_FILE=<path> ./webarmctl install --json`. For a LAN install, first add `TLS_MODE=internal` to `.env`. If `.env` doesn't exist yet, run install once; it creates `.env`.
- **Expected:**
  - `.ok == true`;
  - checks contain `{"check":"app healthy","ok":true}`;
  - `.license == "valid"`. `"missing"` is acceptable if no license file was given; R07 handles it.
- **On failure:** run `./webarmctl logs app | tail -80`. If you see `password authentication failed`, a previous db volume exists with another password. 🛑 Ask the human before touching volumes.
- **Rollback:** `./webarmctl uninstall`. This keeps volumes. Removing data needs 🛑 approval.

### R06 — AI provider and SMTP
- **Precondition:** the human supplied the provider key (and SMTP details).
- **Command:** set the variables in `.env` without echoing them. For example, `./webarmctl` reads `.env`; edit it with `sed -i "s|^ANTHROPIC_API_KEY=.*|ANTHROPIC_API_KEY=$KEY|" .env` while the value is held in an env var. Then run `./webarmctl restart --json`.
- **Expected:** `.ok == true`. In `./webarmctl doctor --json`, the check `AI provider key` has `ok:true` and no warning.
- **On failure:** check the key with the provider. Run `./webarmctl logs app | grep -i "anthropic\|openai"`.
- **Rollback:** restore the previous `.env` line, then `./webarmctl restart`.

### R07 — Admin account and license activation 🛑 (the human chooses the password)
- **Command (human, in a browser):** open `https://<host>/setup`, then **Create admin account** with an `ADMIN_EMAILS` address. If the status is not *valid*, paste the key and click **Activate**.
- **Agent alternative for the key:** `./webarmctl license set <path> --json` gives `.ok == true`.
- **Expected:** `./webarmctl license status --json` → `.license == "valid"`.
- **On failure:** look up `.code` in docs/04-license.md#error-codes. `WA-LIC-002`: the image is newer than the update window, so 🛑 the human decides whether to renew or to pin an older tag.
- **Rollback:** none.

### R08 — Smoke test
- **Precondition:** an admin account exists. `SMOKE_ADMIN_EMAIL` / `SMOKE_ADMIN_PASSWORD` are in the environment, typed by the human.
- **Command:** `./webarmctl smoke --json`
- **Expected:**
  - `.ok == true`;
  - checks include `run succeeded`, `js step output`, `python step output (sandbox)`, `mcp tools/list`.
- **On failure:**
  - `python step output (sandbox)` failed: see docs/12-troubleshooting.md#sandbox.
  - `admin login` failed: wrong credentials.
- **Rollback:** none. The smoke test deletes its own pipeline.

### R09 — First backup and schedule
- **Command:** `./webarmctl backup --json`, then install the cron entry:
  `( crontab -l 2>/dev/null | grep -v 'webarmctl backup'; echo '17 3 * * * cd /opt/webarm && ./webarmctl backup --json >> backups/backup.log 2>&1' ) | crontab -`
- **Expected:** `.ok == true` and `.file` exists. `crontab -l` shows the line.
- **On failure:** check disk space and the db container (`./webarmctl logs db`).
- **Rollback:** remove the cron line.
- 🛑 Remind the human: backups contain `.env` secrets. Copy them off the server, encrypted (docs/09).

### R10 — Final health and handover
- **Command:** `./webarmctl doctor --json`
- **Expected:** `.ok == true`. Report every remaining warning (SMTP, backups age, TLS) to the human.
- **Handover report (agent output):**
  - URL `https://<host>`;
  - version (`./webarmctl version --json`);
  - license state, seats and `updates_until` (from `/setup`);
  - where `.env` and backups are;
  - the cron entry;
  - open warnings;
  - next steps: invite users, MCP keys, upgrade policy (docs/08).

---

## Upgrade runbook (U-steps)

| Id | Step | Command | Expected | On failure |
|---|---|---|---|---|
| U01 | read release notes | `cat docs/CHANGELOG.md` (from the new bundle) | note any `.env` changes | 🛑 ask if notes require manual steps |
| U02 | health before | `./webarmctl doctor --json` | `.ok == true` | fix first; don't upgrade a sick install |
| U03 | upgrade | `./webarmctl upgrade <X.Y.Z> --json` | `.ok == true`, `.to == "<X.Y.Z>"` | output has `rolled_back_to` → report logs; `WA-LIC-002` → 🛑 human: renew or choose an older tag |
| U04 | smoke | `./webarmctl smoke --json` | `.ok == true` | `./webarmctl rollback --yes` (🛑 confirm first) |
| U05 | report | `./webarmctl version --json` | new version running | — |

## Restore runbook (D-steps) 🛑 always needs explicit human approval

| Id | Step | Command | Expected |
|---|---|---|---|
| D01 | pick backup | `ls -1t backups/webarm-*.tgz \| head` | human picks the file |
| D02 | restore | `./webarmctl restore <file> --yes --json` | `.ok == true` |
| D03 | verify | `./webarmctl doctor --json` and `./webarmctl smoke --json` | both `.ok == true` |
