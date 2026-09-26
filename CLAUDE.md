# CLAUDE.md

Single bash script, `firefly-update`, that updates a Firefly III instance on
Debian. See README.md for usage; this file is the non-obvious context.

## Hard-won gotchas (do not relearn these)

- **Install from the GitHub release zip, never `composer create-project`.** The
  composer/Packagist dist ships source only, no compiled frontend bundles
  (`public/v1/js/app.js`, `app_vue.js`, ...). A composer install looks fine but
  the UI is broken: no graphs, `404 /v1/js/app.js`, and the v2 error/login
  pages throw `Vite manifest not found (public/build/manifest.json)` as a
  secondary symptom. The zip is prebuilt and is the supported artifact.

- **v6.7 dropped the legacy `public/v1/js/app.js` bundle** (new layout; only
  `public/v1/js/ff/*` and `.htaccess` remain). The extract check keys on
  `public/build/manifest.json` instead, present in 6.6.x and 6.7.x zips and
  absent from the composer dist. Don't go back to checking `app.js`.

- **Release tag carries a leading `v`** (`v6.6.5`). Asset name is
  `FireflyIII-<tag>.zip` with the `v` included. `--version` is normalized so
  both `6.6.5` and `v6.6.5` work; don't reintroduce a double-`v` bug.

- **Passport OAuth migration collision.** Laravel Passport renames its
  migration files between versions, so a carried-over DB has the `oauth_*`
  tables while the new filenames look pending; a naive `migrate` dies with
  `table "oauth_auth_codes" already exists` (then `oauth_device_codes`, ...).
  The script reconciles the `migrations` ledger before migrating (records
  already-present tables, lets missing ones create). Do NOT "fix" a collision
  by dropping tables, it is whack-a-mole and can lose
  `oauth_personal_access_clients`.

- **The Vite/`public/build` error is a red herring for graphs.** It only fires
  on the v2 error and login pages. Graphs breaking = missing `public/v1/js`
  bundles (see the zip point above), a different cause.

- **Upgrade artisan sequence follows the official self-managed docs**:
  `migrate --seed`, `cache:clear`, `view:clear`, `firefly-iii:upgrade-database`,
  `firefly-iii:laravel-passport-keys`. Use `firefly-iii:laravel-passport-keys`,
  NOT `passport:install`. We add `--force` (non-interactive) and the oauth
  reconcile before migrate. Docs also confirm composer create-project is no
  longer recommended.

- **Release `.sha256` format** is `<hash> *FireflyIII-<tag>.zip` (BSD-style,
  space then `*`); `cut -d' ' -f1` gets the hash. The script verifies it when
  `sha256sum` is available, and validates the extracted tree before swapping.

- **Extract the FULL zip, do NOT `unzip -x 'storage/*'`.** The zip's storage
  skeleton provides `storage/framework/{cache,views,sessions}`, which Laravel
  needs to boot. Excluding it and recreating only the data dirs makes artisan
  die with the misleading `Target class [auth] does not exist` (the real boot
  failure is masked by the exception reporter, which itself calls `auth()`).
  Extract everything, then overlay live user data (uploads/exports/DB) on top.

## Conventions

- Config via env vars (`WORKDIR`, `INSTANCE`, `BACKUPDIR`), matching the
  existing style; no new CLI flags unless asked.
- Keep it a single file. `set -euo pipefail` + `ERR` trap are load-bearing:
  the script must abort before the live swap on any failure and never leave a
  half-update looking successful.
- Any change to the install/migrate flow: test the download+extract and the
  migrate path before claiming it works. The script touches a live personal
  finance instance; a bad run is not cheap.

## Testing without a target machine

There is no Firefly instance in this repo. Test URL construction and
download/extract against the real GitHub releases (the asset URLs are public);
the artisan/migrate steps can only be exercised on the actual host.
