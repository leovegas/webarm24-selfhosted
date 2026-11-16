---
name: webarm-selfhost-upgrade
description: Upgrade, pin or roll back a WebArm24 Self-hosted install with webarmctl. Covers reading the changelog, the license/update-window check, the pre-upgrade backup, drain, switch, smoke test and rollback. Use when the user asks to upgrade, update, patch or roll back WebArm24, change the image tag, or apply a new bundle, or when the app reports WA-LIC-002 after a pull.
---

# Upgrade WebArm24 Self-hosted

Use the U-steps in `rollout/ROLLOUT.md`. Every command takes `--json`.

## Before you start
1. Find the target version. The human tells you, or it is the newest in the new bundle's `docs/CHANGELOG.md`. **Prefer an exact `X.Y.Z`** over `stable`.
2. Read that version's CHANGELOG entries and every entry between the current and the target version:
   - "Upgrade notes" that add `.env` variables: add them, without printing secret values;
   - notes that require manual steps: **ask the human** first.
3. `./webarmctl doctor --json` must be `ok:true`. Don't upgrade a sick install; fix it first (skill `webarm-selfhost-troubleshoot`).
4. License: `./webarmctl license status --json`. The target's build date (CHANGELOG) must be on or before `updates_until`. `upgrade` enforces this too.

## Upgrade
```bash
./webarmctl upgrade X.Y.Z --json
```
- `ok:true`, `to == "X.Y.Z"` → continue.
- `rolled_back_to` present: the new version did not become healthy or the license failed on it, so the tool rolled back automatically. Collect `./webarmctl logs app | tail -120` and report. Don't retry blindly.
- A check `license covers X.Y.Z` with `ok:false` (WA-LIC-002): **stop**. The human decides whether to renew Updates (and then refresh the key: `/setup` → check in, or `./webarmctl license set`) or to choose an older tag.

What `upgrade` does by itself, so don't repeat it: pull, license check, `backup`, drain (waits for running pipelines/workflows, up to `DRAIN_TIMEOUT`), recreate, health check, auto-rollback.

## After
1. `SMOKE_ADMIN_EMAIL=… SMOKE_ADMIN_PASSWORD=… ./webarmctl smoke --json` → `ok:true`.
2. `./webarmctl version --json` → report the old and new versions.
3. If smoke fails: **ask the human**, then `./webarmctl rollback --yes --json`.
   - The database is not rolled back; schema changes are additive.
   - Only if the release notes say the new version changed data incompatibly: offer `./webarmctl restore <pre-upgrade backup>`. It is destructive and needs explicit approval.

## Updating the bundle itself (webarmctl, docs, compose)
If the directory is a git clone (`git rev-parse --is-inside-work-tree` succeeds):
1. `git status --porcelain`: tracked files must be unmodified. If they are modified, ask the human before stashing.
2. `git pull`, or `git fetch --tags && git checkout vX.Y.Z` to match the image version.

`.env`, `license/`, `backups/` and `custom/` are git-ignored.

Otherwise (tarball install):
```bash
tar -xzf webarm-selfhost-bundle-<new>.tgz -C /tmp
rsync -a --exclude .env --exclude license --exclude backups --exclude custom --exclude .webarm-state /tmp/webarm-selfhost/ /opt/webarm/
```
Never overwrite `.env`, `license/`, `backups/`, `custom/` or `.webarm-state`.

## Air-gapped
Run `docker load` on the new image first (docs/13). `upgrade` uses the local image when the registry is unreachable.
