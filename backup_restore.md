# placeholder


# Infrastructure Installation and Data Restoration Guide

## Install and Configure Infrastructure with Ansible
To deploy the necessary infrastructure, execute the following command:

```bash
ansible-playbook infra.yaml
```

---

## Restore MySQL Data from Backup

### Steps:
1. **Log in to the Managed Host**  
   Access the writable host for Grafana (`orinocoz-2`):
   ```bash
   ssh orinocoz-2
   ```

2. **Restore the Backup Using Duplicity**  
   Execute the following command to restore MySQL data from the backup:
   ```bash
   sudo -u backup duplicity --no-encryption restore rsync://orinocoz@backup.acme.ttu/mysql /home/backup/restore/mysql
   ```

3. **Import the Restored Database**  
   Restore the database into MySQL:
   ```bash
   sudo mysql agama < /home/backup/restore/mysql/agama.sql
   ```

4. **Verify the Restoration**  
   Use the following command to confirm the tables are restored:
   ```bash
   sudo mysql -e "USE agama; SHOW TABLES;"
   ```

5. **Reload the Application**  
   After verifying, refresh the application page to ensure data restoration is successful.

---

## Restore InfluxDB Data from Backup

### Steps:
1. **Log in to the Managed Host**  
   Access the host for InfluxDB (`orinocoz-3`):
   ```bash
   ssh orinocoz-3
   ```

2. **Check for Existing Backup Data**  
   Ensure no leftover files or folders are present in `/home/backup/restore`. If any exist, remove them:
   ```bash
   sudo rm -rf /home/backup/restore/influxdb
   ```

3. **Restore the Backup Using Duplicity**  
   Restore InfluxDB data:
   ```bash
   sudo -u backup duplicity --no-encryption restore rsync://orinocoz@backup.acme.ttu/influxdb /home/backup/restore/influxdb
   ```

4. **Stop Telegraf Service**  
   Temporarily stop the Telegraf service:
   ```bash
   sudo service telegraf stop
   ```

5. **Drop the Existing Database**  
   Use InfluxDB to drop the current `telegraf` database:
   ```bash
   sudo influx -execute 'DROP DATABASE telegraf'
   ```

6. **Restore the Database**  
   Perform a portable restore of the `telegraf` database:
   ```bash
   sudo influxd restore -portable -db telegraf /home/backup/restore/influxdb
   ```

7. **Start Telegraf Service**  
   Restart the Telegraf service:
   ```bash
   sudo service telegraf start
   ```

8. **Verify the Restoration**  
   Check the InfluxDB data:
   ```bash
   influx
   > show databases;
   > use telegraf;
   > show measurements;
   ```

   To ensure the `syslog` data is intact, run:
   ```sql
   select * from syslog limit 10;
   ```

---

Make sure that all steps are followed accurately to guarantee a successful restoration of the infrastructure and data.

