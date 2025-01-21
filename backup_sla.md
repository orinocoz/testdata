# Backup SLA

## Coverage

We back up services that satisfy at least one of these criteria:
 - are primary source of truth for particular data
 - contain customer and/or client data
 - are not feasible (or very costly) to restore by other means

Services that are backed up:
 - MySQL 
 - InfluxDB
 - User git repo


## Schedule

MySQL backups are created every day; it takes up to 2 minutes to create and store the backup.

InfluxDB backups are created every day; it takes up to 2 minutes to create and store the backup.

Git repo backups are created every half a hour; it takes up to minute to create and store the backup.

All backups are started automatically by cron daemon.

Backup RPO (recovery point objective) is:
 - 1 day for MySQL
 - 1 day_ for InfluxDB
 - 30 minutes for Git repo


## Storage

MySQL and InfluxDB backups are uploaded to the backup server.

Git repo is mirrored to the internal Git server.

Backup data from both servers will be synchronized to encrypted AWS S3 bucket in future (work in progress).


## Retention

MySQL backups are stored for 30 days_; 5 full backups and daily incremential versions (recovery points) are available to restore.

InfluxDB backups are stored for 30 days; 5 full backups and daily incremential versions are available to restore.

Git repo backups are stored for 180 days; 1 version is available to restore.


## Usability checks

MySQL backups are verified every month by doing recovery procedure.

InfluxDB backups are verified every month by doing recovery procedure.

Git repo backups are verified every 30 minutes by automatic process.


## Restore process

Service is recovered from the backup in case of an incident, and when service cannot be restored in any other way.

RTO (recovery time objective) is:
 - 1 hour for MySQL
 - 1 hour for InfluxDB
 - 15 minutes for Git repo

Detailed backup restore procedure is documented in the [backup_restore.md](./backup_restore.md).

