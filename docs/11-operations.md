# 11 — Operations

## Daily commands

| Task | Command |
|---|---|
| Overview | `./webarmctl status` |
| Everything with fixes | `./webarmctl doctor` |
| Logs | `./webarmctl logs app -f` · `logs caddy` · `logs db` |
| Restart | `./webarmctl restart` |
| Backup | `./webarmctl backup` |
| Diagnostics for support | `./webarmctl support-bundle` |

All of them accept `--json`, so you can feed them to monitoring or to an AI agent.

## Monitoring

| Probe | Healthy | Meaning when not |
|---|---|---|
| `GET https://PUBLIC_HOST/health` | 200 | process down → `./webarmctl logs app` |
| `GET https://PUBLIC_HOST/ready` | 200 and `"license":"valid"` | 503 = starting/draining; `license` ≠ valid = setup mode ([04](04-license.md)) |
| `./webarmctl doctor --json` | `"ok":true` | each failing check has a `fix` |
| disk space on the Docker root | > 10 GiB free | stored files, images, backups |
| backup age | < 26 h | the cron job is not running |

Example Uptime Kuma / Prometheus blackbox target: `https://PUBLIC_HOST/ready` with a keyword match on `"license":"valid"`.

`/ready` also reports `busy_total` and `busy` (running pipelines, workflows, agent jobs) and `browser_sessions`. Use them for capacity planning.

## Capacity and tuning

| Symptom | Setting |
|---|---|
| Workflows queue behind each other | `WORKFLOW_MAX_CONCURRENCY` (default 4); more RAM |
| Python step killed: `MemoryError` / exit 137 | `PIPELINE_PYTHON_MEMORY_MAX_BYTES` (e.g. 2147483648) |
| Python step `timed out` | `PIPELINE_PYTHON_STEP_TIMEOUT_SEC` (e.g. 120; 600 for video) |
| Python step can't write a big file | `PIPELINE_PYTHON_FILE_MAX_BYTES` |
| Users hit stored-file limits | `USER_FILES_TOTAL_MAX_BYTES`, `USER_FILES_FILE_MAX_BYTES` |
| Model calls rate-limited | `LLM_REQUESTS_PER_MINUTE`; provider quota |
| Chromium crashes | keep `shm_size: 1gb`; more RAM; fewer parallel browser steps |

The app runs as one process (one container). It scales vertically. Postgres can be moved to a managed service: set `DATABASE_URL` in a `docker-compose.override.yml` and remove the `db` dependency. Ask support before you do this.

## Users and admins

- Admins are the accounts listed in `ADMIN_EMAILS`. They are applied at every start.
- `/admin` shows usage stats and the user list (delete a user, toggle admin) and lets you send broadcast e-mails to users. **License** links to `/setup`.
- Seats: see [04](04-license.md#seats).

## Housekeeping

- Run `docker image prune -f` after upgrades to remove old images. Keep the previous tag if you may need to roll back.
- Finished HTTP agent jobs are pruned automatically. Pipeline and workflow run history stays in the database, and pipeline version history is capped by `PIPELINE_HISTORY_MAX_REVISIONS`.
- The `knowledge` volume (site crawl cache) can be deleted at any time.
