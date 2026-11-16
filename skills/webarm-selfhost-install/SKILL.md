---
name: webarm-selfhost-install
description: Install WebArm24 Self-hosted on a Linux server from the customer bundle. Covers preflight, .env and secrets, registry login, the first start, the admin account, license activation, a smoke test and the first backup. Use when the user asks to install, set up, deploy or "roll out" WebArm24 / the self-hosted automation platform, or runs into trouble during the first install.
---

# Install WebArm24 Self-hosted

You are operating a production server. Follow `rollout/ROLLOUT.md` steps R01–R10 in order. Use `--json` on every `webarmctl` call and decide from the JSON, not from the coloured text.

## Hard rules
- **Ask the human** (and wait) before you:
  - change DNS, the firewall or sysctl;
  - install Docker;
  - touch existing Docker volumes;
  - run anything destructive: `uninstall --purge`, `down -v`, `volume rm`, `restore`.
- Never echo, log or commit secrets: `.env` values, the license key, the registry password, provider keys. Refer to them by variable name.
- Don't edit `docker-compose.yml`. Use `.env`, or `docker-compose.override.yml` with approval.
- Same error twice means stop and report: step id, command, JSON, `./webarmctl logs app | tail -80`.

## Inputs you need (ask for whatever is missing)
- `PUBLIC_HOST` (DNS name) and the TLS mode: public Let's Encrypt (default), or `TLS_MODE=internal` for a LAN.
- `ADMIN_EMAILS` (at least one; two recommended).
- The license file path (`license.key`, starts with `WA1.`).
- Registry user (`lic_…`) and password (`wreg_…`). Pass them as `WEBARM_REGISTRY_USER` / `WEBARM_REGISTRY_PASSWORD` env vars only.
- An AI provider key (Anthropic, OpenAI or OpenRouter). SMTP details are optional.

## Procedure (summary — the details are in rollout/ROLLOUT.md)
1. `./webarmctl preflight --json`. Every `ok:false` must be fixed. Show `.fix` to the human.
2. Registry login: `printf '%s' "$WEBARM_REGISTRY_PASSWORD" | docker login registry.webarm24.online -u "$WEBARM_REGISTRY_USER" --password-stdin`. Skip it if the image was `docker load`ed (air-gapped, docs/13).
3. `PUBLIC_HOST=… ADMIN_EMAILS=… LICENSE_FILE=… ./webarmctl install --json`. Expect `ok:true` and `app healthy`. The first run creates `.env` with generated secrets (mode 600).
4. Put the AI key (and SMTP) into `.env` without printing it, then run `./webarmctl restart --json`.
5. The human opens `https://PUBLIC_HOST/setup` and creates the admin account. The e-mail must be in `ADMIN_EMAILS`.
6. `./webarmctl license status --json` → `license == "valid"`. If not: `./webarmctl license set <file> --json`. Error codes are in `docs/04-license.md`.
7. `SMOKE_ADMIN_EMAIL=… SMOKE_ADMIN_PASSWORD=… ./webarmctl smoke --json` → `ok:true`. The human types the password.
8. `./webarmctl backup --json` and install the daily cron (R09). Remind the human that backups contain secrets and must be copied off the server, encrypted.
9. `./webarmctl doctor --json` → hand over: URL, version, license, seats, warnings, backup location.

## Common failures
| JSON says | Do |
|---|---|
| `user namespaces … ok:false` | ask, then `sysctl -w user.max_user_namespaces=15000` (+ persist), docs/12#sandbox |
| `port 80/443 already in use` | another web server is running: ask whether to stop it or use HTTP_PORT/HTTPS_PORT + their proxy (docs/05) |
| `dns … does not resolve` | the human creates the A/AAAA record; wait, then re-run preflight |
| install `app healthy ok:false` | `./webarmctl logs app \| tail -80`; `password authentication failed` = an old db volume → ask the human |
| license `missing` / `WA-LIC-001` | `./webarmctl license set license.key --json` |
| license `outdated` / `WA-LIC-002` | the image is newer than the update window → human: renew, or `./webarmctl upgrade <older X.Y.Z>` |
| smoke `python step output (sandbox)` failed | docs/12-troubleshooting.md#sandbox |

## References
- `rollout/ROLLOUT.md`, `rollout/rollout.yaml` (machine-readable)
- `docs/02-install.md` (human version), `docs/03-configuration.md` (every variable)
