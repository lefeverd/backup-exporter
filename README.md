# backup-exporter

A Prometheus exporter for your [Borg](https://borgbackup.readthedocs.io/) and [restic](https://restic.net/)
backups. Point it at your repositories and it tells you, through Prometheus and Grafana:

- when each repository was last backed up, so you can alert on stale backups
- how big each backup is, how many files it holds and how long it took
- how much new data each backup added, and (for Borg) how much space the whole repository uses on disk
- whether the exporter itself is managing to read your repositories

It works by periodically running `borg info` and `restic snapshots`/`restic stats` against the repositories
listed in its config file, and serving the results on a `/metrics` endpoint.

![Grafana Dashboard](./dashboards/grafana-exporter-dashboard.png)

- [Quick start](#quick-start)
- [Configuration](#configuration)
- [Running it as a service](#running-it-as-a-service)
- [Prometheus](#prometheus)
- [Alerting](#alerting)
- [Grafana dashboards](#grafana-dashboards)
- [Metrics](#metrics)
- [How collection works](#how-collection-works)
- [Migrating from borg-exporter](#migrating-from-borg-exporter)
- [Development and releases](#development-and-releases)

## Quick start

1. Download the latest `backup-exporter-<version>-linux-amd64.tar.gz` from the
   [releases page](https://github.com/lefeverd/backup-exporter/releases), and extract the binary to
   `/usr/local/bin/backup-exporter`.

2. Create a config file listing your repositories, e.g. `/etc/backup-exporter/config.yaml`:

   ```yaml
   repositories:
     - name: home
       type: borg
       path: ssh://my-repository/backups/my-machine

     - name: photos
       type: restic
       repository: sftp:user@host:/path/to/repo
       password_command: "pass show restic/photos"
   ```

3. Run it as the user that owns your backups (see [User and permissions](#user-and-permissions)):

   ```
   backup-exporter -config /etc/backup-exporter/config.yaml
   ```

4. After the first collection (which can take a few minutes), check the metrics:

   ```
   curl 127.0.0.1:9099/metrics
   ```

Then [run it as a service](#running-it-as-a-service), [add it to Prometheus](#prometheus) and
[import the dashboards](#grafana-dashboards).

## Configuration

### Repositories

The repositories to collect metrics for are listed in a YAML file, passed with `-config` or `CONFIG_FILE`.
You can mix `borg` and `restic` entries:

```yaml
repositories:
  - name: home            # used as the "repository" label on every metric
    type: borg
    path: ssh://my-repository/backups/my-machine
    opts: "--lock-wait 5" # optional, extra borg options

  - name: photos
    type: restic
    repository: sftp:user@host:/path/to/repo
    # exactly one of the following three is required:
    password_command: "pass show restic/photos"  # recommended
    # password_file: /run/secrets/restic-photos-password
    # password: "not recommended, ends up readable in the config file"
```

`password` mirrors restic's own `RESTIC_PASSWORD` convention and is supported for compatibility with existing
restic setups, but is discouraged: the exporter logs a warning on startup for any restic repository using it.

### Exporter settings

Everything else is set with flags or environment variables. Only the config file is required.

| Environment variable       | Flag                        | Description                                                                             | Default    |
|----------------------------|-----------------------------|-----------------------------------------------------------------------------------------|------------|
| `CONFIG_FILE`              | `-config`                   | Path to the YAML repositories config file (**required**)                                |            |
| `LISTEN_ADDRESS`           | `-listen-address`           | Address the server listens on                                                           | `:9099`    |
| `METRICS_PATH`             | `-metrics-path`             | Path the metrics are served on                                                          | `/metrics` |
| `BORG_REFRESH_INTERVAL`    | `-borg-refresh-interval`    | How often borg metrics are refreshed                                                    | `4h`       |
| `RESTIC_REFRESH_INTERVAL`  | `-restic-refresh-interval`  | How often restic metrics are refreshed                                                  | `4h`       |
| `SCHEDULER_CHECK_INTERVAL` | `-scheduler-check-interval` | How often each scheduler checks whether a refresh is due                                | `20s`      |
| `COMMAND_TIMEOUT`          | `-command-timeout`          | Timeout for a full collection cycle (all repositories of one tool)                      | `120s`     |
| `BORG_PATH`                | `-borg-path`                | Path to the borg binary                                                                 | `borg`     |
| `RESTIC_PATH`              | `-restic-path`              | Path to the restic binary                                                               | `restic`   |
| `LOG_LEVEL`                | `-log-level`                | Logging level (`debug`, `info`, `warn`, `error`)                                        | `info`     |
| `ARCHIVE_HISTORY_LIMIT`    | `-archive-history-limit`    | Number of most recent archives/snapshots to expose per-archive metrics for (both tools) | `10`       |

The `4h` refresh intervals suit daily backups, whose metrics don't change often. Lower them if you back up more
frequently. See [How collection works](#how-collection-works) for details.

### User and permissions

Run the exporter as a user that can read the borg/restic repositories, typically the user that runs the backups.
For restic repositories using `password_command`, the command must run successfully and non-interactively as that
same user.

## Running it as a service

With systemd, create `/etc/systemd/system/backup-exporter.service`:

```
[Unit]
Description=Backup Prometheus Exporter
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/backup-exporter
Environment="CONFIG_FILE=/etc/backup-exporter/config.yaml"
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

If the backups belong to a non-root user, add `User=<that user>` under `[Service]`.

Then reload systemd, and enable and start the service:

```
sudo systemctl daemon-reload
sudo systemctl enable --now backup-exporter.service
```

Follow the logs with:

```
sudo journalctl -fu backup-exporter
```

## Prometheus

Add a scrape job for the exporter:

```yaml
- job_name: 'backup-exporter'
  scrape_interval: 3m
  scrape_timeout: 30s
  static_configs:
    - targets:
      - '<hostname>:9099'
```

Scraping is cheap: the exporter serves cached metrics, it doesn't run borg/restic on each scrape. Keep
`scrape_interval` under `5m` though, otherwise Prometheus considers the series stale between scrapes.

For the [history dashboards](#grafana-dashboards), the longer your Prometheus retention, the further back you can
look.

## Alerting

Some example Prometheus alerting rules to start from:

```yaml
groups:
  - name: backup-exporter
    rules:
      - alert: BackupTooOld
        expr: (time() - borg_last_backup_timestamp) > 2 * 24 * 3600
           or (time() - restic_last_backup_timestamp) > 2 * 24 * 3600
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "Backup too old: {{ $labels.repository }}"
          description: "Last backup of {{ $labels.repository }} was {{ $value | humanizeDuration }} ago."

      - alert: BackupCollectFailing
        expr: exporter_last_collect_error == 1
        for: 6h
        labels:
          severity: warning
        annotations:
          summary: "backup-exporter can't read {{ $labels.repository }} ({{ $labels.tool }})"
          description: "Collection has been failing for 6h on {{ $labels.instance }}. Check the exporter logs."

      - alert: BackupSizeDropped
        expr: borg_last_backup_original_size_bytes < 0.7 * (borg_last_backup_original_size_bytes offset 1d)
           or restic_last_backup_original_size_bytes < 0.7 * (restic_last_backup_original_size_bytes offset 1d)
        labels:
          severity: info
        annotations:
          summary: "Backup of {{ $labels.repository }} shrank by more than 30% in a day"
          description: "Could be a new exclude or deleted data, or files silently missing from the backup."
```

- **`BackupTooOld`** is the one that matters most. Set the threshold from your backup schedule: 2 days suits daily
  backups, but will fire constantly for twice-weekly ones. If your repositories run on different schedules, add a
  label per schedule in your scrape config (e.g. with `metric_relabel_configs` matching on `repository`) and write
  one rule per label.
- **`BackupCollectFailing`** uses a long `for:` so it doesn't flap on machines that sleep or leave the network, such
  as laptops. The exporter retries failed collections on its own (see [How collection works](#how-collection-works)).
- **`BackupSizeDropped`** catches the case where a backup still runs but stops including some data. It's often
  intentional (a new exclude, deleted files), hence the `info` severity. It needs a sample from a day earlier, so it
  won't fire for a drop that happens while the exporter isn't being scraped.

An "exporter down" alert (`up{job="backup-exporter"} == 0`) is left out on purpose: it's noisy for machines that
aren't always on, and `BackupTooOld` already catches what matters, backups that stopped.

## Grafana dashboards

Import the dashboards from [the dashboards directory](./dashboards) into Grafana:

- **`backup-exporter.json`**: the overview. It has a Borg section and a Restic section, each with a summary table
  of that tool's repositories, then per-repository details: an archive history table, size/files/duration bar
  charts for the last `N` archives/snapshots (see `ARCHIVE_HISTORY_LIMIT`), and the exporter's collection health.
- **`backup-exporter-borg-history.json`** and **`backup-exporter-restic-history.json`**: the long-term history of a
  single repository: backup size, files, duration, unique data added, backups per day, repository size on disk
  (Borg only) and collection health. They cover your whole Prometheus retention instead of only the last `N`
  archives, with gaps wherever the exporter wasn't scraped.

Clicking a repository in one of the overview's summary tables opens its history dashboard. Import all three
dashboards for those links to work, and keep their `uid`s (`be2qvzcxrzh1ca`, `backup-exporter-borg-history` and
`backup-exporter-restic-history`), since the links use them.

Borg and restic panels show different fields where the tools' metrics don't map onto each other. For example,
Borg's repository-level totals have no restic equivalent (see [Metrics](#metrics)).

> The overview was previously named `borg-last-backups.json`. Grafana UI imports (keyed by `uid`) are unaffected,
> but if you provision it by file path or URL (e.g. GitOps-style), update the path.

## Metrics

Every metric is labeled by `repository`, the `name` from the config file.

There are two kinds of per-backup metrics:

- **`*_last_backup_*`**: the latest backup only. Scraped over time, they build up a long-term history in
  Prometheus, which is what the history dashboards use. That history starts when the exporter starts tracking a
  repository.
- **`*_archive_*`**: one series per archive/snapshot, for the last `N` of them (`ARCHIVE_HISTORY_LIMIT`, default
  10), labeled additionally by `archive` (Borg archive name) or `snapshot` (restic short snapshot ID). These include
  backups that predate the exporter, but only the most recent `N`.

All timestamps are Unix epoch seconds.

### Borg

| Name                                       | Description                                    | Type  |
|--------------------------------------------|------------------------------------------------|-------|
| `borg_last_backup_duration_seconds`        | Duration of the last backup in seconds         | Gauge |
| `borg_last_backup_compressed_size_bytes`   | Compressed size of the last backup in bytes    | Gauge |
| `borg_last_backup_deduplicated_size_bytes` | Deduplicated size of the last backup in bytes* | Gauge |
| `borg_last_backup_files`                   | Number of files in the last backup             | Gauge |
| `borg_last_backup_original_size_bytes`     | Original size of the last backup in bytes      | Gauge |
| `borg_last_backup_timestamp`               | Timestamp of the last backup                   | Gauge |
| `borg_archive_duration_seconds`            | Duration of the archive's backup in seconds    | Gauge |
| `borg_archive_compressed_size_bytes`       | Compressed size of the archive in bytes        | Gauge |
| `borg_archive_deduplicated_size_bytes`     | Deduplicated size of the archive in bytes*     | Gauge |
| `borg_archive_files`                       | Number of files in the archive                 | Gauge |
| `borg_archive_original_size_bytes`         | Original size of the archive in bytes          | Gauge |
| `borg_archive_timestamp`                   | Timestamp of the archive's backup              | Gauge |
| `borg_total_chunks`                        | Repository total chunks                        | Gauge |
| `borg_total_compressed_size_bytes`         | Repository total compressed size               | Gauge |
| `borg_total_size_bytes`                    | Repository total size                          | Gauge |
| `borg_total_unique_chunks`                 | Repository total unique chunks                 | Gauge |
| `borg_deduplicated_compressed_size_bytes`  | Repository deduplicated compressed size        | Gauge |
| `borg_deduplicated_size_bytes`             | Repository deduplicated size                   | Gauge |
| `borg_last_archive_info`                   | Information about the last backup archive      | Gauge |
| `borg_repository_info`                     | Information about the backup repository        | Gauge |

\* This is *not* "the size of this backup after deduplication". Borg computes it as the data exclusively owned by
this archive relative to the repository's current state, i.e. what would be freed if you deleted just this archive.
It is order-dependent: it can read `0` for a large, valid backup whose content duplicates another archive, and can
change for an older archive once newer ones are created. See the
[borg documentation](https://borgbackup.readthedocs.io/en/stable/usage/info.html) for details.

### Restic

| Name                                         | Description                                                   | Type  |
|----------------------------------------------|---------------------------------------------------------------|-------|
| `restic_last_backup_duration_seconds`        | Duration of the last backup in seconds*                       | Gauge |
| `restic_last_backup_original_size_bytes`     | Original (restore) size of the last backup in bytes           | Gauge |
| `restic_last_backup_files`                   | Number of files in the last backup                            | Gauge |
| `restic_last_backup_timestamp`               | Timestamp of the last backup                                  | Gauge |
| `restic_last_backup_data_added_bytes`        | Uncompressed data added to the repository by the last backup* | Gauge |
| `restic_last_backup_data_added_packed_bytes` | Compressed data added to the repository by the last backup*   | Gauge |
| `restic_archive_duration_seconds`            | Duration of the snapshot's backup in seconds*                 | Gauge |
| `restic_archive_original_size_bytes`         | Original (restore) size of the snapshot in bytes              | Gauge |
| `restic_archive_files`                       | Number of files in the snapshot                               | Gauge |
| `restic_archive_timestamp`                   | Timestamp of the snapshot's backup                            | Gauge |
| `restic_archive_data_added_bytes`            | Uncompressed data added to the repository by this snapshot*   | Gauge |
| `restic_archive_data_added_packed_bytes`     | Compressed data added to the repository by this snapshot*     | Gauge |
| `restic_last_snapshot_info`                  | Information about the last backup snapshot                    | Gauge |

\* restic only records this data (its snapshot "summary" block) from a version recent enough to capture it.
Snapshots taken by an older restic, including ones that already existed before you upgraded, won't have these
series. The exporter leaves them unset rather than reporting a fake `0`.

There's no restic equivalent of Borg's chunk-level dedup stats (`borg_total_chunks` and friends) or repository
info: restic's CLI doesn't expose them per repository, so they're left out rather than approximated.

### Exporter

These describe the exporter's own operation and are shared across tools. They're labeled by `tool` (`borg` or
`restic`) and `repository`, except `exporter_system_info`, which is labeled by `tool`, `hostname` and `version`.

| Name                                     | Description                                           | Type    |
|------------------------------------------|-------------------------------------------------------|---------|
| `exporter_collect_errors`                | Number of errors encountered while collecting metrics | Counter |
| `exporter_last_collect_error`            | 1 if the last collection failed, 0 if successful      | Gauge   |
| `exporter_last_collect_duration_seconds` | Duration of the last metrics collection               | Gauge   |
| `exporter_last_collect_timestamp`        | Timestamp of the last metrics collection              | Gauge   |
| `exporter_system_info`                   | Information about a configured backup tool            | Gauge   |

## How collection works

Collecting metrics can be slow, especially with many repositories or a slow backend, so it's decoupled from
Prometheus scrapes. The exporter refreshes its metrics in the background on its own interval and serves the latest
cached values on each scrape.

- **Independent schedulers.** Borg and restic each have their own scheduler and refresh interval, so a slow (e.g.
  WAN) restic collection doesn't delay Borg's, and vice versa. If the config only has one type of repository, only
  that tool's scheduler runs.
- **Sleep-friendly.** Rather than a fixed timer, each scheduler checks every `SCHEDULER_CHECK_INTERVAL` (default
  `20s`) whether a refresh is due, so it keeps to its interval on machines that sleep and wake up.
- **Partial failures.** If one repository can't be read, the error is logged and the other repositories' metrics are
  still exposed. If the refresh interval is longer than 5 minutes, a failed collection is retried up to 5 times, one
  minute apart, instead of waiting hours for the next refresh after a transient error.
- **Restic size caching.** `restic_*_original_size_bytes` and `restic_*_files` come from
  `restic stats <snapshot> --mode restore-size`, a separate and comparatively slow command per snapshot (especially
  over a WAN/SFTP backend). The exporter caches these per snapshot and only computes them for snapshots it hasn't
  seen before, plus the latest snapshot every cycle, since older snapshots' sizes don't change. Snapshots removed by
  `forget`/`prune` are dropped from the cache too.

## Migrating from borg-exporter

This project was renamed from `borg-exporter` to `backup-exporter`, and its configuration moved from flags/env only
to the YAML file above, as part of adding restic support. It's a one-time breaking change with no compatibility
shim:

- Replace `BORG_REPOSITORIES`/`-borg-repositories` (comma-separated paths) and `BORG_OPTS`/`-borg-opts` with a
  `repositories:` YAML file, one `type: borg` entry per repository, with `opts:` set per entry if needed.
- `METRICS_REFRESH_INTERVAL`/`-metrics-refresh-interval` is now `BORG_REFRESH_INTERVAL`/`-borg-refresh-interval`.
- The binary and systemd unit/service name changed from `borg-exporter` to `backup-exporter`.
- `borg_collect_errors`, `borg_last_collect_error`, `borg_last_collect_duration_seconds`,
  `borg_last_collect_timestamp` and `borg_system_info` were replaced by the shared `exporter_*` metrics (now
  labeled by `tool` too). Update any alerts/dashboards referencing the old names.
- The overview dashboard file was renamed from `borg-last-backups.json` to `backup-exporter.json`.

## Development and releases

```
make test     # run the tests
make build    # build build/backup-exporter (linux/amd64)
```

The project follows semantic versioning. Releases are cut by pushing a tag, which triggers the release pipeline:

```
make release                              # new minor version
./release.sh <major|minor|patch>          # pick the version part
./release.sh <major|minor|patch> --dry-run # show the tag that would be created
```
