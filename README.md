# firefly-update

Update a [Firefly III](https://www.firefly-iii.org/) instance on Debian.

Installs the latest (or a pinned) release via Composer, carries over your
`.env`, uploads, exports, and SQLite database, runs the migrations and
upgrade steps, then swaps the new version in and restarts Apache. A
timestamped database backup is taken before anything is touched, and
`--restore` rolls back to the previous version.

## Usage

Run as root (it chowns files and restarts Apache):

```bash
sudo ./firefly-update              # update to latest release
sudo ./firefly-update -v 6.1.0     # update to a specific tag
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
- `set -euo pipefail`: any failed step aborts before the live instance is
  swapped, so a broken build never replaces a working one.
- The previous version is kept at `$INSTANCE-old` until the next run.

## License

GPLv2-only. See [LICENSE](LICENSE).

Copyright (C) 2026 Danilo M. <danix@danix.xyz>
