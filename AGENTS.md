# AGENTS.md — operating rules for AI agents on this server

This directory holds a **production** WebArm24 Self-hosted install (or its bundle before installation). You are allowed to operate it through `./webarmctl` and to read the docs. Follow these rules strictly.

## Where things are
| Path | What |
|---|---|
| `webarmctl` | the only supported way to operate the install. Run `./webarmctl help`; add `--json` to everything |
| `rollout/ROLLOUT.md`, `rollout/rollout.yaml` | step-by-step install / upgrade / restore runbooks with expected results |
| `skills/` (also `.claude/skills/`) | task skills: install, upgrade, backup-restore, troubleshoot, license, configure |
| `docs/` | full documentation (start at `docs/README.md`) |
| `.env` | configuration **with secrets** (mode 600) |
| `license/webarm_license` | license key (secret) |
| `backups/` | backups (they contain secrets) |
| `docker-compose.yml`, `Caddyfile.template` | stack definition. Don't edit them |

## Always
- Use `./webarmctl … --json` and decide from the JSON (`.ok`, `.checks[].ok`, `.fix`).
- Run `./webarmctl doctor --json` before and after any change, and report the difference.
- Keep the human informed: what you are about to do, and the result.

## Never without explicit human approval in this conversation
- `./webarmctl restore`, `./webarmctl uninstall`, `docker compose down -v`, `docker volume rm`, `docker system prune --volumes`, deleting files in `backups/` or `license/`
- changing DNS, the firewall, sysctl, TLS mode, published ports, or security settings (`PIPELINE_PYTHON_NETWORK=host`, `AI_AGENT_ALLOW_PRIVATE_NETWORK`, `SELFHOST_ALLOW_ANONYMOUS`)
- `./webarmctl upgrade` / `rollback` (they restart the service)
- editing `docker-compose.yml`

## Never at all
- Print, log, commit, upload or paste secrets: `.env` values, the license key, registry tokens, provider keys, backup contents.
- Change `PIPELINE_FIELD_ENCRYPTION_KEY`.
- Try to modify, decompile or bypass the license, the edition or the image.
- Run `docker compose` against this project without `-p webarm --env-file .env`. Prefer `webarmctl`.

## When stuck
Same error twice means stop. Report the command, the JSON and `./webarmctl logs app | tail -80`, and offer `./webarmctl support-bundle`.
