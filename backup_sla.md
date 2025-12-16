# Backup SLA

## Coverage

We back up services that satisfy at least one of these criteria:
 - are primary source of truth for particular data
 - contain customer and/or client data
 - are not feasible (or very costly) to restore by other means

Services that are backed up:
 - MySQL (agama database - customer data)
 - Prometheus (monitoring metrics history)
 - Loki (application logs history)
 - Ansible Git repo (infrastructure configuration)

Services that are NOT backed up (can be restored from Ansible):
 - Nginx
 - AGAMA application
 - Grafana (dashboards provisioned via Ansible)
 - Bind DNS

## Schedule

MySQL backups are created every day at 21:45 UTC (dump) and 22:50 UTC (upload); it takes up to 5 minutes to create and store the backup.

Prometheus backups are created every day at 21:45 UTC (snapshot) and 22:50 UTC (upload); it takes up to 10 minutes to create and store the backup.

Loki backups are created every day at 21:45 UTC (file sync) and 22:50 UTC (upload); it takes up to 5 minutes to create and store the backup.

Full backups are performed on Sundays at 22:50 UTC.
Incremental backups are performed Monday–Saturday at 22:50 UTC.

Backup RPO (recovery point objective) is:
 - 24 hours for MySQL
 - 24 hours for Prometheus
 - 24 hours for Loki
 - 1 hour for Ansible Git repo (mirrored by instructor infrastructure)

## Storage

MySQL, Prometheus, and Loki backups are uploaded to the backup server (backup.acme.ttu) via rsync over SSH.

Ansible Git repo is mirrored to GitHub (github.com/orinocoz/ica0002).

Backup data from both servers will be synchronized to encrypted AWS S3 bucket in future (work in progress).

## Storage Limitations

Backup server quota: **200 MB** per user.

Current approximate usage per service:
 - MySQL: ~1 MB (small, text-based SQL dumps)
 - Prometheus: ~100-150 MB (largest - snapshot directory names change daily, causing large incremental backups)
 - Loki: ~5-20 MB (WAL and TSDB files)

To stay within quota limits:
 - Old backup chains are automatically cleaned after each Sunday full backup (keep only 1 full chain + its incrementals)
 - Prometheus consumes the most space because snapshot timestamps create new directory names daily, making each "incremental" backup nearly full-sized

If quota is exceeded, backups will fail silently (0-volume backup sets) and monitoring metrics will show stale timestamps.

## Retention

MySQL backups are stored for 1 week; 7 versions (recovery points) are available to restore.

Prometheus backups are stored for 1 week; 7 versions are available to restore.

Loki backups are stored for 1 week; 7 versions are available to restore.


## Usability checks

MySQL backups are verified every month by restoring to a test database and running integrity checks.

Prometheus backups are verified every month by restoring snapshots and querying historical data.

Loki backups are verified every month by restoring logs and verifying log queries return expected results.


## Restore process

Service is recovered from the backup in case of an incident, and when service cannot be restored in any other way.

RTO (recovery time objective) is:
 - 1 hour for MySQL (restore from dump + replication sync)
 - 2 hours for Prometheus (restore snapshots + restart service)
 - 1 hour for Loki (restore WAL and TSDB files + restart service)

Detailed backup restore procedure is documented in the [backup_restore.md](./backup_restore.md).
