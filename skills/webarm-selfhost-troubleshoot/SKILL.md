---
name: webarm-selfhost-troubleshoot
description: Diagnose and fix a broken or degraded WebArm24 Self-hosted install, using webarmctl doctor/status JSON, logs and the support bundle. Covers setup-mode redirects, WA-LIC error codes, TLS and certificate failures, Python sandbox (bubblewrap) errors, egress proxy blocks, database and SMTP problems, and failing AI steps. Use when the user says WebArm24 is down, slow, erroring, stuck on /setup, or a pipeline step fails on their own server.
---

# Troubleshoot WebArm24 Self-hosted

## 1. Collect facts (read-only, always safe)
```bash
./webarmctl status --json
./webarmctl doctor --json
./webarmctl logs app --since 30m 2>&1 | tail -150
```
Each failed check in `doctor` carries `.fix`. Start with the first failing check in this order: containers → database → license → sandbox → https → disk → AI/SMTP warnings.

## 2. Decision table
| Evidence | Cause | Action |
|---|---|---|
| `container app` not running / restarting | crash on start | logs: look for the first `Traceback` or `ERROR`. `.env` typo → fix it, then `./webarmctl restart`. `password authentication failed` → `POSTGRES_PASSWORD` was changed after the first start: restore the old value (ask the human) |
| `database` ok:false | db down or disk full | `./webarmctl logs db`; disk → see the disk row |
| `license` missing / `WA-LIC-001` | no key | `./webarmctl license set license.key --json` (the human provides the file) |
| `license` outdated / `WA-LIC-002` | the image is newer than the update window | `./webarmctl rollback` (ask first), or the human renews and refreshes the key |
| `license` invalid / `WA-LIC-003` | broken key | the human downloads `license.key` again |
| `license` revoked / `WA-LIC-004` | revoked by the vendor | escalate to the human / vendor |
| sign-ups fail `WA-LIC-005` | seats are full | the human removes users at /admin or buys seats |
| `python sandbox (bubblewrap)` ok:false | namespaces blocked | docs/12-troubleshooting.md#sandbox, steps 1→5. sysctl changes **need approval** |
| Python step error lists "blocked" egress | the target is private or on a non-80/443 port | by design. Ask whether to add an allow list or change `PIPELINE_PYTHON_NETWORK` (security decision → human) |
| `https://host` HTTP 000 / certificate errors | DNS, firewall, ACME | `./webarmctl logs caddy \| grep -i "error\|challenge"`; docs/05 |
| `disk free` ok:false | images, backups, stored files | `docker image prune -f` (safe); deleting old backups or user files → ask |
| `AI provider key` warning; AI steps 401 | key missing or wrong | the human supplies a key → `.env` → restart |
| `SMTP` warning; e-mail steps fail | no SMTP | docs/07 |
| API 401 "Sign in required" | an anonymous script | use a user JWT, an MCP key on /mcp, or a webhook URL; `SELFHOST_ALLOW_ANONYMOUS=true` only with human approval |
| UI unstyled / code editor missing | the browser can't reach fonts.googleapis.com / cdnjs.cloudflare.com | client-side proxy; docs/13 |

## 3. Fix safely
- Allowed without asking: `restart`, `start`, `logs`, `status`, `doctor`, `smoke`, `backup`, `license status`, `docker image prune -f`.
- Ask first: editing `.env` values that change behaviour or security, sysctl, firewall, DNS, `rollback`, `upgrade`, `restore`, deleting anything, `SELFHOST_ALLOW_ANONYMOUS`, `PIPELINE_PYTHON_NETWORK=host`, `AI_AGENT_ALLOW_PRIVATE_NETWORK=true`.
- After every fix: `./webarmctl doctor --json` → report what changed.

## 4. Escalate to vendor support
```bash
./webarmctl support-bundle --json     # → .file (secrets are redacted)
```
Tell the human to review the archive and send it to support. Include the failing step, the time, and what was tried. Never attach `.env`, backups or the license key.
