---
name: webarm-selfhost-backup-restore
description: Back up, schedule backups, copy them off-site, restore or migrate a WebArm24 Self-hosted install, and run a disaster-recovery drill. Use when the user asks about backups, restore, migration to a new server, disaster recovery, "we lost the server", or moving WebArm24 between hosts.
---

# Backup & restore WebArm24 Self-hosted

## Facts
- `./webarmctl backup --json` writes `backups/webarm-<UTC>.tgz` (mode 600) and prints its path in `.file`. It contains:
  - `db.dump` (pg_dump -Fc);
  - `appdata.tgz` (stored files);
  - `env` (**all secrets, including `PIPELINE_FIELD_ENCRYPTION_KEY`**);
  - `license.key`, `custom.tgz`, `manifest.json`.
- Encrypted DB fields can only be read with the same `PIPELINE_FIELD_ENCRYPTION_KEY`. The dump and the `.env` belong together.
- Retention: the newest `BACKUP_KEEP` (default 14) are kept in `BACKUP_DIR` (default `./backups`).

## Make and schedule backups (safe — no approval needed)
```bash
./webarmctl backup --json
( crontab -l 2>/dev/null | grep -v 'webarmctl backup'; echo '17 3 * * * cd /opt/webarm && ./webarmctl backup --json >> backups/backup.log 2>&1' ) | crontab -
```
Off-site copies: **ask the human** which tool and destination to use (age/gpg + rsync/S3/restic/borg). Always encrypt; the archive contains secrets. Never upload unencrypted backups and never print their contents.

## Restore (DESTRUCTIVE — explicit human approval required in this conversation)
1. `ls -1t backups/webarm-*.tgz | head`. The human picks one. Show its `manifest.json`: `tar -xzOf <file> ./manifest.json`.
2. Confirm with the human: "This replaces the current database and stored files with the backup from <created>."
3. `./webarmctl restore <file> --yes --json`.
   - It restores into a temporary database first. If that fails, the current database is untouched.
   - If the backup's encryption key differs from the current one, it restores the backup's `.env` and keeps the old one as `.env.before-restore`. Tell the human.
4. `./webarmctl doctor --json` and `./webarmctl smoke --json` → both `ok:true`.

## Migrate to a new server
1. New server: bundle + `./webarmctl preflight --json` (skill `webarm-selfhost-install`, R01–R03).
2. Copy the newest backup over, and take its `.env`: `tar -xzf <backup> ./env && mv env .env && chmod 600 .env`. If the DNS name changes, adjust `PUBLIC_HOST`.
3. `./webarmctl install --yes --json` (it reuses `.env`), then `./webarmctl restore <backup> --yes --json` (with approval).
4. The human switches DNS. Then `./webarmctl doctor --json`.
5. Shut down the old install. License terms allow one production install per license.

## DR drill
Follow `rollout/checklists/dr-drill.md` and report the timings and any problems.
