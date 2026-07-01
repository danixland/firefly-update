# firefly-update

Update a [Firefly III](https://www.firefly-iii.org/) instance on Debian.

Downloads the latest (or a pinned) release, carries over your `.env`, uploads,
exports, and SQLite database, runs the migrations and upgrade steps, then swaps
the new version in and restarts Apache. A timestamped database backup is taken
before anything is touched, and `--restore` rolls back to the previous version.

Installs from the **official GitHub release zip**, not `composer
create-project`. The composer/Packagist dist ships source only and lacks the
compiled frontend bundles, which leaves the UI broken (no graphs, 404s on
`/v1/js/app.js`). The release zip is prebuilt.

Requires `curl`, `unzip`, `php`, and `sqlite3` on the host. If `sha256sum` is
present, the download is verified against the release checksum, and the
extracted tree is validated (`artisan`, `bootstrap`, `vendor`, the compiled
frontend bundle) before the live swap.

The upgrade commands follow the official self-managed upgrade docs
(`migrate --seed`, `cache:clear`, `view:clear`, `firefly-iii:upgrade-database`,
`firefly-iii:laravel-passport-keys`), with `--force` for non-interactive use.

## Usage

Run as root (it chowns files and restarts Apache):

```bash
sudo ./firefly-update              # update to latest release
sudo ./firefly-update -v 6.6.5     # update to a specific tag (v-prefix optional)
sudo ./firefly-update --restore    # roll back to the previous version
sudo ./firefly-update --help
```

### Configuration

Override with environment variables:

| Variable    | Default                   | Meaning                                    |
|-------------|---------------------------|--------------------------------------------|
| `WORKDIR`   | `/var/www`                | Directory holding the instance             |
| `INSTANCE`  | `piggy`                   | Instance directory name under `$WORKDIR`   |
| `BACKUPDIR` | `$WORKDIR/firefly-backups`| Where timestamped DB backups are written   |

```bash
WORKDIR=/srv INSTANCE=ff sudo -E ./firefly-update
```

## Notes

- Assumes an SQLite database at `storage/database/database.sqlite`.
- `set -euo pipefail` plus an `ERR` trap: any failed step aborts before the
  live instance is swapped (a broken build never replaces a working one) and
  prints the failing line, so a partial run cannot masquerade as success.
- Before migrating, the script reconciles Laravel Passport's OAuth migrations:
  Passport renames its migration files between versions, so a carried-over DB
  has the `oauth_*` tables while the new filenames look pending, and a naive
  `migrate` would fail trying to recreate existing tables. The script records
  those already-present tables as migrated and lets genuinely-missing ones be
  created.
- The previous version is kept at `$INSTANCE-old` until the next run;
  `--restore` swaps it back and overlays the newest DB backup. A failed restore
  leaves the broken version at `$INSTANCE-broken`.

## License

GPLv2-only. See [LICENSE](LICENSE).

Copyright (C) 2026 Danilo M. <danix@danix.xyz>
