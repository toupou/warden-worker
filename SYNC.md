# Upstream Sync & Deploy Runbook

Keeping this fork (`toupou/warden-worker`) current with upstream
(`qaz741wsd856/warden-worker`) and deploying safely to all workers.

## Remotes
- `origin`   → git@github.com:toupou/warden-worker.git  (this fork)
- `upstream` → https://github.com/qaz741wsd856/warden-worker.git

If `upstream` is missing:
`git remote add upstream https://github.com/qaz741wsd856/warden-worker.git`

## 1. Notice new releases
- On the upstream repo → **Watch → Custom → Releases** (email on each new tag).
- Or check manually:
  ```
  git fetch upstream
  git log --oneline main..upstream/main
  ```

## 2. Review what's coming
Watch for new migrations and tooling/version bumps:
```
git diff --stat main..upstream/main -- migrations/ wrangler.toml .github/workflows/
```
New files under `migrations/` = schema changes that apply on the next deploy.

## 3. Merge
```
git switch -c backup-main-pre-merge main   # safety snapshot
git switch main
git merge upstream/main
```
Conflicts are usually only in the workflow YAMLs
(`push-cloudflare.yaml`, `backup-d1.yaml`).
**Rule:** keep our per-worker / D1 logic, take upstream's improvements and
new tool versions.

After resolving, confirm no markers remain, then commit:
```
grep -rn -E '^(<<<<<<<|=======|>>>>>>>)' .github/workflows/
git add -A && git commit
```

## 4. Version sanity check
The worker-build version now lives **only** in the workflow env
(`WORKER_BUILD_VERSION` in `push-cloudflare.yaml`). The per-worker
`wrangler-*.toml` build command is just `worker-build --release --locked`
and uses whatever the workflow installs — so after a merge just confirm the
env block looks sane:
```
grep -n 'WORKER_BUILD_VERSION\|WRANGLER_VERSION\|BW_WEB_VERSION' .github/workflows/push-cloudflare.yaml
```
Do **not** re-add a hardcoded worker-build version to the per-worker tomls.

## 5. Back up D1 (always, before deploying)
```
mkdir -p ~/warden-d1-backups
TS=$(date +%Y%m%d-%H%M%S)
for db in vault vault-dgcs vault-homx vault-makani vault-ordi vault-private; do
  npx wrangler d1 export "$db" --remote --output="$HOME/warden-d1-backups/${db}-${TS}.sql"
done
```

## 6. Push + staged deploy
```
git push origin main
```
Deploy via **GitHub Actions → "Build" workflow → Run workflow** → branch `main`,
pick `target`. Order, safest first:
1. `ordi`    — empty, verifies build + migrations
2. `private` — personal: log in + sync, confirm an entry survives
3. `homx`, `makani`
4. `dgcs`    — most users, deploy last

(Or `target: all` once a populated vault like `private` is verified.)

After each deploy: open the vault, log in, sync, confirm entries are intact.

## Rollback
- **D1 Time Travel** (30-day window): inspect with
  `npx wrangler d1 time-travel info <db>`, then
  `npx wrangler d1 time-travel restore <db> --timestamp <ISO>`
  (see `npx wrangler d1 time-travel --help`).
- Or re-import a dump from `~/warden-d1-backups`.
- Or roll back to the previous Worker version in the Cloudflare dashboard.

## Gotchas (learned the hard way)
- **Never** `UPDATE users SET email=...` directly — the email is the KDF salt;
  changing it bricks login + decryption. Use the app's change-email flow.
- Deleting a user in SQL must also clear `ciphers`, `folders`, `devices`,
  `sends`, `sends_pending`, `attachments`, `attachments_pending`, `twofactor`,
  `auth_requests` (children first, then `users`). Never touch `d1_migrations`.
- `wrangler d1 execute <name>` uses the **account** DB name (e.g. `vault-ordi`),
  not the binding (`vault1`). Always pass `--remote` to hit production.
