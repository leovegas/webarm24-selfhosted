# Upgrade checklist

- [ ] Read `docs/CHANGELOG.md` of the target version; apply any `.env` additions it lists
- [ ] Target image is covered by the license (build date ≤ `updates_until`) — `webarmctl upgrade` checks it too
- [ ] `./webarmctl doctor` all OK before starting
- [ ] Announce a short maintenance window (20–60 s restart; running work is drained first)
- [ ] `./webarmctl upgrade X.Y.Z` → `upgrade — now on X.Y.Z`
- [ ] `./webarmctl smoke` passes
- [ ] Spot-check one real pipeline and one workflow run
- [ ] If anything is wrong: `./webarmctl rollback` (DB stays; restore the pre-upgrade backup only if the notes say so)
- [ ] `docker image prune -f` after a few days (keep the previous tag until then)
