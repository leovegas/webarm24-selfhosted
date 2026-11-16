# 08 — Upgrades

## Tags

| Tag | Meaning |
|---|---|
| `X.Y.Z` (e.g. `1.4.2`) | an exact release. Pin this in production |
| `X.Y` | latest patch of a minor line |
| `stable` | latest release |

Release notes are in `docs/CHANGELOG.md` (each new bundle) and on https://webarm24.online/self-hosted/docs/CHANGELOG. Read the notes before upgrading across a minor version. They list any new required settings.

## Upgrade

```bash
./webarmctl upgrade 1.5.0        # or: ./webarmctl upgrade   (= stable)
```

What it does, in order:
1. **Pull** the image.
2. **License check.** It compares the image's build date with your key's `updates_until`. If the image isn't covered, it stops before touching anything (`WA-LIC-002`).
3. **Backup:** `./webarmctl backup`.
4. **Drain.** The app stops accepting new work. Running pipelines and workflow nodes finish, with a timeout of `DRAIN_TIMEOUT`, default 900 s. Scheduled triggers that fire during the drain run after the switch.
5. **Switch.** It sets `WEBARM_TAG` in `.env`, recreates the app container, and waits for it to be healthy. Database migrations are additive and run automatically on start.
6. **Verify.** The license must be valid on the new version. If it isn't, or if the app doesn't become healthy, it **rolls back automatically** to the previous tag.

Downtime is the container restart, usually 20–60 s. In-flight runs are not interrupted, because the drain waits for them.

## Roll back

```bash
./webarmctl rollback
```
This switches back to the previously recorded tag. The **database is not rolled back**. Schema changes are additive, so older versions run on a newer schema. If a release note says otherwise, restore the pre-upgrade backup: `./webarmctl restore backups/webarm-<timestamp>.tgz`.

## Updates subscription expired?

`./webarmctl upgrade` refuses images built after your window. To find the newest tag you may run, compare the build dates in the CHANGELOG with `updates_until` from `./webarmctl license status`. After renewing, wait for the weekly check-in, or click **Check for a renewed key now** on `/setup`. Then upgrade.

## Pinning and staged rollouts

- Production: pin `X.Y.Z` and upgrade deliberately.
- Run a staging install first. Each install counts against your license terms, so ask sales about a staging key.
- `./webarmctl upgrade <tag> --json` exits non-zero on failure and prints `{"ok":false,...,"rolled_back_to":"…"}`. This is suitable for automation.

## Bundle updates

New bundles may update `webarmctl`, the compose file and the docs.

If you installed with `git clone`:
```bash
cd /opt/webarm && git pull            # on the `stable` branch; or: git fetch --tags && git checkout vX.Y.Z
```
`.env`, `license/`, `backups/`, `custom/` and `.webarm-state` are in `.gitignore`, so `git pull` never touches them. If you edited a tracked file such as `docker-compose.yml`, move the change into an override file first (`docker-compose.override.yml`) or `git stash` it.

If you installed from the tarball:
```bash
tar -xzf webarm-selfhost-bundle-<new>.tgz -C /tmp
rsync -a --exclude .env --exclude license --exclude backups --exclude custom --exclude .webarm-state /tmp/webarm-selfhost/ /opt/webarm/
```
Your `.env`, license, backups and custom pages are never overwritten. If there are new settings, the release note tells you which lines to add to `.env`.
