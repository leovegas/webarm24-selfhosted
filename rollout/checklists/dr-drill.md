# Disaster-recovery drill (quarterly)

Goal: prove that a backup + `.env` restore to a fresh server in < 1 hour.

1. Provision a scratch server (same OS family), install Docker, copy the bundle.
2. Copy the newest off-site backup and decrypt it.
3. `./webarmctl preflight`
4. Extract `env` from the backup to `.env` (`tar -xzf backup.tgz ./env && mv env .env`), set `PUBLIC_HOST` to a scratch name, `TLS_MODE=internal`.
5. `./webarmctl install --yes`, then `./webarmctl restore <backup> --yes`
6. Sign in with a real account; open three pipelines; run one; check a stored file downloads; check a saved browser login decrypts.
7. Record: start/end time, problems, backup age. Destroy the scratch server (license terms: one production install).
