# 09 — Backup & restore

## What a backup contains

`./webarmctl backup` writes `backups/webarm-<UTC timestamp>.tgz` (mode 600). It contains:

| File | Content |
|---|---|
| `db.dump` | `pg_dump -Fc` of the whole database: users, pipelines, workflows, runs, history, schedules, stores, skills, encrypted secrets |
| `appdata.tgz` | the `appdata` volume: stored files, browser state, install state. It leaves out the rebuildable Python package caches |
| `env` | your `.env`, **including secrets and `PIPELINE_FIELD_ENCRYPTION_KEY`** |
| `license.key` | your license key |
| `custom.tgz` | your legal pages |
| `manifest.json` | timestamp, image, bundle version |

The last `BACKUP_KEEP` backups are kept (default 14).

> Encrypted fields in the database can only be read with the same `PIPELINE_FIELD_ENCRYPTION_KEY`. A database dump without the matching `.env` is only partly usable. Keep both together, and keep them **encrypted** off the server.

## Schedule

```bash
crontab -e
17 3 * * * cd /opt/webarm && ./webarmctl backup --json >> backups/backup.log 2>&1
```

## Copy off the server

Encrypt the backup and ship it somewhere else:
```bash
# age (https://age-encryption.org) — recipient = your public key
age -r age1yourpublickey… -o /mnt/offsite/webarm-$(date +%F).tgz.age backups/$(ls -1t backups | grep tgz | head -1)
# or restic / borg / rclone to S3 — exclude nothing, the archive is already compressed
```
Test that you can decrypt it.

## Restore

```bash
./webarmctl restore backups/webarm-20261115-031700.tgz     # asks for confirmation (or --yes)
```

Steps:
1. Unpack the backup and check it.
2. If the backup's encryption key differs from the current `.env`, the backup's `.env` is restored and the current one is saved as `.env.before-restore`.
3. Stop the app.
4. Restore into a temporary database. If that fails, **the current database is untouched**.
5. Swap the temporary database in.
6. Replace stored files, restore the license and custom pages.
7. Start the app and wait until it is healthy.

### Onto a new server (migration / disaster recovery)

1. Install the bundle on the new server and run `./webarmctl preflight`.
2. Copy the backup and the bundle `.env` to the new server.
3. Run `./webarmctl install`. It will reuse `.env` if it is present.
4. Run `./webarmctl restore <backup> --yes`.
5. Point DNS at the new server, then run `./webarmctl doctor`.

## DR drill (do it quarterly)

Follow `rollout/checklists/dr-drill.md`. Restore the latest backup to a scratch server, log in, open a pipeline, and run it.
