# Backup Restore Procedures

This document provides step-by-step instructions to restore services from backups.

**Backup server:** `backup.acme.ttu` (192.168.42.5)
**Backup user:** `orinocoz`
**Local backup user:** `backup`

---

## MySQL Restore

### Prerequisites
- SSH access to the database server (mysql_backup_host)
- MySQL running and accessible
- **Root access required** for importing database dump

### Steps

1. **List available backups:**
   ```bash
   sudo -u backup duplicity collection-status rsync://orinocoz@backup.acme.ttu/mysql
   ```

2. **Restore the latest backup to local directory:**
   ```bash
   sudo -u backup duplicity restore rsync://orinocoz@backup.acme.ttu/mysql /home/backup/restore/mysql
   ```

   To restore a specific point in time:
   ```bash
   sudo -u backup duplicity restore --time 2D rsync://orinocoz@backup.acme.ttu/mysql /home/backup/restore/mysql
   ```
   (Where `2D` means 2 days ago; use format like `2025-12-01` for specific date)

3. **Verify the dump file:**
   ```bash
   ls -la /home/backup/restore/mysql/
   head -50 /home/backup/restore/mysql/agama.sql
   ```

4. **Stop the application to prevent writes:**
   ```bash
   sudo docker stop agama-0 agama-1
   ```

5. **Import the database dump (as root):**
   ```bash
   sudo mysql agama < /home/backup/restore/mysql/agama.sql
   ```

   **Note:** MySQL import requires root privileges to write to the database.

6. **Verify data integrity:**
   ```bash
   sudo mysql -e "SELECT COUNT(*) FROM agama.item;"
   ```

7. **Restart the application:**
   ```bash
   sudo docker start agama-0 agama-1
   ```

8. **Verify application works:**
   ```bash
   curl -s http://localhost:8001/ | head -20
   ```

### Notes
- MySQL backups run on `mysql_backup_host` (the replica server)
- To restore production data: restore dump on the primary server, then replication will sync to replica
- RPO: 24 hours | RTO: 1 hour

---

## Prometheus Restore

### Prerequisites
- SSH access to the Prometheus server
- Prometheus service access
- **Root access required** for copying files and fixing permissions

### Steps

1. **List available backups:**
   ```bash
   sudo -u backup duplicity collection-status rsync://orinocoz@backup.acme.ttu/prometheus
   ```

2. **Stop Prometheus service:**
   ```bash
   sudo systemctl stop prometheus
   ```

3. **Restore snapshots to local directory:**
   ```bash
   sudo -u backup duplicity --no-restore-ownership restore rsync://orinocoz@backup.acme.ttu/prometheus /home/backup/restore/prometheus
   ```

   To restore a specific point in time:
   ```bash
   sudo -u backup duplicity --no-restore-ownership restore --time 3D rsync://orinocoz@backup.acme.ttu/prometheus /home/backup/restore/prometheus
   ```

   **Note:** The `--no-restore-ownership` flag is required because the backup user cannot set file ownership to `prometheus`.

4. **Backup current data (optional):**
   ```bash
   sudo mv /var/lib/prometheus/metrics2 /var/lib/prometheus/metrics2.old
   ```

5. **Copy restored snapshot to Prometheus data directory:**
   ```bash
   # Find the snapshot directory name
   ls /home/backup/restore/prometheus/

   # Copy the snapshot (replace SNAPSHOT_NAME with actual name)
   sudo cp -r /home/backup/restore/prometheus/SNAPSHOT_NAME /var/lib/prometheus/metrics2
   ```

6. **Fix permissions:**
   ```bash
   sudo chown -R prometheus:prometheus /var/lib/prometheus/metrics2
   ```

7. **Start Prometheus:**
   ```bash
   sudo systemctl start prometheus
   ```

8. **Verify metrics are available:**
   ```bash
   curl -s "http://localhost:9090/prometheus/api/v1/query?query=up" | head -20
   ```

9. **Verify in Grafana:**
   - Open Grafana dashboards
   - Check that historical metrics are visible

### Notes
- Prometheus snapshots are stored in `/var/lib/prometheus/metrics2/snapshots/`
- Snapshot files are owned by `prometheus:prometheus` - restored files must have same ownership
- The `backup` user needs read access to `/var/lib/prometheus/metrics2/snapshots/` for duplicity upload
- RPO: 24 hours | RTO: 2 hours

---

## Loki Restore

### Prerequisites
- SSH access to the Loki server
- Loki service access
- **Root access required** for copying files and fixing permissions (Loki files are owned by `loki` user)

### Steps

1. **List available backups:**
   ```bash
   sudo -u backup duplicity collection-status rsync://orinocoz@backup.acme.ttu/loki
   ```

2. **Stop Loki service:**
   ```bash
   sudo systemctl stop loki
   ```

3. **Restore Loki data to local directory:**
   ```bash
   sudo -u backup duplicity --no-restore-ownership restore rsync://orinocoz@backup.acme.ttu/loki /home/backup/restore/loki
   ```

   To restore a specific point in time:
   ```bash
   sudo -u backup duplicity --no-restore-ownership restore --time 1D rsync://orinocoz@backup.acme.ttu/loki /home/backup/restore/loki
   ```

   **Note:** The `--no-restore-ownership` flag is required because the backup user cannot set file ownership to `loki`.

4. **Backup current data (optional):**
   ```bash
   sudo mv /tmp/loki/tsdb-shipper-active /tmp/loki/tsdb-shipper-active.old
   sudo mv /tmp/loki/wal /tmp/loki/wal.old
   ```

5. **Copy restored files to Loki data directory:**
   ```bash
   sudo cp -r /home/backup/restore/loki/tsdb-shipper-active /tmp/loki/
   sudo cp -r /home/backup/restore/loki/wal /tmp/loki/
   ```

6. **Fix permissions:**
   ```bash
   sudo chown -R loki:loki /tmp/loki/tsdb-shipper-active
   sudo chown -R loki:loki /tmp/loki/wal
   ```

7. **Start Loki:**
   ```bash
   sudo systemctl start loki
   ```

8. **Verify Loki is running:**
   ```bash
   curl -s http://localhost:3100/ready
   ```

9. **Verify in Grafana:**
   - Open Grafana Explore
   - Query Loki datasource for historical logs
   - Example query: `{job="syslog"}`

### Notes
- Loki data directories: `/tmp/loki/tsdb-shipper-active` and `/tmp/loki/wal`
- **Backup process runs as `root`** to copy files owned by `loki` user, then changes ownership to `backup` for upload
- When restoring, files must be owned by `loki:loki` for Loki to read them
- RPO: 24 hours | RTO: 1 hour

---

## Duplicity Common Commands

### List all backups (collection status)
```bash
sudo -u backup duplicity collection-status rsync://orinocoz@backup.acme.ttu/<service>
```

### List files in backup
```bash
sudo -u backup duplicity list-current-files rsync://orinocoz@backup.acme.ttu/<service>
```

### Restore specific file
```bash
sudo -u backup duplicity restore --file-to-restore <filename> rsync://orinocoz@backup.acme.ttu/<service> /home/backup/restore/
```

### Restore from specific time
```bash
# 2 days ago
sudo -u backup duplicity restore --time 2D rsync://orinocoz@backup.acme.ttu/<service> /home/backup/restore/

# Specific date
sudo -u backup duplicity restore --time 2025-12-01 rsync://orinocoz@backup.acme.ttu/<service> /home/backup/restore/
```

### Verify backup integrity
```bash
sudo -u backup duplicity verify rsync://orinocoz@backup.acme.ttu/<service> /home/backup/<service>
```

---

## Troubleshooting

### SSH connection issues
```bash
# Test SSH connection
sudo -u backup ssh orinocoz@backup.acme.ttu id

# Check known_hosts
cat /home/backup/.ssh/known_hosts
```

### Permission denied errors
```bash
# Ensure backup user owns restore directory
sudo chown -R backup:backup /home/backup/restore/
```

### Duplicity GPG errors
All backups use `--no-encryption` flag. If you see GPG errors, ensure you're using the correct duplicity command with `--no-encryption`.

---

## Recovery Time Objectives (RTO)

| Service | RTO | Notes |
|---------|-----|-------|
| MySQL | 1 hour | Includes dump restore + replication sync |
| Prometheus | 2 hours | Includes snapshot restore + metric verification |
| Loki | 1 hour | Includes WAL/TSDB restore + log verification |

## Recovery Point Objectives (RPO)

| Service | RPO | Backup Schedule |
|---------|-----|-----------------|
| MySQL | 24 hours | Daily at 21:45 UTC |
| Prometheus | 24 hours | Daily at 21:45 UTC |
| Loki | 24 hours | Daily at 21:45 UTC |
