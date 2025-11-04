# Lab 6: Physical backups and point-in-time recovery

**Estimated time:** 90 minutes  
**Difficulty level:** Intermediate to Advanced  
**PostgreSQL version:** 18.x  
**Platform requirements:** Windows only

---

## 1. Learning objectives

By the end of this lab, you will be able to:

- Understand the difference between physical and logical backups
- Configure WAL archiving for continuous backup
- Perform physical backups using pg_basebackup
- Verify backup integrity with pg_verifybackup
- Restore from a physical backup
- Configure and perform Point-in-Time Recovery (PITR)
- Understand WAL segments and their role in recovery
- Clone a running PostgreSQL cluster

---

## 2. Prerequisites

**Knowledge prerequisites:**
- Understanding of PGDATA directory structure
- Familiarity with Windows command line
- Basic understanding of database transactions
- Completed Lab 2 (Northwind database installation)

**Previous labs:**
- Lab 2 (Installing Northwind database) must be completed
- Lab 5 (Logical backups) recommended for comparison

**Standard environment checklist:**
- [ ] PostgreSQL 18 installed on Windows (port 5432)
- [ ] Northwind database exists with data
- [ ] Can connect as postgres superuser
- [ ] At least 2 GB free disk space
- [ ] Administrative access to modify postgresql.conf

---

## 3. Lab-specific setup

### Create directory structure for physical backups

```
mkdir C:\pg_physical_backups
mkdir C:\pg_physical_backups\basebackup
mkdir C:\pg_physical_backups\wal_archive
mkdir C:\pg_physical_backups\pitr_test
mkdir C:\pg_physical_backups\clone
```

### Verify Northwind database baseline

```sql
psql -h localhost -p 5432 -U postgres -d northwind
```

```sql
-- Record baseline counts
SELECT 'customers' AS table_name, COUNT(*) FROM customers
UNION ALL SELECT 'orders', COUNT(*) FROM orders
UNION ALL SELECT 'products', COUNT(*) FROM products
UNION ALL SELECT 'employees', COUNT(*) FROM employees;

-- Record a specific customer for PITR testing
SELECT customer_id, company_name, city FROM customers 
WHERE customer_id = 'ALFKI';

\q
```

Expected: 91 customers, 830 orders, 77 products, 9 employees

### Locate PGDATA directory

```
psql -h localhost -p 5432 -U postgres -c "SHOW data_directory;"
```

Expected output (may vary):
```
           data_directory            
-------------------------------------
 C:/Program Files/PostgreSQL/18/data
(1 row)
```

**Note this path** - you'll need it throughout the lab. We'll refer to it as `PGDATA`.

---

## 4. Concept overview

Physical backups (also called hot backups or base backups) copy the entire PGDATA directory and WAL (Write-Ahead Log) files at the filesystem level while the database is running. Unlike logical backups that extract data through SQL queries, physical backups capture the raw data files in an inconsistent state, then rely on WAL replay during restoration to achieve consistency. This approach is less invasive, supports Point-in-Time Recovery (PITR), and is ideal for large databases. However, physical backups only work between identical PostgreSQL major versions and operating system architectures. The process involves: configuring WAL archiving, taking a base backup with pg_basebackup, continuously archiving WAL segments, and during recovery, replaying WALs to bring the backup to a consistent state at any desired point in time.

---

## 5. Exercises

### Exercise 1.1: Configure WAL archiving

**Platform:** Windows  
**Objective:** Set up continuous WAL archiving, which is essential for PITR  
**Scenario:** You need to enable continuous archiving of WAL files to support point-in-time recovery for Northwind

**Background:**

WAL (Write-Ahead Log) files record every change made to the database. By default, PostgreSQL recycles WAL files once changes are written to data files. For PITR, we must archive these WALs before they're recycled, creating a continuous stream of transaction history.

**Tasks:**

1. **Locate and backup the current configuration:**

   ```
   cd "C:\Program Files\PostgreSQL\18\data"
   copy postgresql.conf postgresql.conf.backup
   copy pg_hba.conf pg_hba.conf.backup
   ```

2. **Edit postgresql.conf to enable WAL archiving:**

   Open `C:\Program Files\PostgreSQL\18\data\postgresql.conf` in a text editor (as Administrator)

   Find and modify these settings:

   ```
   # WAL ARCHIVING SETTINGS
   wal_level = replica                # Was probably 'replica' already
   archive_mode = on                  # Enable archiving (was probably 'off')
   archive_command = 'copy "%p" "C:\\pg_physical_backups\\wal_archive\\%f"'
   archive_timeout = 60               # Force WAL switch every 60 seconds
   max_wal_senders = 3                # Allow pg_basebackup connections
   ```

   **Explanation:**
   - `wal_level = replica`: Includes enough information for base backups and replication
   - `archive_mode = on`: Activates the archiving system
   - `archive_command`: Command executed for each WAL segment
     - `%p` = full path to WAL file
     - `%f` = WAL filename only
     - Uses Windows `copy` command to archive WALs
   - `archive_timeout = 60`: Forces WAL switch after 60 seconds even if WAL not full
   - `max_wal_senders = 3`: Allows pg_basebackup to request WAL streaming

3. **Configure replication connection in pg_hba.conf:**

   Open `C:\Program Files\PostgreSQL\18\data\pg_hba.conf` in text editor (as Administrator)

   Add this line near the top (after the comments):

   ```
   # TYPE  DATABASE        USER            ADDRESS                 METHOD
   host    replication     postgres        127.0.0.1/32            scram-sha-256
   ```

   **Explanation:**
   - `replication`: Special pseudo-database for backup/replication connections
   - `postgres`: User allowed to perform backups
   - `127.0.0.1/32`: Only allow connections from localhost
   - `scram-sha-256`: Secure authentication method

4. **Restart PostgreSQL to apply changes:**

   Open Services (Win+R, type `services.msc`)
   
   Find "postgresql-x64-18" service
   
   Right-click → Restart

   Or from command line (as Administrator):
   ```
   net stop postgresql-x64-18
   net start postgresql-x64-18
   ```

5. **Verify WAL archiving is active:**

   ```sql
   psql -h localhost -p 5432 -U postgres -d postgres
   ```

   ```sql
   -- Check archive settings
   SHOW wal_level;
   SHOW archive_mode;
   SHOW archive_command;

   -- Check if archiving is working
   SELECT archived_count, failed_count, last_archived_wal, last_archived_time
   FROM pg_stat_archiver;
   ```

   Expected output:
   ```
    wal_level 
   -----------
    replica
   (1 row)

    archive_mode 
   --------------
    on
   (1 row)

    archive_command                                                    
   --------------------------------------------------------------------
    copy "%p" "C:\\pg_physical_backups\\wal_archive\\%f"
   (1 row)

    archived_count | failed_count | last_archived_wal | last_archived_time  
   ----------------+--------------+-------------------+---------------------
                 1 |            0 | 000000010000000000000001 | 2025-11-04...
   (1 row)
   ```

6. **Force a WAL switch to test archiving:**

   ```sql
   SELECT pg_switch_wal();
   
   -- Wait 5 seconds
   SELECT pg_sleep(5);
   
   -- Check archiving happened
   SELECT archived_count FROM pg_stat_archiver;
   ```

   The `archived_count` should have increased.

7. **Verify WAL files in archive directory:**

   ```sql
   \q
   ```

   ```
   dir C:\pg_physical_backups\wal_archive
   ```

   Expected output:
   ```
   Directory of C:\pg_physical_backups\wal_archive

   11/04/2025  03:30 PM        16,777,216 000000010000000000000001
   11/04/2025  03:31 PM        16,777,216 000000010000000000000002
                  2 File(s)     33,554,432 bytes
   ```

   Each WAL file is 16 MB by default.

**Verification steps:**

- [ ] archive_mode is ON
- [ ] archive_command is configured
- [ ] WAL files appear in C:\pg_physical_backups\wal_archive
- [ ] pg_stat_archiver shows successful archiving (failed_count = 0)

---

### Exercise 1.2: Create baseline data for PITR testing

**Platform:** Windows  
**Objective:** Create timestamped data changes that we can recover to specific points  
**Scenario:** Before taking backups, create known data points in Northwind to test PITR

**Tasks:**

1. **Connect to Northwind:**

   ```sql
   psql -h localhost -p 5432 -U postgres -d northwind
   ```

2. **Record BEFORE backup timestamp:**

   ```sql
   -- Get current timestamp
   SELECT NOW() AS before_backup_time;
   ```

   **IMPORTANT:** Write down this timestamp! You'll need it for PITR.
   
   Example: `2025-11-04 15:35:22.123456-05`

3. **Insert test customer (will exist in backup):**

   ```sql
   INSERT INTO customers (customer_id, company_name, contact_name, city, country)
   VALUES ('TEST1', 'Test Company Before Backup', 'John Smith', 'New York', 'USA');
   
   -- Verify
   SELECT customer_id, company_name FROM customers WHERE customer_id = 'TEST1';
   ```

4. **Commit and wait:**

   ```sql
   -- Ensure data is committed
   SELECT pg_sleep(2);
   ```

5. **Exit for now:**

   ```sql
   \q
   ```

**Verification steps:**

- [ ] Have recorded the "before_backup_time" timestamp
- [ ] TEST1 customer exists in database
- [ ] At least 2 WAL files in archive directory

---

### Exercise 1.3: Perform base backup with pg_basebackup

**Platform:** Windows  
**Objective:** Create a physical backup using pg_basebackup  
**Scenario:** Take a base backup of the entire PostgreSQL cluster including Northwind

**Tasks:**

1. **Verify backup directory is empty:**

   ```
   dir C:\pg_physical_backups\basebackup
   ```

   Should show empty directory or not exist yet.

2. **Run pg_basebackup:**

   ```
   pg_basebackup -h localhost -p 5432 -U postgres -D C:\pg_physical_backups\basebackup -Fp -Xs -P -v
   ```

   **Explanation of options:**
   - `-h localhost -p 5432 -U postgres`: Connection parameters
   - `-D C:\pg_physical_backups\basebackup`: Destination directory
   - `-Fp`: Plain format (directory structure, not tar)
   - `-Xs`: Stream WALs during backup (includes in backup)
   - `-P`: Show progress
   - `-v`: Verbose output

   Expected output:
   ```
   pg_basebackup: initiating base backup, waiting for checkpoint to complete
   pg_basebackup: checkpoint completed
   pg_basebackup: write-ahead log start point: 0/3000028 on timeline 1
   pg_basebackup: starting background WAL receiver
   pg_basebackup: created temporary replication slot "pg_basebackup_12345"
   24567/24567 kB (100%), 1/1 tablespace
   pg_basebackup: write-ahead log end point: 0/3000100
   pg_basebackup: waiting for background process to finish streaming ...
   pg_basebackup: syncing data to disk ...
   pg_basebackup: renaming backup_manifest.tmp to backup_manifest
   pg_basebackup: base backup completed
   ```

3. **Examine the backup directory structure:**

   ```
   dir C:\pg_physical_backups\basebackup
   ```

   Expected output:
   ```
   Directory of C:\pg_physical_backups\basebackup

   <DIR>          base
   <DIR>          global
   <DIR>          pg_commit_ts
   <DIR>          pg_dynshmem
   <DIR>          pg_logical
   <DIR>          pg_multixact
   <DIR>          pg_notify
   <DIR>          pg_replslot
   <DIR>          pg_serial
   <DIR>          pg_snapshots
   <DIR>          pg_stat
   <DIR>          pg_stat_tmp
   <DIR>          pg_subtrans
   <DIR>          pg_tblspc
   <DIR>          pg_twophase
   <DIR>          pg_wal
   <DIR>          pg_xact
                  backup_label
                  backup_manifest
                  pg_hba.conf
                  pg_ident.conf
                  postgresql.auto.conf
                  postgresql.conf
                  PG_VERSION
   ```

   This is a complete copy of PGDATA!

4. **Check the backup_label file:**

   ```
   type C:\pg_physical_backups\basebackup\backup_label
   ```

   Expected content:
   ```
   START WAL LOCATION: 0/3000028 (file 000000010000000000000003)
   CHECKPOINT LOCATION: 0/3000060
   BACKUP METHOD: streamed
   BACKUP FROM: primary
   START TIME: 2025-11-04 15:40:15 EST
   LABEL: pg_basebackup base backup
   START TIMELINE: 1
   ```

   **Explanation:**
   - Shows exactly when and where backup started
   - Critical for determining which WALs are needed for recovery

5. **Check backup size:**

   ```
   powershell "Get-ChildItem C:\pg_physical_backups\basebackup -Recurse | Measure-Object -Property Length -Sum | Select-Object Sum"
   ```

   Expect: Several hundred MB (includes all databases, not just Northwind)

**Verification steps:**

- [ ] Backup completed successfully
- [ ] backup_label file exists
- [ ] Directory structure matches PGDATA
- [ ] backup_manifest exists

---

### Exercise 1.4: Verify backup integrity with pg_verifybackup

**Platform:** Windows  
**Objective:** Use pg_verifybackup to ensure backup is not corrupted  
**Scenario:** Before relying on the backup, verify its integrity

**Tasks:**

1. **Run pg_verifybackup:**

   ```
   pg_verifybackup C:\pg_physical_backups\basebackup
   ```

   **Explanation:**
   - Reads backup_manifest
   - Checks all files are present
   - Verifies checksums
   - Validates WAL records

   Expected output:
   ```
   backup successfully verified
   ```

2. **Verbose verification:**

   ```
   pg_verifybackup -v C:\pg_physical_backups\basebackup
   ```

   Shows detailed checking process:
   ```
   pg_verifybackup: parsing backup manifest
   pg_verifybackup: checking backup manifest
   pg_verifybackup: scanning directory "base"
   pg_verifybackup: scanning directory "global"
   pg_verifybackup: scanning directory "pg_wal"
   ...
   pg_verifybackup: backup successfully verified
   ```

3. **Test verification by corrupting a file (optional demonstration):**

   **WARNING:** Only do this on backup copy, NOT production!

   ```
   echo corrupted >> C:\pg_physical_backups\basebackup\base\5\1259
   pg_verifybackup C:\pg_physical_backups\basebackup
   ```

   Expected error:
   ```
   pg_verifybackup: error: file "base/5/1259" has incorrect size
   pg_verifybackup: backup verification failed
   ```

   **Restore the backup file from our good backup** (or retake the backup):
   ```
   rmdir /S /Q C:\pg_physical_backups\basebackup
   ```

   Then repeat Exercise 1.3 to retake the backup.

**Verification steps:**

- [ ] pg_verifybackup reports success
- [ ] Understand that verification ensures backup can be trusted

---

### Exercise 1.5: Create post-backup data for PITR testing

**Platform:** Windows  
**Objective:** Create more data changes AFTER backup to test PITR  
**Scenario:** Make changes to Northwind that will be recoverable via WAL replay

**Tasks:**

1. **Connect to Northwind:**

   ```sql
   psql -h localhost -p 5432 -U postgres -d northwind
   ```

2. **Wait and record AFTER backup timestamp:**

   ```sql
   -- Wait to ensure we're past backup time
   SELECT pg_sleep(5);
   
   -- Record timestamp AFTER backup
   SELECT NOW() AS after_backup_time;
   ```

   **IMPORTANT:** Write down this timestamp!
   
   Example: `2025-11-04 15:45:30.789012-05`

3. **Insert another test customer (after backup):**

   ```sql
   INSERT INTO customers (customer_id, company_name, contact_name, city, country)
   VALUES ('TEST2', 'Test Company After Backup', 'Jane Doe', 'Boston', 'USA');
   
   -- Verify
   SELECT customer_id, company_name FROM customers 
   WHERE customer_id IN ('TEST1', 'TEST2')
   ORDER BY customer_id;
   ```

   Expected:
   ```
    customer_id |        company_name        
   -------------+----------------------------
    TEST1       | Test Company Before Backup
    TEST2       | Test Company After Backup
   (2 rows)
   ```

4. **Make an order modification (after backup):**

   ```sql
   -- Record original freight
   SELECT order_id, freight FROM orders WHERE order_id = 10248;
   
   -- Update it
   UPDATE orders SET freight = 999.99 WHERE order_id = 10248;
   
   -- Verify
   SELECT order_id, freight FROM orders WHERE order_id = 10248;
   ```

   Expected: freight is now 999.99

5. **Record timestamp BEFORE disaster:**

   ```sql
   SELECT pg_sleep(5);
   SELECT NOW() AS before_disaster_time;
   ```

   **IMPORTANT:** Write down this timestamp!
   
   Example: `2025-11-04 15:47:45.123456-05`

6. **Simulate a disaster - accidental DELETE:**

   ```sql
   -- DISASTER! Accidentally delete all customers starting with 'A'
   DELETE FROM customers WHERE customer_id LIKE 'A%';
   
   -- Check the damage
   SELECT COUNT(*) FROM customers;
   ```

   Expected: Reduced count (lost customers ALFKI, ANATR, ANTON, etc.)

7. **Record AFTER disaster timestamp:**

   ```sql
   SELECT NOW() AS after_disaster_time;
   ```

   **IMPORTANT:** Write down this timestamp!
   
   Example: `2025-11-04 15:48:50.654321-05`

8. **Check what we lost:**

   ```sql
   -- Try to find ALFKI (should be gone)
   SELECT * FROM customers WHERE customer_id = 'ALFKI';
   ```

   Expected: 0 rows (disaster confirmed!)

9. **Exit:**

   ```sql
   \q
   ```

10. **Force WAL archiving of recent changes:**

    ```sql
    psql -h localhost -p 5432 -U postgres -c "SELECT pg_switch_wal();"
    psql -h localhost -p 5432 -U postgres -c "SELECT pg_sleep(3);"
    ```

11. **Verify WALs are archived:**

    ```
    dir C:\pg_physical_backups\wal_archive
    ```

    Should show multiple WAL files, including recent ones.

**Summary of timeline:**

Write down your four timestamps:
1. **before_backup_time**: Before base backup (TEST1 customer added)
2. **after_backup_time**: After base backup (TEST2 customer added)
3. **before_disaster_time**: After good changes (order 10248 modified to 999.99)
4. **after_disaster_time**: After disaster (customers deleted)

**Verification steps:**

- [ ] Have all four timestamps recorded
- [ ] TEST1 and TEST2 customers were in database
- [ ] Orders freight was modified
- [ ] Customers were deleted (disaster occurred)
- [ ] Multiple WAL files exist in archive

---

### Exercise 1.6: Restore from physical backup (basic recovery)

**Platform:** Windows  
**Objective:** Perform a basic restore that brings us to the end of the backup  
**Scenario:** Practice basic physical backup restoration

**Tasks:**

1. **Stop PostgreSQL:**

   ```
   net stop postgresql-x64-18
   ```

   Or use Services GUI.

2. **Rename current PGDATA (backup current state):**

   ```
   cd "C:\Program Files\PostgreSQL\18"
   move data data_disaster
   ```

3. **Copy backup to PGDATA location:**

   ```
   xcopy C:\pg_physical_backups\basebackup "C:\Program Files\PostgreSQL\18\data" /E /I /H /Y
   ```

   **Explanation:**
   - `/E`: Copy subdirectories including empty
   - `/I`: Assume destination is directory
   - `/H`: Copy hidden and system files
   - `/Y`: Suppress prompt to overwrite

4. **Create recovery.signal file (tells PostgreSQL to enter recovery mode):**

   ```
   echo. > "C:\Program Files\PostgreSQL\18\data\recovery.signal"
   ```

   **Explanation:**
   - Presence of recovery.signal triggers recovery mode
   - PostgreSQL will replay WALs from archive
   - File is deleted automatically when recovery completes

5. **Configure recovery in postgresql.conf:**

   Edit `C:\Program Files\PostgreSQL\18\data\postgresql.conf`

   Add or modify:
   ```
   restore_command = 'copy "C:\\pg_physical_backups\\wal_archive\\%f" "%p"'
   ```

   **Explanation:**
   - `restore_command`: How to retrieve archived WAL files
   - `%f`: WAL filename to restore
   - `%p`: Full path where PostgreSQL expects the WAL
   - Uses Windows `copy` to retrieve WALs from archive

6. **Start PostgreSQL:**

   ```
   net start postgresql-x64-18
   ```

7. **Watch the recovery in logs:**

   ```
   type "C:\Program Files\PostgreSQL\18\data\log\postgresql-*.log" | findstr /C:"redo" /C:"recovery"
   ```

   Expected log messages:
   ```
   LOG:  starting point-in-time recovery
   LOG:  restored log file "000000010000000000000003" from archive
   LOG:  redo starts at 0/3000028
   LOG:  restored log file "000000010000000000000004" from archive
   LOG:  consistent recovery state reached at 0/4000138
   LOG:  restored log file "000000010000000000000005" from archive
   LOG:  redo done at 0/5000180
   LOG:  restored log file "000000010000000000000005" from archive
   LOG:  selected new timeline ID: 2
   LOG:  archive recovery complete
   LOG:  database system is ready to accept connections
   ```

8. **Verify recovery completed:**

   ```sql
   psql -h localhost -p 5432 -U postgres -d northwind
   ```

   ```sql
   -- Check if TEST1 customer exists (from before backup)
   SELECT customer_id, company_name FROM customers WHERE customer_id = 'TEST1';
   
   -- Check if TEST2 customer exists (from after backup, should exist if WALs replayed)
   SELECT customer_id, company_name FROM customers WHERE customer_id = 'TEST2';
   
   -- Check order 10248 freight (should be 999.99 if WALs replayed)
   SELECT order_id, freight FROM orders WHERE order_id = 10248;
   
   -- Check if ALFKI exists (was deleted in disaster, should not exist)
   SELECT * FROM customers WHERE customer_id = 'ALFKI';
   
   -- Check total customer count
   SELECT COUNT(*) FROM customers;
   ```

**Expected results:**
- TEST1: EXISTS (was in backup)
- TEST2: EXISTS (restored from WALs)
- Order 10248 freight: 999.99 (restored from WALs)
- ALFKI: DOES NOT EXIST (disaster was restored)
- Customer count: REDUCED (disaster was restored)

**This demonstrates:** Basic recovery restores everything up to the last archived WAL, including the disaster!

9. **Exit:**

   ```sql
   \q
   ```

**Verification steps:**

- [ ] PostgreSQL started successfully
- [ ] Recovery completed (recovery.signal was removed automatically)
- [ ] TEST1 and TEST2 both exist
- [ ] Disaster was also restored (not what we want - see next exercise)

---

### Exercise 1.7: Point-in-Time Recovery (PITR) - recover to before disaster

**Platform:** Windows  
**Objective:** Perform PITR to restore database to a point before the disaster  
**Scenario:** Use PITR to recover Northwind to the moment before customers were deleted

**Tasks:**

1. **Stop PostgreSQL:**

   ```
   net stop postgresql-x64-18
   ```

2. **Remove the post-disaster PGDATA:**

   ```
   rmdir /S /Q "C:\Program Files\PostgreSQL\18\data"
   ```

3. **Restore from backup again:**

   ```
   xcopy C:\pg_physical_backups\basebackup "C:\Program Files\PostgreSQL\18\data" /E /I /H /Y
   ```

4. **Create recovery.signal again:**

   ```
   echo. > "C:\Program Files\PostgreSQL\18\data\recovery.signal"
   ```

5. **Configure PITR in postgresql.conf:**

   Edit `C:\Program Files\PostgreSQL\18\data\postgresql.conf`

   Modify to add recovery_target_time (use your **before_disaster_time** timestamp):

   ```
   restore_command = 'copy "C:\\pg_physical_backups\\wal_archive\\%f" "%p"'
   recovery_target_time = '2025-11-04 15:47:45.123456-05'
   recovery_target_action = 'promote'
   ```

   **CRITICAL:** Replace the timestamp with YOUR **before_disaster_time** from Exercise 1.5!

   **Explanation:**
   - `recovery_target_time`: Stop recovery at this exact moment
   - `recovery_target_action = 'promote'`: After reaching target, start as normal database
   - Alternative actions: 'pause' (stop but don't start), 'shutdown' (stop and shutdown)

6. **Start PostgreSQL:**

   ```
   net start postgresql-x64-18
   ```

7. **Watch recovery in logs:**

   ```
   type "C:\Program Files\PostgreSQL\18\data\log\postgresql-*.log" | findstr /C:"redo" /C:"recovery" /C:"target"
   ```

   Expected log messages:
   ```
   LOG:  starting point-in-time recovery to 2025-11-04 15:47:45.123456-05
   LOG:  restored log file "000000010000000000000003" from archive
   LOG:  redo starts at 0/3000028
   LOG:  restored log file "000000010000000000000004" from archive
   LOG:  recovery stopping before commit of transaction 750, time 2025-11-04 15:48:50
   LOG:  pausing at the end of recovery
   LOG:  recovery target reached
   LOG:  selected new timeline ID: 2
   LOG:  archive recovery complete
   LOG:  database system is ready to accept connections
   ```

8. **Verify PITR success:**

   ```sql
   psql -h localhost -p 5432 -U postgres -d northwind
   ```

   ```sql
   -- Check if TEST1 exists (should exist - before backup)
   SELECT customer_id, company_name FROM customers WHERE customer_id = 'TEST1';
   
   -- Check if TEST2 exists (should exist - after backup but before disaster)
   SELECT customer_id, company_name FROM customers WHERE customer_id = 'TEST2';
   
   -- Check order 10248 (should be 999.99 - modified before disaster)
   SELECT order_id, freight FROM orders WHERE order_id = 10248;
   
   -- Check if ALFKI exists (should EXIST - disaster didn't happen yet!)
   SELECT customer_id, company_name FROM customers WHERE customer_id = 'ALFKI';
   
   -- Check customer count (should be FULL count - no deletion occurred)
   SELECT COUNT(*) FROM customers;
   ```

**Expected results:**
- TEST1: EXISTS ✓
- TEST2: EXISTS ✓
- Order 10248 freight: 999.99 ✓
- ALFKI: EXISTS ✓ (disaster avoided!)
- Customer count: FULL COUNT ✓ (disaster avoided!)

**Success!** We've recovered to the exact moment before the disaster, keeping all the good changes but avoiding the bad DELETE.

9. **Verify specific customers that were deleted:**

   ```sql
   -- These customers were deleted in the disaster
   -- They should all exist now
   SELECT customer_id, company_name 
   FROM customers 
   WHERE customer_id IN ('ALFKI', 'ANATR', 'ANTON', 'AROUT')
   ORDER BY customer_id;
   ```

   Expected: All 4 customers exist!

10. **Exit:**

    ```sql
    \q
    ```

**Verification steps:**

- [ ] Recovery stopped at target time
- [ ] All test customers exist
- [ ] Disaster (DELETE) was avoided
- [ ] Good changes (TEST2, freight update) are present
- [ ] PITR successfully recovered to specific point in time

---

### Exercise 1.8: Clone a running cluster

**Platform:** Windows  
**Objective:** Use pg_basebackup to clone the running cluster to a different port  
**Scenario:** Create a development clone of Northwind for testing without affecting production

**Tasks:**

1. **Take a new base backup for cloning:**

   ```
   pg_basebackup -h localhost -p 5432 -U postgres -D C:\pg_physical_backups\clone -Fp -Xs -P
   ```

2. **Modify postgresql.conf in clone:**

   Edit `C:\pg_physical_backups\clone\postgresql.conf`

   Change these settings:

   ```
   port = 5433                        # Different port!
   archive_mode = off                 # Don't archive from clone
   ```

3. **Start the cloned cluster:**

   ```
   pg_ctl -D C:\pg_physical_backups\clone start
   ```

   Expected output:
   ```
   waiting for server to start....
   LOG:  starting PostgreSQL 18.0 on x86_64-pc-windows-msvc, compiled by msvc
   LOG:  listening on IPv4 address "0.0.0.0", port 5433
   LOG:  database system was interrupted; last known up at 2025-11-04 16:00:00 EST
   LOG:  redo starts at 0/7000028
   LOG:  redo done at 0/7000138
   LOG:  database system is ready to accept connections
   done
   server started
   ```

   **Note:** The clone goes through crash recovery, just like a restored backup

4. **Connect to clone on port 5433:**

   ```sql
   psql -h localhost -p 5433 -U postgres -d northwind
   ```

   ```sql
   -- Show we're on port 5433
   SHOW port;
   
   -- Verify Northwind data is present
   SELECT COUNT(*) FROM customers;
   SELECT COUNT(*) FROM orders;
   
   -- Make a test change (won't affect port 5432)
   INSERT INTO customers (customer_id, company_name, city, country)
   VALUES ('CLONE', 'Test Clone Customer', 'Clone City', 'Cloneland');
   
   \q
   ```

5. **Verify original (port 5432) is unchanged:**

   ```sql
   psql -h localhost -p 5432 -U postgres -d northwind
   ```

   ```sql
   -- Should NOT have CLONE customer
   SELECT * FROM customers WHERE customer_id = 'CLONE';
   
   \q
   ```

   Expected: 0 rows (clone change didn't affect original)

6. **Stop the cloned cluster:**

   ```
   pg_ctl -D C:\pg_physical_backups\clone stop
   ```

**Verification steps:**

- [ ] Clone started successfully on port 5433
- [ ] Clone has full copy of Northwind database
- [ ] Changes to clone don't affect original
- [ ] Understand how to run multiple PostgreSQL instances

---

### Exercise 1.9: Understanding WAL files and timelines

**Platform:** Windows  
**Objective:** Explore WAL files and understand PostgreSQL timelines  
**Scenario:** Deep dive into the mechanics of physical backups

**Tasks:**

1. **Examine WAL filenames:**

   ```
   dir C:\pg_physical_backups\wal_archive
   ```

   Expected format: `000000010000000000000001`

   **WAL Filename structure:**
   - `00000001`: Timeline ID (increments on recovery)
   - `000000000`: High 32 bits of WAL position
   - `0000001`: Low 32 bits / segment number

2. **Check current WAL location:**

   ```sql
   psql -h localhost -p 5432 -U postgres
   ```

   ```sql
   -- Current WAL insert location
   SELECT pg_current_wal_lsn();
   
   -- Current WAL file
   SELECT pg_walfile_name(pg_current_wal_lsn());
   
   -- Timeline ID
   SELECT timeline_id FROM pg_control_checkpoint();
   ```

   Expected output:
   ```
    pg_current_wal_lsn 
   --------------------
    0/8001234
   (1 row)

    pg_walfile_name       
   --------------------------
    000000020000000000000008
   (1 row)

    timeline_id 
   -------------
             2
   (1 row)
   ```

   **Note:** Timeline is 2 because we did recovery (timeline incremented)

3. **Calculate WAL size and count:**

   ```sql
   -- How many WAL files archived?
   SELECT archived_count FROM pg_stat_archiver;
   
   -- WAL files waiting to be archived
   SELECT COUNT(*) FROM pg_ls_waldir() WHERE name ~ '^[0-9A-F]{24}$';
   
   \q
   ```

4. **Check archive directory size:**

   ```
   powershell "Get-ChildItem C:\pg_physical_backups\wal_archive | Measure-Object -Property Length -Sum | Select-Object Sum"
   ```

   Calculate: Number of files × 16 MB = total archive size

5. **Understand timeline history:**

   The timeline file records recovery history:

   ```
   type "C:\Program Files\PostgreSQL\18\data\pg_wal\00000002.history"
   ```

   Expected content:
   ```
   1       0/5000180       before 2025-11-04 15:47:45.123456-05

   Previous timeline was 1, switched to timeline 2 during recovery to 2025-11-04 15:47:45.123456-05
   ```

   **Explanation:**
   - Shows when and why timeline changed
   - Documents PITR recovery points
   - Important for understanding recovery history

**Verification steps:**

- [ ] Understand WAL filename structure
- [ ] Can identify current timeline
- [ ] Understand how timelines increment during recovery
- [ ] Can calculate WAL archive size

---

## 6. Validation checklist

After completing all exercises, verify:

- [ ] WAL archiving is configured and working
- [ ] Can perform pg_basebackup successfully
- [ ] Can verify backups with pg_verifybackup
- [ ] Can restore from physical backup
- [ ] Can perform PITR to specific timestamp
- [ ] Understand timeline concept
- [ ] Can clone a running cluster
- [ ] Understand restore_command and recovery.signal
- [ ] Know the difference between physical and logical backups

**Final validation queries (on restored database):**

```sql
psql -h localhost -p 5432 -U postgres -d northwind

-- Verify PITR worked correctly
SELECT 
    'TEST1' AS test,
    CASE WHEN EXISTS (SELECT 1 FROM customers WHERE customer_id = 'TEST1')
        THEN 'EXISTS' ELSE 'MISSING' END AS status
UNION ALL
SELECT 'TEST2',
    CASE WHEN EXISTS (SELECT 1 FROM customers WHERE customer_id = 'TEST2')
        THEN 'EXISTS' ELSE 'MISSING' END
UNION ALL
SELECT 'ALFKI',
    CASE WHEN EXISTS (SELECT 1 FROM customers WHERE customer_id = 'ALFKI')
        THEN 'EXISTS' ELSE 'MISSING' END;
```

Expected: All three should be 'EXISTS'

---

## 7. Troubleshooting guide

### Windows

**Problem:** "could not start archiving"
- **Solution:** Check archive_command path uses double backslashes
- **Verify:** Directory C:\pg_physical_backups\wal_archive exists
- **Check:** PostgreSQL service account has write permission

**Problem:** "no pg_hba.conf entry for replication"
- **Solution:** Add replication entry to pg_hba.conf as shown in Exercise 1.1
- **Remember:** Must restart PostgreSQL after pg_hba.conf changes

**Problem:** "archive command failed"
- **Solution:** Check Windows Event Viewer for details
- **Test manually:** `copy "test.txt" "C:\pg_physical_backups\wal_archive\test.txt"`
- **Verify:** No permission issues on target directory

**Problem:** pg_basebackup hangs or times out
- **Solution:** Check max_wal_senders >= 2
- **Verify:** No firewall blocking port 5432
- **Check:** Enough disk space for backup

**Problem:** "requested WAL segment has already been removed"
- **Solution:** Increase wal_keep_size or use replication slot
- **Or:** archive_timeout may be too large

**Problem:** PITR goes too far (past target time)
- **Solution:** Check clock synchronization
- **Verify:** Used correct timestamp format with timezone
- **Remember:** Format must match: 'YYYY-MM-DD HH:MI:SS.ffffff+TZ'

**Problem:** recovery.signal not working
- **Solution:** Must be plain text file with no extension
- **Verify:** Not recovery.signal.txt
- **Create:** Use `echo. >` not `echo "" >`

**Problem:** restore_command fails
- **Solution:** Check paths use double backslashes in postgresql.conf
- **Test:** Can you manually copy a WAL file using the command?
- **Verify:** WAL archive directory is accessible

**Problem:** "timeline 2 of the replica does not match primary"
- **Solution:** This is expected after recovery - timeline incremented
- **Understand:** Timelines prevent accidental wrong-path recovery

**Problem:** Can't stop PostgreSQL service
- **Solution:** Check for active connections: `SELECT * FROM pg_stat_activity;`
- **Terminate:** `SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE pid <> pg_backend_pid();`

---

## 8. Questions

1. What are the key differences between physical and logical backups, and when would you choose each?

2. Why is WAL archiving essential for PITR, and what would happen if a WAL segment was missing from the archive?

3. Explain what happens during the recovery process when PostgreSQL replays WALs. Why does the backup start in an inconsistent state?

4. What is a timeline in PostgreSQL, and why does it increment after recovery? What problems does this prevent?

5. How does recovery_target_time provide more granular recovery than just restoring to the end of WALs?

---

## 9. Clean-up

To restore normal operation and clean up test data:

```sql
-- Connect to Northwind
psql -h localhost -p 5432 -U postgres -d northwind

-- Remove test customers
DELETE FROM customers WHERE customer_id IN ('TEST1', 'TEST2', 'CLONE');

-- Reset order freight
UPDATE orders SET freight = 32.38 WHERE order_id = 10248;

-- Verify cleanup
SELECT COUNT(*) FROM customers;
```

Remove backup directories (keep if you want to practice):

```
rmdir /S /Q C:\pg_physical_backups\basebackup
rmdir /S /Q C:\pg_physical_backups\clone
rmdir /S /Q "C:\Program Files\PostgreSQL\18\data_disaster"
```

Keep WAL archive (good practice) or remove:

```
REM Optional - removes all archived WALs
rmdir /S /Q C:\pg_physical_backups\wal_archive
mkdir C:\pg_physical_backups\wal_archive
```

**Note:** Do NOT disable WAL archiving in production - leave it configured.

---

## 10. Key takeaways

- Physical backups copy PGDATA and WALs at filesystem level, not through SQL
- WAL archiving creates continuous transaction history for PITR
- pg_basebackup is the standard tool for physical backups and cloning
- pg_verifybackup ensures backup integrity before you need it
- recovery.signal triggers recovery mode on startup
- restore_command tells PostgreSQL how to retrieve archived WALs
- PITR allows recovery to any point in time between backup and now
- Timelines track recovery history and prevent wrong-path recovery
- Physical backups are faster for large databases but version-specific
- Always test your backups - a backup you can't restore is useless

---

## 11. Additional resources

**Official PostgreSQL 18 documentation:**
- [Continuous Archiving and PITR](https://www.postgresql.org/docs/18/continuous-archiving.html)
- [pg_basebackup](https://www.postgresql.org/docs/18/app-pgbasebackup.html)
- [pg_verifybackup](https://www.postgresql.org/docs/18/app-pgverifybackup.html)
- [Recovery Configuration](https://www.postgresql.org/docs/18/recovery-config.html)
- [Write-Ahead Logging (WAL)](https://www.postgresql.org/docs/18/wal.html)

**Advanced backup tools:**
- [pgBackRest](https://pgbackrest.org/) - Enterprise backup and restore
- [Barman](https://www.pgbarman.org/) - Backup and Recovery Manager
- [WAL-G](https://github.com/wal-g/wal-g) - Archival restoration tool

---

## 12. Appendices

### Appendix A: Quick reference commands

**Configure WAL archiving:**
```
# In postgresql.conf
wal_level = replica
archive_mode = on
archive_command = 'copy "%p" "C:\\archive\\%f"'
max_wal_senders = 3
```

**Take base backup:**
```
pg_basebackup -h localhost -p 5432 -U postgres -D C:\backup -Fp -Xs -P -v
```

**Verify backup:**
```
pg_verifybackup C:\backup
```

**Configure PITR recovery (in postgresql.conf):**
```
restore_command = 'copy "C:\\archive\\%f" "%p"'
recovery_target_time = '2025-11-04 15:00:00-05'
recovery_target_action = 'promote'
```

**Trigger recovery:**
```
echo. > recovery.signal
```

**Force WAL switch:**
```sql
SELECT pg_switch_wal();
```

**Check archiving status:**
```sql
SELECT * FROM pg_stat_archiver;
```

### Appendix B: Recovery target options

| Parameter                 | Description         | Example                  |
| ------------------------- | ------------------- | ------------------------ |
| recovery_target_time      | Specific timestamp  | '2025-11-04 15:47:45-05' |
| recovery_target_xid       | Transaction ID      | '1234567'                |
| recovery_target_name      | Named restore point | 'before_disaster'        |
| recovery_target_lsn       | WAL location        | '0/5000180'              |
| recovery_target_immediate | End of backup       | N/A (boolean)            |

### Appendix C: Timeline management

**Timeline history:**
- Timeline 1: Original database
- Timeline 2: After first recovery
- Timeline 3: After recovery from timeline 2
- And so on...

**View current timeline:**
```sql
SELECT timeline_id FROM pg_control_checkpoint();
```

**View timeline history files:**
```
dir "C:\Program Files\PostgreSQL\18\data\pg_wal\*.history"
```

### Appendix D: WAL archiving strategies

**Local copy (development):**
```
archive_command = 'copy "%p" "C:\\archive\\%f"'
```

**Network copy (production):**
```
archive_command = 'copy "%p" "\\\\backup-server\\pg_wal\\%f"'
```

**With verification:**
```
archive_command = 'copy "%p" "C:\\archive\\%f" && if exist "C:\\archive\\%f" exit 0'
```

**Compressed archiving:**
```
archive_command = '7z a -t7z "C:\\archive\\%f.7z" "%p"'
```

### Appendix E: Backup validation checklist

- [ ] pg_verifybackup reports success
- [ ] backup_manifest exists
- [ ] backup_label shows correct start time
- [ ] Can list files in backup directory
- [ ] WAL archive has files after backup
- [ ] Test restore on non-production system
- [ ] Document recovery time objective (RTO)
- [ ] Document recovery point objective (RPO)
- [ ] Schedule regular backup testing
- [ ] Monitor archive disk space

---

## 13. Answers

### Answer to Question 1

Physical backups copy raw database files (PGDATA directory and WALs) at the filesystem level while the database runs, creating an initially inconsistent copy that relies on WAL replay to achieve consistency. They're faster for large databases (TB+), support PITR, are less invasive to ongoing operations, but only work between identical PostgreSQL versions and architectures. Logical backups use pg_dump to extract data through SQL queries, creating consistent, portable backups that work across versions and platforms, but they're slower for large databases, can impact concurrent transactions, and don't support PITR. Choose physical backups for large production databases requiring PITR and fast recovery, or when cloning servers. Choose logical backups for database migrations, version upgrades, selective table backups, or when you need human-readable SQL output.

### Answer to Question 2

WAL archiving captures every transaction after the base backup, creating a continuous record of all database changes. Without archived WALs, you can only restore to the exact moment the base backup was taken—losing all subsequent work. If even one WAL segment is missing from the archive sequence, recovery fails completely because PostgreSQL cannot guarantee consistency without the complete transaction chain. This is why archive_command reliability is critical and why production systems often use tools like pgBackRest that verify archival success, maintain redundant archives, and alert on failures. The archive forms a time machine allowing recovery to any second after the backup, but only if the WAL chain is unbroken.

### Answer to Question 3

During recovery, PostgreSQL starts from the base backup (which is an inconsistent snapshot taken while the database was running) and replays Write-Ahead Log (WAL) files in sequence, re-executing every committed transaction. The backup is inconsistent because pg_basebackup copied files over a period of time while transactions were modifying data—some changes were captured mid-transaction, tables are from different points in time. WAL replay brings consistency by: starting from the backup_label's START WAL LOCATION, reading archived WAL segments in order, redoing all committed transactions, ignoring uncommitted ones, and applying all changes until reaching the recovery target or WAL end. This mimics crash recovery—PostgreSQL uses the same self-healing mechanism that handles unexpected shutdowns.

### Answer to Question 4

A timeline is a distinct history branch in PostgreSQL's WAL sequence that increments each time the database completes recovery (PITR, crash recovery, failover). Timeline IDs appear in WAL filenames (first 8 digits) and track which version of history you're on. After PITR recovery, the timeline increments to prevent accidental "wrong-path recovery"—where you might mistakenly replay WALs from the old timeline (including the disaster you just recovered from) onto your recovered database. Without timelines, you could restore to Tuesday 3pm, then accidentally replay Wednesday's WALs containing the disaster you were avoiding. Timelines make history branches explicit and incompatible, forcing administrators to explicitly choose which history path to follow. Timeline history files (.history) document when and why each timeline switch occurred, creating an audit trail of recoveries.

### Answer to Question 5

Without recovery_target_time, basic physical backup restoration replays all archived WALs until none remain, bringing you to the most recent committed transaction—which likely includes the disaster you want to avoid. recovery_target_time provides surgical precision by instructing PostgreSQL to stop replaying WALs at an exact timestamp (recovery_target_time), specific transaction ID (recovery_target_xid), or named restore point (recovery_target_name). This allows you to recover to "5 minutes before the disaster" or "just before the bad deployment" rather than "everything up to our last WAL." Combined with recovery_target_action, you can pause recovery to inspect the database before promoting, or automatically promote if the recovery point is correct. This granularity is what makes PITR powerful for recovering from logical errors (bad DELETEs, incorrect updates) while preserving all good work done after the base backup but before the disaster.