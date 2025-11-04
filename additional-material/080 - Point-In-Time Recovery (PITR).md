# Point-In-Time Recovery (PITR)

Point-In-Time Recovery, commonly abbreviated as PITR, is one of the most powerful disaster recovery techniques available in PostgreSQL. While physical backups provide a way to restore a database to the moment the backup was taken, PITR extends this capability by allowing you to restore your database to any specific point in time between the backup and the present. This chapter explores the concepts, configuration, implementation, and best practices for PITR in PostgreSQL 18.

In this chapter, we will cover the following topics:

- Understanding PITR concepts and architecture
- Configuring WAL archiving for PITR
- Performing PITR operations
- Advanced PITR techniques and recovery targets
- Monitoring and troubleshooting PITR
- PITR best practices and production considerations

## Technical requirements

You need to know about the following to complete this chapter:

- How to perform physical backups using pg_basebackup
- Understanding of Write-Ahead Logs (WALs) and their role in PostgreSQL
- Familiarity with PostgreSQL configuration files (postgresql.conf)
- Basic command-line operations (Windows Command Prompt/PowerShell or Linux/Unix shell)
- Understanding of filesystem operations and permissions

The chapter examples assume you have:

- PostgreSQL 18 installed and running on Windows and/or Linux
- Sufficient disk space for WAL archives (at least 10-20 GB recommended for testing)
- Access to the PostgreSQL data directory (PGDATA)
- Appropriate permissions to stop and start PostgreSQL services

**Platform notes:**
- Windows examples use paths like `C:\archive\` and commands like `copy`
- Linux examples use paths like `/mnt/archive/` and commands like `cp`
- Both platforms are covered throughout this chapter

**Quick reference: Platform differences**

| Feature           | Linux                         | Windows                                    |
| ----------------- | ----------------------------- | ------------------------------------------ |
| Path separator    | `/`                           | `\` (escaped as `\\` in config)            |
| Archive directory | `/mnt/archive`                | `C:\archive`                               |
| Copy command      | `cp`                          | `copy`                                     |
| Existence check   | `test ! -f`                   | `IF NOT EXIST`                             |
| Permissions       | `chmod 700` / `chown`         | `icacls`                                   |
| Service control   | `systemctl` / `pg_ctl`        | `net start/stop` / Services GUI            |
| Log location      | `/var/log/postgresql/`        | `C:\Program Files\PostgreSQL\18\data\log\` |
| View logs         | `tail -f` / `journalctl`      | PowerShell `Get-Content -Wait`             |
| Compression       | `gzip`                        | `7-Zip` / PowerShell                       |
| PGDATA default    | `/var/lib/postgresql/18/main` | `C:\Program Files\PostgreSQL\18\data`      |

## Understanding PITR concepts and architecture

Point-In-Time Recovery represents a sophisticated approach to database recovery that goes beyond simple backup restoration. To truly appreciate PITR's capabilities, we must first understand the underlying architecture and the components that make it possible.

### The foundation of PITR

PITR builds upon PostgreSQL's Write-Ahead Logging mechanism. As you learned in *Chapter 11*, *Transactions, MVCC, WALs, and Checkpoints*, every modification to the database is first written to the WAL before being applied to the actual data files. This design choice, originally made to ensure crash recovery, provides the foundation for PITR.

Under normal circumstances, PostgreSQL generates WAL segments, uses them for crash recovery, and then recycles them once the data they protect has been safely written to disk. However, for PITR, we need to preserve these WAL segments continuously. By archiving every WAL segment as it's created, we maintain a complete history of all database changes from the moment we took a base backup.

The PITR process can be visualized as follows:

![PITR Architecture](media/pitr_architecture.jpg)

*Figure: PITR Architecture - Base Backup + Continuous WAL Archive*

The combination of a base backup (taken at time T0) and the continuous stream of archived WAL files allows us to reconstruct the database state at any point from T0 onwards, up to the last available WAL file.

### How PITR differs from standard backups

A standard physical backup, taken with pg_basebackup, captures the database at a specific moment. If you need to restore, you get exactly that momentâ€"no more, no less. This is useful, but it has limitations:

- You cannot recover to a point after the backup was taken
- You cannot avoid restoring corrupted data if corruption occurred before the backup
- You cannot selectively recover to just before an accidental DROP TABLE statement

PITR solves these problems by giving you temporal flexibility. With PITR, you can:

- Restore to any timestamp between the base backup and the latest archived WAL
- Restore to a specific transaction ID
- Restore to a named restore point created by the administrator
- Stop recovery just before a known problematic transaction

### Recovery targets

PostgreSQL 18 supports several types of recovery targets:

**Time-based recovery**: Restore to a specific timestamp. This is the most common recovery target and is expressed as a standard PostgreSQL timestamp:

```sql
recovery_target_time = '2025-11-04 14:30:00'
```

**Transaction ID-based recovery**: Restore up to (but not including) a specific transaction:

```sql
recovery_target_xid = '1234567'
```

**Named restore points**: Restore to a logical marker created by the DBA:

```sql
-- Creating a restore point during normal operation
SELECT pg_create_restore_point('before_major_update');

-- Later, during recovery
recovery_target_name = 'before_major_update'
```

**LSN-based recovery**: Restore to a specific Log Sequence Number, useful for precise replication scenarios:

```sql
recovery_target_lsn = '0/3000000'
```

**Immediate recovery**: Apply all available WAL segments and then stop:

```sql
recovery_target = 'immediate'
```

### The WAL archive

The WAL archive is a directory (or set of directories) where PostgreSQL stores copies of completed WAL segments. This archive can be:

- A local directory on the same server (simple but not disaster-proof)
- A network-mounted filesystem (NFS, CIFS, etc.)
- Cloud storage (AWS S3, Azure Blob Storage, Google Cloud Storage)
- A dedicated backup server accessed via SSH, rsync, or specialized tools

The archive grows continuously as long as the database is running and making changes. Planning for adequate storage and implementing archive cleanup strategies is crucial for production systems.

### Timeline IDs and recovery

When PostgreSQL performs PITR and starts up as a recovered database, it creates a new timeline. A timeline represents a sequence of WAL records following a particular recovery path. This concept becomes important when you need to perform multiple recoveries or when dealing with replication scenarios.

Each timeline has a unique integer identifier. When PostgreSQL starts normally, it continues on the current timeline. When recovery is performed and the database is started, a new timeline is created. This mechanism prevents accidental mixing of WAL data from different recovery scenarios.

Timeline history files (with the .history extension) are created in the pg_wal directory and contain information about when and why timeline switches occurred.

## Configuring WAL archiving for PITR

Implementing PITR requires proper configuration of WAL archiving. This section walks through the complete configuration process, explaining each parameter and its implications.

### Essential configuration parameters

To enable WAL archiving, you need to modify several parameters in postgresql.conf. Let's examine each one:

#### wal_level

The wal_level parameter controls how much information is written to the WAL. For PITR, you need to set it to replica or higher:

```ini
wal_level = replica
```

This setting ensures that enough information is logged to support both PITR and streaming replication. The other possible values are:

- `minimal`: Only information needed for crash recovery (PITR not possible)
- `replica`: Adds information needed for WAL archiving and replication
- `logical`: Adds information needed for logical decoding

For PITR purposes, replica is the minimum required level, and it's the default in PostgreSQL 18.

#### archive_mode

This boolean parameter enables or disables WAL archiving:

```ini
archive_mode = on
```

When set to on, PostgreSQL will execute the archive_command for each completed WAL segment. Note that enabling archive_mode requires a server restart.

In PostgreSQL 18, archive_mode can also be set to always, which forces archiving even on standby servers. This is useful in advanced replication scenarios.

#### archive_command

The archive_command is executed for each WAL segment that needs to be archived. This is where you specify how and where WAL files should be copied.

**Placeholders used in archive_command:**
- `%p`: Placeholder for the full path of the WAL file to archive
- `%f`: Placeholder for the WAL filename only

The archive_command must return zero exit status on success. If it returns non-zero, PostgreSQL will retry the archiving later.

**Important**: The archive_command must ensure that files are not overwritten.

**Linux example:**

```ini
archive_command = 'test ! -f /mnt/archive/%f && cp %p /mnt/archive/%f'
```

Breaking down the Linux command:
- `test ! -f /mnt/archive/%f`: Checks that the file doesn't already exist in the archive
- `&&`: Only execute the copy if the test succeeds
- `cp %p /mnt/archive/%f`: Copies the WAL file to the archive location

**Windows example:**

```ini
archive_command = 'IF NOT EXIST "C:\\archive\\%f" copy "%p" "C:\\archive\\%f"'
```

Breaking down the Windows command:
- `IF NOT EXIST "C:\\archive\\%f"`: Checks that the file doesn't already exist in the archive
- `copy "%p" "C:\\archive\\%f"`: Copies the WAL file to the archive location
- Note: Backslashes must be escaped in postgresql.conf (use `\\` instead of `\`)

**Alternative Windows example using PowerShell:**

```ini
archive_command = 'powershell -Command "if (!(Test-Path C:\\archive\\%f)) { Copy-Item ''%p'' ''C:\\archive\\%f'' }"'
```

#### Additional configuration for production

For production systems, consider these additional parameters:

```ini
# Maximum size of WAL segments to keep in pg_wal directory
max_wal_size = 2GB

# Minimum size to keep even if checkpoint is triggered
min_wal_size = 1GB

# How many WAL segments to keep for standby servers
# (If not using replication slots)
wal_keep_size = 5GB

# Archive timeout - force switch even if segment not full
archive_timeout = 300  # 5 minutes
```

The archive_timeout parameter is particularly important. Without it, a low-activity database might not switch WAL files for hours, meaning your most recent recoverable point could be very old. Setting this to 5-15 minutes provides a good balance.

### Setting up the archive directory

Before enabling archiving, prepare your archive directory:

**Linux:**

```bash
# Create the archive directory
sudo mkdir -p /mnt/archive

# Set ownership to the postgres user
sudo chown postgres:postgres /mnt/archive

# Set appropriate permissions (owner read/write/execute only)
sudo chmod 700 /mnt/archive

# Verify setup
ls -ld /mnt/archive
```

Example output:

```
drwx------ 2 postgres postgres 4096 Nov  4 10:30 /mnt/archive
```

**Windows:**

```cmd
REM Create the archive directory
mkdir C:\archive

REM Set permissions (using icacls)
icacls C:\archive /inheritance:r
icacls C:\archive /grant:r "NT AUTHORITY\NetworkService:(OI)(CI)F"
icacls C:\archive /remove:g Users

REM Verify setup
icacls C:\archive
```

Example output:

```
C:\archive NT AUTHORITY\NetworkService:(OI)(CI)F
```

**Note for Windows:** If PostgreSQL runs as a different service account, replace `NT AUTHORITY\NetworkService` with the appropriate account name. You can check the service account in Services (services.msc) under the PostgreSQL service properties.

### Advanced archive command examples

Here are more sophisticated archive_command examples for different scenarios:

**Archiving to a remote server:**

Linux (via SSH):
```ini
archive_command = 'rsync -a %p postgres@backup-server:/archive/%f'
```

Windows (to network share):
```ini
archive_command = 'copy "%p" "\\\\backup-server\\archive\\%f"'
```

**Archiving with compression:**

Linux:
```ini
archive_command = 'gzip < %p > /mnt/archive/%f.gz'
```

Windows (using 7-Zip, if installed):
```ini
archive_command = '"C:\\Program Files\\7-Zip\\7z.exe" a -tgzip "C:\\archive\\%f.gz" "%p" > nul'
```

Windows (using PowerShell):
```ini
archive_command = 'powershell -Command "Compress-Archive -Path ''%p'' -DestinationPath ''C:\\archive\\%f.zip''"'
```

**Archiving to AWS S3:**

Linux:
```ini
archive_command = 'aws s3 cp %p s3://my-backup-bucket/wal-archive/%f'
```

Windows:
```ini
archive_command = 'aws s3 cp "%p" s3://my-backup-bucket/wal-archive/%f'
```

**Archiving with multiple destinations (redundancy):**

Linux:
```ini
archive_command = 'test ! -f /mnt/archive/%f && cp %p /mnt/archive/%f && rsync -a %p backup-server:/archive/%f'
```

Windows:
```ini
archive_command = 'IF NOT EXIST "C:\\archive\\%f" (copy "%p" "C:\\archive\\%f" && copy "%p" "\\\\backup-server\\archive\\%f")'
```

**Using pgBackRest (recommended for production):**

Linux:
```ini
archive_command = 'pgbackrest --stanza=main archive-push %p'
```

Windows:
```ini
archive_command = 'pgbackrest --stanza=main archive-push "%p"'
```

### Configuring archive cleanup

Over time, your archive will grow continuously. You need a strategy to clean up old WAL files that are no longer needed. However, be careful: deleting WAL files that are still needed will make PITR impossible for certain time ranges.

A safe approach is to keep WAL files for a specific retention period, for example, 30 days:

**Linux cleanup script:**

```bash
# Create a cleanup script
cat > /usr/local/bin/cleanup-archive.sh << 'EOF'
#!/bin/bash
ARCHIVE_DIR=/mnt/archive
RETENTION_DAYS=30

# Find and delete WAL files older than retention period
find "$ARCHIVE_DIR" -name "*.gz" -type f -mtime +$RETENTION_DAYS -delete
find "$ARCHIVE_DIR" -type f ! -name "*.history" -mtime +$RETENTION_DAYS -delete

# Log the cleanup
echo "$(date): Cleaned up archive files older than $RETENTION_DAYS days"
EOF

chmod +x /usr/local/bin/cleanup-archive.sh
```

Schedule this script with cron:

```bash
# Run cleanup daily at 2 AM
crontab -e
0 2 * * * /usr/local/bin/cleanup-archive.sh >> /var/log/archive-cleanup.log 2>&1
```

**Windows cleanup script:**

```powershell
# Create a cleanup script (save as C:\Scripts\cleanup-archive.ps1)
$archiveDir = "C:\archive"
$retentionDays = 30
$cutoffDate = (Get-Date).AddDays(-$retentionDays)

# Find and delete WAL files older than retention period
Get-ChildItem -Path $archiveDir -File |
    Where-Object { $_.Name -notlike "*.history" -and $_.LastWriteTime -lt $cutoffDate } |
    Remove-Item -Force

# Log the cleanup
$logMessage = "$(Get-Date): Cleaned up archive files older than $retentionDays days"
Add-Content -Path "C:\Logs\archive-cleanup.log" -Value $logMessage
```

Schedule this script with Task Scheduler:

```cmd
REM Create a scheduled task to run daily at 2 AM
schtasks /create /tn "PostgreSQL Archive Cleanup" /tr "powershell.exe -ExecutionPolicy Bypass -File C:\Scripts\cleanup-archive.ps1" /sc daily /st 02:00 /ru SYSTEM
```

Or use PowerShell to create the scheduled task:

```powershell
$action = New-ScheduledTaskAction -Execute 'powershell.exe' -Argument '-ExecutionPolicy Bypass -File C:\Scripts\cleanup-archive.ps1'
$trigger = New-ScheduledTaskTrigger -Daily -At 2am
$principal = New-ScheduledTaskPrincipal -UserId "SYSTEM" -LogonType ServiceAccount -RunLevel Highest
Register-ScheduledTask -TaskName "PostgreSQL Archive Cleanup" -Action $action -Trigger $trigger -Principal $principal
```

**Important**: If you're using this archive for multiple backup purposes (like maintaining replicas), ensure your cleanup policy accounts for the needs of all consumers.

### Verifying archive configuration

After configuring archiving, verify that it's working correctly:

1. Reload the configuration:

```sql
SELECT pg_reload_conf();
```

2. Check the current archive settings:

```sql
SELECT name, setting, unit, context 
FROM pg_settings 
WHERE name IN ('wal_level', 'archive_mode', 'archive_command');
```

Example output:

```
      name       | setting                                           | unit | context
-----------------+---------------------------------------------------+------+-----------
 archive_command | test ! -f /mnt/archive/%f && cp %p /mnt/archive/%f |      | sighup
 archive_mode    | on                                                 |      | postmaster
 wal_level       | replica                                            |      | postmaster
```

3. Force a WAL switch to test archiving:

```sql
SELECT pg_switch_wal();
```

4. Check if the file was archived:

**Linux:**
```bash
ls -lh /mnt/archive/
```

**Windows:**
```cmd
dir C:\archive\
```

Or using PowerShell:
```powershell
Get-ChildItem C:\archive\ | Format-Table Name, Length, LastWriteTime
```

You should see at least one WAL file (16 MB each by default).

5. Monitor archive status:

```sql
SELECT archived_count, 
       failed_count, 
       last_archived_wal,
       last_archived_time,
       last_failed_wal,
       last_failed_time
FROM pg_stat_archiver;
```

Example output:

```
 archived_count | failed_count |    last_archived_wal     |     last_archived_time     | last_failed_wal | last_failed_time
----------------+--------------+--------------------------+----------------------------+-----------------+------------------
             12 |            0 | 000000010000000000000004 | 2025-11-04 10:35:27.123456 |                 |
```

If failed_count is increasing, check the PostgreSQL log file for error messages related to the archive_command.

**Checking PostgreSQL logs:**

Linux:
```bash
tail -f /var/log/postgresql/postgresql-18-main.log
# or
journalctl -u postgresql -f
```

Windows:
```cmd
REM Using PowerShell to tail the log
powershell -Command "Get-Content 'C:\Program Files\PostgreSQL\18\data\log\postgresql-*.log' -Wait -Tail 50"
```

Or check Windows Event Viewer:
```cmd
eventvwr.msc
REM Navigate to: Windows Logs > Application
REM Filter for Source: postgresql
```

### Testing archive recovery

Before relying on PITR in production, test the entire process:

```sql
-- Create a test database
CREATE DATABASE pitr_test;

-- Connect to it
\c pitr_test

-- Create and populate a test table
CREATE TABLE test_data (
    id SERIAL PRIMARY KEY,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    data TEXT
);

INSERT INTO test_data (data) 
SELECT 'Test record ' || generate_series(1, 10000);

-- Note the current time
SELECT now() AS recovery_target;
```

Take note of the timestamp returned. This will be your recovery target point. Later in the chapter, we'll use this to perform a test recovery.

## Performing PITR operations

Now that WAL archiving is configured and tested, let's explore how to actually perform Point-In-Time Recovery. This section covers the complete PITR process, from taking a base backup through recovery to a specific point in time.

### Taking a base backup for PITR

A base backup is the starting point for PITR. You need a physical backup taken while the database is running and archiving is active. The most straightforward way to take a base backup is using pg_basebackup:

**Linux:**

```bash
# Create a directory for the backup
mkdir -p /mnt/backups/base

# Take the base backup
pg_basebackup -h localhost -U postgres -D /mnt/backups/base/$(date +%Y%m%d-%H%M%S) \
              -Ft -z -P -X stream
```

**Windows:**

```cmd
REM Create a directory for the backup
mkdir C:\backups\base

REM Take the base backup (using Command Prompt date/time format)
pg_basebackup -h localhost -U postgres -D "C:\backups\base\%date:~-4,4%%date:~-10,2%%date:~-7,2%-%time:~0,2%%time:~3,2%%time:~6,2%" -Ft -z -P -X stream
```

Or using PowerShell (cleaner date format):

```powershell
# Create a directory for the backup
New-Item -ItemType Directory -Force -Path C:\backups\base

# Take the base backup
$timestamp = Get-Date -Format "yyyyMMdd-HHmmss"
pg_basebackup -h localhost -U postgres -D "C:\backups\base\$timestamp" -Ft -z -P -X stream
```

Let's break down these options:

- `-D`: Destination directory for the backup
- `-Ft`: Format as tar (easier to manage and transfer)
- `-z`: Compress with gzip
- `-P`: Show progress
- `-X stream`: Include WAL files needed to make the backup consistent

Example output:

```
24567/24567 kB (100%), 1/1 tablespace
```

After completion, you'll have tar files in your backup directory:

**Linux:**
```bash
ls -lh /mnt/backups/base/20251104-103000/
```

**Windows:**
```cmd
dir C:\backups\base\20251104-103000\
```

Output:

```
-rw------- 1 postgres postgres  45M Nov  4 10:32 base.tar.gz
-rw------- 1 postgres postgres  16M Nov  4 10:32 pg_wal.tar.gz
```

**Important**: Label your base backups with clear timestamps and keep notes about when they were taken. You cannot perform PITR to a point before your base backup was started.

### Understanding the recovery process

The PITR recovery process follows these steps:

1. Restore the base backup to a new location or to the original PGDATA
2. Create a recovery configuration file or use recovery parameters
3. Start PostgreSQL in recovery mode
4. PostgreSQL applies WAL from the backup, then from the archive
5. When the recovery target is reached, PostgreSQL stops applying WAL
6. PostgreSQL performs final consistency checks
7. The database becomes available for connections (if promoted)

During recovery, PostgreSQL creates a standby.signal file or checks for recovery.signal to know it should enter recovery mode. In PostgreSQL 18, this behavior is well-defined and simpler than in earlier versions.

### Performing a basic PITR

Let's walk through a complete PITR scenario. Assume you need to recover to a specific point in time after some unintended changes were made.

**Step 1: Stop the current PostgreSQL instance**

First, stop the running database:

**Linux:**
```bash
# Using pg_ctl
pg_ctl -D /var/lib/postgresql/18/main stop -m fast

# Or using systemd
sudo systemctl stop postgresql
```

**Windows:**
```cmd
REM Stop the service using net stop
net stop postgresql-x64-18

REM Or using PowerShell
Stop-Service postgresql-x64-18
```

Or use Windows Services GUI (services.msc).

**Step 2: Backup the current PGDATA (optional but recommended)**

Before restoring, it's wise to preserve the current state:

**Linux:**
```bash
sudo mv /var/lib/postgresql/18/main /var/lib/postgresql/18/main.old
```

**Windows:**
```cmd
REM Rename the data directory
move "C:\Program Files\PostgreSQL\18\data" "C:\Program Files\PostgreSQL\18\data.old"
```

Or using PowerShell:
```powershell
Move-Item "C:\Program Files\PostgreSQL\18\data" "C:\Program Files\PostgreSQL\18\data.old"
```

**Step 3: Restore the base backup**

Extract the base backup to the PGDATA directory:

**Linux:**
```bash
# Create new PGDATA directory
sudo mkdir -p /var/lib/postgresql/18/main

# Extract base backup
sudo -u postgres tar -xzf /mnt/backups/base/20251104-103000/base.tar.gz \
     -C /var/lib/postgresql/18/main

# Extract WAL from the backup
sudo -u postgres tar -xzf /mnt/backups/base/20251104-103000/pg_wal.tar.gz \
     -C /var/lib/postgresql/18/main/pg_wal
```

**Windows (Command Prompt):**
```cmd
REM Create new data directory
mkdir "C:\Program Files\PostgreSQL\18\data"

REM Extract base backup using tar (included with Windows 10+)
cd "C:\Program Files\PostgreSQL\18\data"
tar -xzf "C:\backups\base\20251104-103000\base.tar.gz"

REM Extract WAL
cd pg_wal
tar -xzf "C:\backups\base\20251104-103000\pg_wal.tar.gz"
```

**Windows (PowerShell with 7-Zip):**
```powershell
# Create new data directory
New-Item -ItemType Directory -Force -Path "C:\Program Files\PostgreSQL\18\data"

# Extract using 7-Zip
& "C:\Program Files\7-Zip\7z.exe" x "C:\backups\base\20251104-103000\base.tar.gz" -o"C:\Program Files\PostgreSQL\18\data\"
& "C:\Program Files\7-Zip\7z.exe" x "C:\backups\base\20251104-103000\pg_wal.tar.gz" -o"C:\Program Files\PostgreSQL\18\data\pg_wal\"
```

**Step 4: Configure recovery parameters**

In PostgreSQL 18, recovery configuration is done through postgresql.conf or by creating a recovery.signal file with recovery parameters.

**Linux:**
```bash
sudo -u postgres cat >> /var/lib/postgresql/18/main/postgresql.conf << EOF

# Recovery configuration for PITR
restore_command = 'cp /mnt/archive/%f %p'
recovery_target_time = '2025-11-04 14:30:00'
recovery_target_action = 'promote'
EOF
```

**Windows (Command Prompt):**
```cmd
REM Append to postgresql.conf
echo. >> "C:\Program Files\PostgreSQL\18\data\postgresql.conf"
echo # Recovery configuration for PITR >> "C:\Program Files\PostgreSQL\18\data\postgresql.conf"
echo restore_command = 'copy "C:\\archive\\%%f" "%%p"' >> "C:\Program Files\PostgreSQL\18\data\postgresql.conf"
echo recovery_target_time = '2025-11-04 14:30:00' >> "C:\Program Files\PostgreSQL\18\data\postgresql.conf"
echo recovery_target_action = 'promote' >> "C:\Program Files\PostgreSQL\18\data\postgresql.conf"
```

**Windows (PowerShell - cleaner approach):**
```powershell
$configFile = "C:\Program Files\PostgreSQL\18\data\postgresql.conf"
@"

# Recovery configuration for PITR
restore_command = 'copy "C:\\archive\\%f" "%p"'
recovery_target_time = '2025-11-04 14:30:00'
recovery_target_action = 'promote'
"@ | Add-Content -Path $configFile
```

Create the recovery.signal file to tell PostgreSQL to enter recovery mode:

**Linux:**
```bash
sudo -u postgres touch /var/lib/postgresql/18/main/recovery.signal
```

**Windows:**
```cmd
REM Create empty recovery.signal file
type nul > "C:\Program Files\PostgreSQL\18\data\recovery.signal"
```

Or using PowerShell:
```powershell
New-Item -ItemType File -Path "C:\Program Files\PostgreSQL\18\data\recovery.signal" -Force
```

**Step 5: Start PostgreSQL in recovery mode**

Start the database:

**Linux:**
```bash
# Using systemd
sudo systemctl start postgresql

# Or using pg_ctl
pg_ctl -D /var/lib/postgresql/18/main start
```

**Windows:**
```cmd
REM Start the service
net start postgresql-x64-18

REM Or using PowerShell
Start-Service postgresql-x64-18
```

**Step 6: Monitor the recovery process**

Watch the PostgreSQL log file to monitor recovery progress:

**Linux:**
```bash
tail -f /var/log/postgresql/postgresql-18-main.log

# Or using journalctl
journalctl -u postgresql -f
```

**Windows (Command Prompt):**
```cmd
REM Find the latest log file and display it
powershell -Command "Get-Content 'C:\Program Files\PostgreSQL\18\data\log\postgresql-*.log' -Wait -Tail 50"
```

**Windows (PowerShell):**
```powershell
# Get the most recent log file
$logFile = Get-ChildItem "C:\Program Files\PostgreSQL\18\data\log\postgresql-*.log" | 
           Sort-Object LastWriteTime -Descending | 
           Select-Object -First 1

# Tail the log
Get-Content $logFile.FullName -Wait -Tail 50
```

You'll see messages like:

```
2025-11-04 14:25:00.123 UTC [1234] LOG:  starting point-in-time recovery to 2025-11-04 14:30:00+00
2025-11-04 14:25:01.456 UTC [1234] LOG:  restored log file "000000010000000000000001" from archive
2025-11-04 14:25:02.789 UTC [1234] LOG:  restored log file "000000010000000000000002" from archive
...
2025-11-04 14:25:30.123 UTC [1234] LOG:  recovery stopping before commit of transaction 12345, time 2025-11-04 14:30:15.123456+00
2025-11-04 14:25:30.234 UTC [1234] LOG:  recovery has paused
2025-11-04 14:25:30.345 UTC [1234] LOG:  selected new timeline ID: 2
2025-11-04 14:25:30.456 UTC [1234] LOG:  archive recovery complete
2025-11-04 14:25:31.567 UTC [1234] LOG:  database system is ready to accept connections
```

**Step 7: Verify the recovery**

Once recovery is complete, verify the database state:

```sql
-- Check the current timestamp
SELECT now();

-- Verify that your data is in the expected state
-- This depends on your specific recovery scenario
SELECT count(*) FROM your_critical_table;

-- Check recovery status
SELECT pg_is_in_recovery();
```

If pg_is_in_recovery() returns false, the recovery is complete and the database has been promoted to a normal running state.

### Recovery target actions

The recovery_target_action parameter controls what happens when the recovery target is reached. PostgreSQL 18 supports three options:

**pause**: Pause recovery at the target point. This allows you to inspect the database state before deciding whether to promote:

```ini
recovery_target_action = 'pause'
```

When paused, you can query the database (read-only) to verify the state, then either:

```sql
-- Promote to normal operation
SELECT pg_wal_replay_resume();
```

Or shut down and try a different recovery target.

**promote**: Automatically end recovery and promote the server to normal operation:

```ini
recovery_target_action = 'promote'
```

This is the most common setting for PITR.

**shutdown**: Shut down the server when the target is reached:

```ini
recovery_target_action = 'shutdown'
```

This is useful when you want to inspect the recovered files before starting the server normally.

### Recovery target inclusivity

By default, recovery stops just before the specified target (the target is excluded). You can control this with recovery_target_inclusive:

```ini
# Stop just before the target (default)
recovery_target_inclusive = false

# Include the target transaction/timestamp
recovery_target_inclusive = true
```

For example, if you know that transaction 12345 was the bad transaction that dropped a table, you would set:

```ini
recovery_target_xid = '12345'
recovery_target_inclusive = false  # Stop before executing this transaction
```

## Advanced PITR techniques and recovery targets

Beyond basic time-based recovery, PostgreSQL 18 offers sophisticated PITR capabilities. This section explores advanced techniques that provide greater control and flexibility in recovery scenarios.

### Using named restore points

Named restore points allow you to create logical markers in the WAL stream during normal database operation. These markers can be used as recovery targets, which is especially useful before risky operations.

**Creating restore points:**

```sql
-- Before performing a major update
SELECT pg_create_restore_point('before_version_3_upgrade');
```

The function returns the LSN (Log Sequence Number) where the restore point was created:

```
 pg_create_restore_point
-------------------------
 0/3000140
(1 row)
```

**Best practices for restore points:**

Create restore points:
- Before major application upgrades
- Before bulk data modifications
- Before schema changes
- At the end of each business day
- Before running potentially destructive operations

**Using restore points for recovery:**

```ini
restore_command = 'cp /mnt/archive/%f %p'
recovery_target_name = 'before_version_3_upgrade'
recovery_target_action = 'promote'
```

**Listing restore points:**

Unfortunately, PostgreSQL doesn't provide a built-in way to list all restore points. However, you can maintain a log:

```sql
CREATE TABLE restore_point_log (
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    restore_point_name TEXT,
    lsn PG_LSN,
    notes TEXT
);

-- When creating a restore point
INSERT INTO restore_point_log (restore_point_name, lsn, notes)
VALUES ('before_version_3_upgrade', 
        pg_create_restore_point('before_version_3_upgrade'),
        'Major version upgrade from 3.0 to 3.5');
```

### Transaction ID-based recovery

When you need precise control, recovering to a specific transaction ID provides the most accuracy. This is particularly useful when you know the exact transaction that caused a problem.

**Finding problematic transactions:**

```sql
-- Enable tracking of transaction IDs (if not already enabled)
ALTER SYSTEM SET track_commit_timestamp = on;
SELECT pg_reload_conf();

-- Query to find transactions in a time range
SELECT transaction, 
       timestamp 
FROM pg_xact_commit_timestamp('12340', '12350');
```

**Recovering to a specific XID:**

```ini
restore_command = 'cp /mnt/archive/%f %p'
recovery_target_xid = '12349'
recovery_target_inclusive = false
recovery_target_action = 'promote'
```

**Important note**: Transaction IDs wrap around approximately every 2 billion transactions. Always combine XID-based recovery with timestamp verification.

### LSN-based recovery

Log Sequence Numbers provide the most granular recovery target. LSNs represent specific positions in the WAL stream:

```ini
restore_command = 'cp /mnt/archive/%f %p'
recovery_target_lsn = '0/3000000'
recovery_target_action = 'promote'
```

**Finding specific LSNs:**

```sql
-- Current WAL LSN
SELECT pg_current_wal_lsn();

-- LSN of a specific backup
SELECT backup_start_lsn 
FROM pg_get_wal_replay_pause_state();

-- Calculating LSN differences
SELECT pg_wal_lsn_diff('0/4000000', '0/3000000') AS bytes_difference;
```

LSN-based recovery is most useful in replication scenarios or when coordinating recovery across multiple databases.

### Timeline management

Timelines represent different histories of the WAL stream. Every time you perform PITR and promote a server, a new timeline is created.

**Understanding timeline files:**

After recovery, you'll see .history files in pg_wal:

```bash
ls -l /var/lib/postgresql/18/main/pg_wal/*.history
```

Output:

```
-rw------- 1 postgres postgres 42 Nov  4 14:30 00000002.history
```

**Contents of a timeline history file:**

```bash
cat /var/lib/postgresql/18/main/pg_wal/00000002.history
```

Output:

```
1	0/3000000	before 2025-11-04 14:30:00.123456+00
```

This indicates:
- Previous timeline: 1
- LSN where timeline diverged: 0/3000000
- Reason for divergence: recovery to the specified time

**Recovering along non-current timelines:**

By default, recovery follows the current timeline. To follow a different timeline:

```ini
restore_command = 'cp /mnt/archive/%f %p'
recovery_target_timeline = '2'
recovery_target_time = '2025-11-04 15:00:00'
```

**Timeline divergence scenarios:**

Timeline management becomes critical in complex scenarios:

1. **Re-PITR from the same base backup**: If you perform PITR, then realize you need to go back to an earlier point, you'll be working with multiple timelines.

2. **Standby promotion scenarios**: When a standby is promoted to primary, it creates a new timeline. If you later need to restore the original primary, timeline management is essential.

3. **Point-in-time cloning**: Creating multiple test environments from different recovery points of the same base backup results in separate timelines.

### Partial database recovery

Sometimes you don't need to recover the entire database, just specific tables or schemas. While PITR operates on the entire cluster, you can extract specific data:

**Approach 1: Recover to a temporary location**

1. Perform PITR to a secondary location:

```bash
# Restore to temporary directory
pg_basebackup -h localhost -U postgres -D /tmp/recovery-temp
```

2. Configure recovery in the temporary location

3. Start the recovered instance on a different port:

```ini
# In /tmp/recovery-temp/postgresql.conf
port = 5433
```

4. Extract the needed data:

```bash
pg_dump -h localhost -p 5433 -U postgres -t specific_table database_name | \
pg_restore -h localhost -p 5432 -U postgres -d database_name
```

**Approach 2: Use logical replication**

After PITR, use logical replication to selectively copy data:

```sql
-- On recovered database (source)
CREATE PUBLICATION recovery_pub FOR TABLE specific_table;

-- On target database
CREATE SUBSCRIPTION recovery_sub 
CONNECTION 'host=recovery-server dbname=database_name' 
PUBLICATION recovery_pub;
```

### Parallel recovery

PostgreSQL 18 supports parallel recovery, which can significantly speed up PITR for large databases:

```ini
# Enable parallel recovery (default in PostgreSQL 18)
recovery_prefetch = on

# Number of parallel workers for recovery (default: 0 = auto)
max_recovery_parallelism = 4
```

With parallel recovery enabled:
- WAL records are prefetched and applied in parallel
- Recovery time can be reduced by 30-50% for large databases
- CPU and I/O usage will increase during recovery

**Monitoring parallel recovery:**

```sql
SELECT pid,
       backend_type,
       state,
       wait_event_type,
       wait_event
FROM pg_stat_activity
WHERE backend_type LIKE '%wal receiver%' OR backend_type LIKE '%startup%';
```

### Recovery conflict management

During recovery, conflicts can arise, especially if you're recovering a database that had active connections at the time of backup. Configure how PostgreSQL handles these conflicts:

```ini
# Maximum time to wait before canceling conflicting queries (in ms)
max_standby_streaming_delay = 30s

# For archive recovery
max_standby_archive_delay = 300s

# Kill conflicting connections instead of canceling queries
hot_standby_feedback = off
```

**Understanding recovery conflicts:**

Conflicts occur when:
- Recovery needs to apply a WAL record that conflicts with a running query
- A query holds locks on objects that need to be modified during recovery
- Vacuum records need to be applied but queries are reading old row versions

**Monitoring conflicts:**

```sql
SELECT datname,
       confl_tablespace,
       confl_lock,
       confl_snapshot,
       confl_bufferpin,
       confl_deadlock
FROM pg_stat_database_conflicts;
```

## Monitoring and troubleshooting PITR

Successful PITR implementations require robust monitoring and the ability to diagnose and resolve issues quickly. This section covers essential monitoring techniques and common troubleshooting scenarios.

### Monitoring archive status

The pg_stat_archiver view provides real-time information about archiving:

```sql
SELECT * FROM pg_stat_archiver;
```

Example output:

```
-[ RECORD 1 ]------+------------------------------
archived_count     | 245
last_archived_wal  | 000000010000000000000034
last_archived_time | 2025-11-04 14:45:23.123456+00
failed_count       | 3
last_failed_wal    | 000000010000000000000031
last_failed_time   | 2025-11-04 14:40:15.789012+00
stats_reset        | 2025-11-04 00:00:00+00
```

**Key metrics to monitor:**

- **failed_count**: Should remain at zero or very low. Increasing failures indicate archiving problems
- **archived_count**: Should increase steadily with database activity
- **Time since last_archived_time**: Shouldn't exceed your archive_timeout plus a few seconds

**Creating monitoring alerts:**

```sql
-- Alert if archiving hasn't happened in 10 minutes
SELECT CASE 
    WHEN EXTRACT(EPOCH FROM (now() - last_archived_time)) > 600 
    THEN 'ALERT: No archiving in last 10 minutes'
    ELSE 'OK'
END AS archive_status,
last_archived_time,
archived_count,
failed_count
FROM pg_stat_archiver;
```

### Monitoring WAL generation rate

Understanding your WAL generation rate helps with capacity planning:

```sql
-- Current WAL position
SELECT pg_current_wal_lsn();

-- Wait 60 seconds, then check again
SELECT pg_sleep(60);

-- Calculate WAL generation rate
SELECT 
    pg_wal_lsn_diff(pg_current_wal_lsn(), '0/3000000') AS bytes_generated,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), '0/3000000')) AS human_readable;
```

**Tracking WAL generation over time:**

```sql
CREATE TABLE wal_generation_log (
    logged_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    current_lsn PG_LSN,
    bytes_since_last_check BIGINT,
    wal_files_since_last_check INTEGER
);

-- Function to log WAL generation
CREATE OR REPLACE FUNCTION log_wal_generation()
RETURNS void AS $$
DECLARE
    current_lsn PG_LSN;
    last_lsn PG_LSN;
    bytes_diff BIGINT;
BEGIN
    current_lsn := pg_current_wal_lsn();
    
    SELECT current_lsn INTO last_lsn 
    FROM wal_generation_log 
    ORDER BY logged_at DESC 
    LIMIT 1;
    
    IF last_lsn IS NOT NULL THEN
        bytes_diff := pg_wal_lsn_diff(current_lsn, last_lsn);
    ELSE
        bytes_diff := 0;
    END IF;
    
    INSERT INTO wal_generation_log (current_lsn, bytes_since_last_check)
    VALUES (current_lsn, bytes_diff);
END;
$$ LANGUAGE plpgsql;

-- Schedule with pg_cron or external scheduler
SELECT log_wal_generation();
```

### Verifying backup and archive integrity

Regular verification ensures your backups and archives are usable when needed:

**Verify individual WAL files:**

**Linux:**
```bash
# Check WAL file integrity
pg_waldump /mnt/archive/000000010000000000000001 > /dev/null

# If successful, no output
# If corrupted, you'll see error messages
```

**Windows:**
```cmd
REM Check WAL file integrity
pg_waldump "C:\archive\000000010000000000000001" > nul

REM If successful, no output
REM If corrupted, you'll see error messages
```

**Verify base backup:**

PostgreSQL 18 includes pg_verifybackup:

**Linux:**
```bash
pg_verifybackup /mnt/backups/base/20251104-103000
```

**Windows:**
```cmd
pg_verifybackup "C:\backups\base\20251104-103000"
```

Example output:

```
pg_verifybackup: backup successfully verified
```

**Automated backup testing:**

Create a script that regularly tests PITR:

**Linux test script:**

```bash
#!/bin/bash
# test-pitr.sh

BACKUP_DIR="/mnt/backups/base/latest"
ARCHIVE_DIR="/mnt/archive"
TEST_DIR="/tmp/pitr-test-$$"
LOG_FILE="/var/log/pitr-test.log"

echo "$(date): Starting PITR test" >> "$LOG_FILE"

# Create test directory
mkdir -p "$TEST_DIR"

# Restore base backup
tar -xzf "$BACKUP_DIR/base.tar.gz" -C "$TEST_DIR"

# Configure recovery
cat >> "$TEST_DIR/postgresql.conf" << EOF
port = 5444
restore_command = 'cp $ARCHIVE_DIR/%f %p'
recovery_target = 'immediate'
recovery_target_action = 'shutdown'
EOF

touch "$TEST_DIR/recovery.signal"

# Start PostgreSQL for recovery
pg_ctl -D "$TEST_DIR" -l "$TEST_DIR/logfile" start

# Wait for recovery to complete
sleep 30

# Check if recovery was successful
if grep -q "database system is shut down" "$TEST_DIR/logfile"; then
    echo "$(date): PITR test PASSED" >> "$LOG_FILE"
    RESULT=0
else
    echo "$(date): PITR test FAILED" >> "$LOG_FILE"
    RESULT=1
fi

# Cleanup
rm -rf "$TEST_DIR"

exit $RESULT
```

Make it executable and schedule:

```bash
chmod +x /usr/local/bin/test-pitr.sh

# Add to crontab - run weekly on Sundays at 2 AM
0 2 * * 0 /usr/local/bin/test-pitr.sh
```

**Windows test script (PowerShell):**

```powershell
# test-pitr.ps1

$backupDir = "C:\backups\base\latest"
$archiveDir = "C:\archive"
$testDir = "C:\temp\pitr-test-$(Get-Date -Format 'yyyyMMddHHmmss')"
$logFile = "C:\Logs\pitr-test.log"

Add-Content -Path $logFile -Value "$(Get-Date): Starting PITR test"

# Create test directory
New-Item -ItemType Directory -Force -Path $testDir | Out-Null

# Restore base backup
tar -xzf "$backupDir\base.tar.gz" -C $testDir

# Configure recovery
$config = @"
port = 5444
restore_command = 'copy "$archiveDir\%f" "%p"'
recovery_target = 'immediate'
recovery_target_action = 'shutdown'
"@
Add-Content -Path "$testDir\postgresql.conf" -Value $config

New-Item -ItemType File -Path "$testDir\recovery.signal" | Out-Null

# Start PostgreSQL for recovery
& pg_ctl -D $testDir -l "$testDir\logfile.txt" start

# Wait for recovery to complete
Start-Sleep -Seconds 30

# Check if recovery was successful
$logContent = Get-Content "$testDir\logfile.txt" -Raw
if ($logContent -match "database system is shut down") {
    Add-Content -Path $logFile -Value "$(Get-Date): PITR test PASSED"
    $result = 0
} else {
    Add-Content -Path $logFile -Value "$(Get-Date): PITR test FAILED"
    $result = 1
}

# Cleanup
Remove-Item -Recurse -Force $testDir

exit $result
```

Schedule with Task Scheduler:

```powershell
# Create scheduled task to run weekly on Sundays at 2 AM
$action = New-ScheduledTaskAction -Execute 'powershell.exe' -Argument '-ExecutionPolicy Bypass -File C:\Scripts\test-pitr.ps1'
$trigger = New-ScheduledTaskTrigger -Weekly -DaysOfWeek Sunday -At 2am
$principal = New-ScheduledTaskPrincipal -UserId "SYSTEM" -LogonType ServiceAccount -RunLevel Highest
Register-ScheduledTask -TaskName "PostgreSQL PITR Test" -Action $action -Trigger $trigger -Principal $principal
```

### Common issues and solutions

**Issue 1: Archive command failing**

Symptoms:
- failed_count increasing in pg_stat_archiver
- WAL files accumulating in pg_wal directory
- Errors in PostgreSQL log

Diagnosis:

```sql
SELECT last_failed_wal, 
       last_failed_time, 
       failed_count 
FROM pg_stat_archiver;
```

Check PostgreSQL log:

**Linux:**
```bash
grep "archive command failed" /var/log/postgresql/postgresql-18-main.log
```

**Windows:**
```powershell
Select-String -Path "C:\Program Files\PostgreSQL\18\data\log\postgresql-*.log" -Pattern "archive command failed"
```

Common causes and solutions:

1. **Insufficient disk space on archive destination:**

**Linux:**
```bash
df -h /mnt/archive
```

**Windows:**
```powershell
Get-PSDrive C | Select-Object Used,Free
# Or
Get-Volume | Where-Object {$_.DriveLetter -eq 'C'}
```

Solution: Free up space or expand storage

2. **Permission issues:**

**Linux:**
```bash
sudo -u postgres test -w /mnt/archive && echo "Writable" || echo "Not writable"
```

**Windows:**
```powershell
# Test write permissions
$testFile = "C:\archive\test_write_$(Get-Date -Format 'yyyyMMddHHmmss').tmp"
try {
    [System.IO.File]::WriteAllText($testFile, "test")
    Remove-Item $testFile
    Write-Host "Writable"
} catch {
    Write-Host "Not writable: $_"
}
```

Solution: Fix permissions:

**Linux:**
```bash
sudo chown -R postgres:postgres /mnt/archive
sudo chmod 700 /mnt/archive
```

**Windows:**
```cmd
REM Grant full control to the PostgreSQL service account
icacls C:\archive /grant "NT AUTHORITY\NetworkService:(OI)(CI)F"
```

3. **Archive command syntax error:**

Test the command manually:

**Linux:**
```bash
sudo -u postgres /bin/sh -c 'cp /var/lib/postgresql/18/main/pg_wal/000000010000000000000001 /mnt/archive/test_archive'
```

**Windows:**
```cmd
REM Test as the service account (may need to run as that user)
copy "C:\Program Files\PostgreSQL\18\data\pg_wal\000000010000000000000001" "C:\archive\test_archive"
```

4. **Network issues (for remote archiving):**

Test connectivity:

**Linux:**
```bash
sudo -u postgres ssh backup-server echo "Connection OK"
```

**Windows:**
```powershell
# Test network share access
Test-Path "\\backup-server\archive"
```

**Issue 2: WAL files not found during recovery**

Symptoms:
- Recovery stops with "could not open file" errors
- Missing WAL segment messages in log

Diagnosis:

**Linux:**
```bash
# Check which WAL file is needed
grep "could not open file" /var/log/postgresql/postgresql-18-main.log

# Verify it exists in archive
ls -l /mnt/archive/000000010000000000000005
```

**Windows:**
```powershell
# Check which WAL file is needed
Select-String -Path "C:\Program Files\PostgreSQL\18\data\log\postgresql-*.log" -Pattern "could not open file"

# Verify it exists in archive
Get-Item "C:\archive\000000010000000000000005"
```

Solutions:

1. **WAL file actually missing (archive gap):**

This is serious. You cannot perform PITR past this point. Options:
- Recover to before the gap
- Accept data loss and force recovery past the gap (not recommended)

2. **restore_command incorrect:**

Test restore command manually:

**Linux:**
```bash
# Simulate what PostgreSQL does
sudo -u postgres sh -c 'cp /mnt/archive/000000010000000000000005 /tmp/test_restore'
```

**Windows:**
```cmd
REM Simulate the restore
copy "C:\archive\000000010000000000000005" "C:\temp\test_restore"
```

3. **Compressed archives without decompression in restore_command:**

If archives are gzipped, update restore_command:

**Linux:**
```ini
restore_command = 'gunzip < /mnt/archive/%f.gz > %p'
```

**Windows:**
```ini
# Using 7-Zip
restore_command = '"C:\\Program Files\\7-Zip\\7z.exe" x -so "C:\\archive\\%f.gz" > "%p"'

# Or using PowerShell
restore_command = 'powershell -Command "Expand-Archive -Path ''C:\\archive\\%f.zip'' -DestinationPath (Split-Path ''%p'') -Force; Move-Item ''%p.extracted'' ''%p''"'
```

**Issue 3: Recovery taking too long**

Symptoms:
- Recovery has been running for hours or days
- Progress seems very slow

Diagnosis:

```sql
-- Monitor recovery progress
SELECT pg_last_wal_receive_lsn(),
       pg_last_wal_replay_lsn(),
       pg_wal_lsn_diff(pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn()) AS replay_lag_bytes,
       pg_last_xact_replay_timestamp();
```

Solutions:

1. **Enable parallel recovery:**

```ini
recovery_prefetch = on
max_recovery_parallelism = 4
```

2. **Increase checkpoint frequency during recovery:**

```ini
# Force more frequent checkpoints during recovery
checkpoint_timeout = 5min
max_wal_size = 1GB
```

3. **Optimize I/O:**

```ini
# Increase maintenance_work_mem for faster index creation
maintenance_work_mem = 2GB

# Reduce fsync pressure
wal_sync_method = fdatasync
```

4. **Hardware limitations:**

Check I/O performance:

**Linux:**
```bash
# Monitor I/O wait times
iostat -x 5

# Check disk queue depth
iotop -o
```

**Windows:**
```powershell
# Monitor disk performance
Get-Counter '\PhysicalDisk(*)\% Disk Time' -Continuous

# Or use Performance Monitor
perfmon
# Add counters: PhysicalDisk -> % Disk Time, Avg. Disk Queue Length
```

Consider faster storage if recovery time is critical

**Issue 4: Recovery stops at wrong point**

Symptoms:
- Database recovers to a different time than specified
- Recovery stops earlier or later than expected

Diagnosis:

Check recovery settings:

**Linux:**
```bash
grep -E "recovery_target" /var/lib/postgresql/18/main/postgresql.conf
```

**Windows:**
```powershell
Select-String -Path "C:\Program Files\PostgreSQL\18\data\postgresql.conf" -Pattern "recovery_target"
```

Check what recovery actually stopped at:

**Linux:**
```bash
grep "recovery stopping" /var/log/postgresql/postgresql-18-main.log
```

**Windows:**
```powershell
Select-String -Path "C:\Program Files\PostgreSQL\18\data\log\postgresql-*.log" -Pattern "recovery stopping"
```

Solutions:

1. **Timezone confusion:**

Ensure recovery_target_time includes correct timezone:

```ini
# Explicit timezone
recovery_target_time = '2025-11-04 14:30:00+00'

# Or use UTC
recovery_target_time = '2025-11-04 14:30:00 UTC'
```

2. **Transaction timing:**

If using recovery_target_xid, remember that recovery_target_inclusive matters:

```ini
# Stop BEFORE transaction 12345
recovery_target_xid = '12345'
recovery_target_inclusive = false
```

3. **Timeline issues:**

Verify you're recovering on the correct timeline:

**Linux:**
```bash
ls -l /var/lib/postgresql/18/main/pg_wal/*.history
```

**Windows:**
```cmd
dir "C:\Program Files\PostgreSQL\18\data\pg_wal\*.history"
```

**Issue 5: Archive disk space filling up**

Symptoms:
- Archive directory growing uncontrollably
- Disk space alerts

Diagnosis:

**Linux:**
```bash
# Check archive size
du -sh /mnt/archive

# Check growth rate
du -sh /mnt/archive && sleep 3600 && du -sh /mnt/archive
```

**Windows:**
```powershell
# Check archive size
Get-ChildItem -Path C:\archive -Recurse | Measure-Object -Property Length -Sum | 
    Select-Object @{Name="Size(GB)";Expression={[math]::Round($_.Sum/1GB,2)}}

# Or simpler
"{0:N2} GB" -f ((Get-ChildItem C:\archive -Recurse | Measure-Object -Property Length -Sum).Sum / 1GB)

# Check available disk space
Get-Volume | Where-Object {$_.DriveLetter -eq 'C'}
```

Solutions:

1. **Implement archive cleanup:**

Use the cleanup script from earlier in the chapter.

2. **Reduce WAL generation:**

```ini
# Reduce full page writes (carefully!)
full_page_writes = off  # Only in specific scenarios

# Optimize checkpoint frequency
checkpoint_timeout = 30min
max_wal_size = 4GB
```

3. **Compress archives:**

**Linux:**
```ini
archive_command = 'gzip < %p > /mnt/archive/%f.gz'
restore_command = 'gunzip < /mnt/archive/%f.gz > %p'
```

**Windows:**
```ini
# Using 7-Zip
archive_command = '"C:\\Program Files\\7-Zip\\7z.exe" a -tgzip "C:\\archive\\%f.gz" "%p" > nul'
restore_command = '"C:\\Program Files\\7-Zip\\7z.exe" x -so "C:\\archive\\%f.gz" > "%p"'
```

4. **Use external backup tools:**

Tools like pgBackRest automatically manage archive retention.

### Logging and diagnostics

Configure appropriate logging for PITR operations:

```ini
# Recovery-specific logging
log_min_messages = info
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '

# Archive activity logging
log_checkpoints = on
log_connections = on
log_disconnections = on

# Additional WAL information
wal_debug = on  # Only for troubleshooting, disable in production

# Recovery conflict logging
log_recovery_conflict_waits = on
```

Create a dedicated log file for recovery:

```ini
# During recovery configuration
logging_collector = on
log_directory = '/var/log/postgresql'
log_filename = 'postgresql-recovery-%Y-%m-%d.log'
```

## PITR best practices and production considerations

Implementing PITR in production requires careful planning and adherence to best practices. This section distills key recommendations from real-world experience.

### Backup strategy design

A robust PITR strategy encompasses multiple elements:

**Backup frequency:**

Base backups should be taken regularly to minimize recovery time:
- High-activity databases: Daily full backups
- Medium-activity databases: Weekly full backups
- Low-activity databases: Weekly or bi-weekly full backups

Consider that recovery time increases with the age of the base backup, as more WAL must be replayed.

**Retention policy:**

Define clear retention policies:
- Operational backups: 7-30 days (for operational recovery)
- Compliance backups: As required by regulations (often 1-7 years)
- Archival backups: Long-term retention for historical purposes

Example policy:

```
Daily backups: Keep for 7 days
Weekly backups: Keep for 4 weeks
Monthly backups: Keep for 12 months
Yearly backups: Keep for 7 years
```

**Testing schedule:**

Test your backups regularly:
- Quick restore test: Weekly (automate with scripts)
- Full recovery test: Monthly (complete PITR to specific point)
- Disaster recovery drill: Quarterly (full team exercise)

Document test results:

```sql
CREATE TABLE backup_test_log (
    test_date DATE,
    backup_date DATE,
    recovery_target_time TIMESTAMPTZ,
    test_result TEXT,
    recovery_duration INTERVAL,
    notes TEXT
);
```

### Storage considerations

**Archive storage capacity:**

Calculate required archive storage:

```
Daily WAL Generation × Retention Days = Required Storage

Example:
- WAL generation: 50 GB/day
- Retention: 30 days
- Minimum storage: 1.5 TB
- Recommended: 2.5 TB (with 60% safety margin)
```

**Storage reliability:**

Archive storage must be highly reliable:
- Use RAID or equivalent redundancy
- Regular disk health checks
- Consider cloud storage with built-in redundancy
- Maintain multiple archive copies if possible

**Storage performance:**

Archive write performance affects database performance:
- SSDs recommended for high-throughput databases
- Network storage should have low latency and high bandwidth
- Consider write caching for local archives

**Geographic distribution:**

For disaster recovery, maintain archives in multiple locations:
- Primary site: Fast, local storage
- Secondary site: Offsite or cloud storage
- Tertiary site: Long-term archival storage

Example multi-location setup:

```ini
# Archive to both local and remote locations
archive_command = 'cp %p /mnt/local-archive/%f && \
                   rsync -a %p remote-site:/mnt/archive/%f'
```

### Security considerations

**Encryption at rest:**

Encrypt archived WAL files:

**Linux (using OpenSSL):**
```ini
# Encrypt during archiving
archive_command = 'openssl enc -aes-256-cbc -salt -pbkdf2 -in %p -out /mnt/archive/%f.enc -pass pass:$ENCRYPTION_KEY'

# Decrypt during recovery
restore_command = 'openssl enc -d -aes-256-cbc -pbkdf2 -in /mnt/archive/%f.enc -out %p -pass pass:$ENCRYPTION_KEY'
```

**Windows (using certutil or OpenSSL):**

If you have OpenSSL for Windows installed:
```ini
# Encrypt during archiving
archive_command = 'openssl enc -aes-256-cbc -salt -pbkdf2 -in "%p" -out "C:\\archive\\%f.enc" -pass pass:%ENCRYPTION_KEY%'

# Decrypt during recovery
restore_command = 'openssl enc -d -aes-256-cbc -pbkdf2 -in "C:\\archive\\%f.enc" -out "%p" -pass pass:%ENCRYPTION_KEY%'
```

Using PowerShell and .NET encryption:
```ini
archive_command = 'powershell -Command "$key=[System.Text.Encoding]::UTF8.GetBytes(''YourEncryptionKey''); $aes=[System.Security.Cryptography.Aes]::Create(); $aes.Key=$key; $encryptor=$aes.CreateEncryptor(); $fs=[System.IO.File]::OpenRead(''%p''); $cs=New-Object System.Security.Cryptography.CryptoStream([System.IO.File]::OpenWrite(''C:\\archive\\%f.enc''),$encryptor,[System.Security.Cryptography.CryptoStreamMode]::Write); $fs.CopyTo($cs); $cs.Close(); $fs.Close()"'
```

Better approach: Use key management systems:

**Linux (using AWS KMS):**
```bash
# Using AWS KMS
archive_command = 'aws kms encrypt \
                   --key-id alias/wal-archive-key \
                   --plaintext fileb://%p \
                   --output text \
                   --query CiphertextBlob | \
                   base64 -d > /mnt/archive/%f.enc'
```

**Windows (using AWS KMS):**
```ini
archive_command = 'powershell -Command "aws kms encrypt --key-id alias/wal-archive-key --plaintext fileb://\"%p\" --output text --query CiphertextBlob | ForEach-Object { [Convert]::FromBase64String($_) | Set-Content -Path ''C:\\archive\\%f.enc'' -Encoding Byte }"'
```

**Encryption in transit:**

When archiving to remote locations:

**Linux:**
- Use SSH/SCP for encrypted transfer:
  ```ini
  archive_command = 'scp %p backup-server:/archive/%f'
  ```
- Implement VPNs for site-to-site archiving
- Use HTTPS for cloud storage uploads

**Windows:**
- Use secure network shares with SMB3 encryption:
  ```ini
  archive_command = 'copy "%p" "\\\\backup-server\\archive\\%f"'
  ```
  (Ensure SMB3 encryption is enabled on the share)
- Use HTTPS for cloud storage uploads
- Implement VPNs for site-to-site archiving

**Access control:**

Restrict access to archives:

**Linux:**
```bash
# Archive directory permissions
chmod 700 /mnt/archive
chown postgres:postgres /mnt/archive

# Regular audit of access
sudo ausearch -f /mnt/archive -ts recent

# Or using auditctl to set up monitoring
sudo auditctl -w /mnt/archive -p rwxa -k archive_access
```

**Windows:**
```cmd
REM Archive directory permissions - restrict to service account only
icacls C:\archive /inheritance:r
icacls C:\archive /grant:r "NT AUTHORITY\NetworkService:(OI)(CI)F"
icacls C:\archive /remove:g Users
icacls C:\archive /remove:g "Authenticated Users"

REM Regular audit of access - enable auditing
auditpol /set /category:"Object Access" /success:enable /failure:enable

REM Set auditing on the archive folder
icacls C:\archive /audit:enable
icacls C:\archive /audit "Everyone:(OI)(CI)W"
```

**Windows (PowerShell - more detailed):**
```powershell
# Set strict permissions on archive directory
$path = "C:\archive"
$acl = Get-Acl $path
$acl.SetAccessRuleProtection($true, $false)  # Disable inheritance
$acl.Access | ForEach-Object { $acl.RemoveAccessRule($_) }  # Remove all existing rules

# Add access only for PostgreSQL service account
$serviceAccount = "NT AUTHORITY\NetworkService"
$rule = New-Object System.Security.AccessControl.FileSystemAccessRule(
    $serviceAccount,
    "FullControl",
    "ContainerInherit,ObjectInherit",
    "None",
    "Allow"
)
$acl.AddAccessRule($rule)
Set-Acl $path $acl

# Enable auditing
$audit = New-Object System.Security.AccessControl.FileSystemAuditRule(
    "Everyone",
    "Write,Delete,DeleteSubdirectoriesAndFiles",
    "ContainerInherit,ObjectInherit",
    "None",
    "Success,Failure"
)
$acl.AddAuditRule($audit)
Set-Acl $path $acl
```

**Backup verification:**

Implement checksums or signatures:

**Linux:**
```ini
# Generate SHA256 checksum during archive
archive_command = 'cp %p /mnt/archive/%f && \
                   sha256sum /mnt/archive/%f > /mnt/archive/%f.sha256'

# Verify during restore
restore_command = 'sha256sum -c /mnt/archive/%f.sha256 && \
                   cp /mnt/archive/%f %p'
```

**Windows:**
```ini
# Generate SHA256 checksum during archive (using PowerShell)
archive_command = 'copy "%p" "C:\\archive\\%f" && powershell -Command "$hash = Get-FileHash -Algorithm SHA256 ''C:\\archive\\%f''; $hash.Hash + ''  '' + $hash.Path | Out-File ''C:\\archive\\%f.sha256''"'

# Verify during restore (using PowerShell)
restore_command = 'powershell -Command "$expected = (Get-Content ''C:\\archive\\%f.sha256'').Split()[0]; $actual = (Get-FileHash -Algorithm SHA256 ''C:\\archive\\%f'').Hash; if ($expected -eq $actual) { copy ''C:\\archive\\%f'' ''%p'' } else { exit 1 }"'
```

Or using certutil (built into Windows):
```ini
# Generate checksum during archive
archive_command = 'copy "%p" "C:\\archive\\%f" && certutil -hashfile "C:\\archive\\%f" SHA256 > "C:\\archive\\%f.sha256"'

# Verify during restore
restore_command = 'certutil -hashfile "C:\\archive\\%f" SHA256 | findstr /v ":" > "C:\\temp\\current_hash.txt" && findstr /v ":" "C:\\archive\\%f.sha256" > "C:\\temp\\expected_hash.txt" && fc "C:\\temp\\current_hash.txt" "C:\\temp\\expected_hash.txt" > nul && copy "C:\\archive\\%f" "%p"'
```

### Performance optimization

**Archiving performance:**

Optimize archive_command for performance:

```ini
# Parallel compression (faster than gzip)
archive_command = 'pigz < %p > /mnt/archive/%f.gz'

# Asynchronous archiving (PostgreSQL 18 feature)
archive_library = 'basic_archive'
archive_mode = on
```

**Recovery performance:**

Optimize recovery speed:

```ini
# Enable parallel recovery
recovery_prefetch = on
max_recovery_parallelism = 4

# Increase memory for recovery
maintenance_work_mem = 2GB

# Optimize checkpoint behavior
checkpoint_timeout = 5min
checkpoint_completion_target = 0.9
```

**Resource management:**

Control resources during recovery:

```ini
# Limit parallel workers if recovery impacts other operations
max_recovery_parallelism = 2

# Throttle I/O if needed
# (Use external tools like ionice or cgroups)
```

### Integration with backup tools

While PostgreSQL provides the foundation for PITR, production environments often benefit from specialized backup tools.

**pgBackRest:**

pgBackRest is a popular choice for production PITR:

**Linux configuration (/etc/pgbackrest.conf):**
```ini
[global]
repo1-path=/var/lib/pgbackrest
repo1-retention-full=4

[pg1]
pg1-path=/var/lib/postgresql/18/main
```

**Windows configuration (C:\Program Files\pgBackRest\pgbackrest.conf):**
```ini
[global]
repo1-path=C:/pgbackrest
repo1-retention-full=4

[pg1]
pg1-path=C:/Program Files/PostgreSQL/18/data
```

**Note for Windows:** Use forward slashes (/) in pgBackRest config files, not backslashes.

pgBackRest handles:
- Compression and parallel backup
- Automatic retention management
- Backup verification
- Point-in-time recovery
- Backup encryption
- Cloud storage integration

**Barman:**

Barman (Backup and Recovery Manager) is another excellent tool for Linux environments:

**Note:** Barman is primarily designed for Linux/Unix systems. For Windows PostgreSQL instances, you can run Barman on a separate Linux server and connect to your Windows PostgreSQL instance remotely.

**Linux configuration:**
```ini
# Barman configuration
[pg1]
description = "Production PostgreSQL 18"
ssh_command = ssh postgres@pg1
conninfo = host=pg1 user=barman dbname=postgres
backup_method = postgres
archiver = on
```

**For Windows PostgreSQL instances managed by Linux Barman:**
```ini
[pg1-windows]
description = "Windows PostgreSQL 18"
conninfo = host=windows-pg-server port=5432 user=barman dbname=postgres
backup_method = postgres
archiver = on
streaming_archiver = on
slot_name = barman
```

**Wal-G:**

Wal-G focuses on cloud-native environments and works on both Linux and Windows:

**Linux configuration (using environment variables):**
```bash
# Configure for AWS S3
export AWS_ACCESS_KEY_ID=your-key
export AWS_SECRET_ACCESS_KEY=your-secret
export WALG_S3_PREFIX=s3://your-bucket/wal-archive

# Archive command
archive_command = 'wal-g wal-push %p'

# Restore command
restore_command = 'wal-g wal-fetch %f %p'
```

**Windows configuration:**

Set environment variables (System Properties > Environment Variables):
```
AWS_ACCESS_KEY_ID=your-key
AWS_SECRET_ACCESS_KEY=your-secret
WALG_S3_PREFIX=s3://your-bucket/wal-archive
```

Or set via PowerShell (temporary):
```powershell
$env:AWS_ACCESS_KEY_ID="your-key"
$env:AWS_SECRET_ACCESS_KEY="your-secret"
$env:WALG_S3_PREFIX="s3://your-bucket/wal-archive"
```

**PostgreSQL configuration (both platforms):**
```ini
# Archive command
archive_command = 'wal-g wal-push "%p"'

# Restore command
restore_command = 'wal-g wal-fetch "%f" "%p"'
```

### Monitoring and alerting

Implement comprehensive monitoring:

**Critical alerts:**

```sql
-- Alert on archive failures
SELECT CASE 
    WHEN failed_count > 0 
    THEN 'CRITICAL: WAL archiving failing'
    ELSE 'OK'
END AS alert
FROM pg_stat_archiver;

-- Alert on archive lag
SELECT CASE
    WHEN EXTRACT(EPOCH FROM (now() - last_archived_time)) > 600
    THEN 'WARNING: Archive lag exceeds 10 minutes'
    ELSE 'OK'
END AS alert
FROM pg_stat_archiver;

-- Alert on disk space
SELECT CASE
    WHEN (pg_available_space / pg_total_space::float) < 0.2
    THEN 'CRITICAL: Archive disk space below 20%'
    ELSE 'OK'
END AS alert
FROM (
    SELECT
        (SELECT setting FROM pg_settings WHERE name = 'archive_command') AS command,
        -- Add logic to extract path and check disk space
        pg_total_space,
        pg_available_space
    FROM check_disk_space()
) AS disk_check;
```

**Dashboard metrics:**

Create a monitoring dashboard tracking:
- Archive success rate (last hour/day)
- WAL generation rate
- Archive storage used/available
- Time since last successful archive
- Number of archived segments per hour
- Recovery testing results

### Documentation requirements

Maintain comprehensive documentation:

**Configuration documentation:**
- Archive location(s)
- Retention policies
- Encryption keys and procedures
- Recovery procedures
- Responsible personnel

**Recovery runbooks:**

Document step-by-step recovery procedures:

```markdown
# Emergency PITR Recovery Procedure

## Prerequisites
- Access to backup server
- PostgreSQL admin credentials
- Estimated downtime: 2-4 hours

## Steps
1. Identify recovery target time
2. Notify stakeholders
3. Stop application connections
4. Stop PostgreSQL
5. Backup current PGDATA
6. Restore base backup
7. Configure recovery
8. Start PostgreSQL in recovery mode
9. Monitor recovery progress
10. Verify database state
11. Promote if verified
12. Restore application connections
13. Document incident

## Emergency contacts
- DBA on-call: [phone]
- System administrator: [phone]
- Management: [phone]
```

**Change control:**

Document all changes to PITR configuration:
- Date and time of change
- Person making the change
- Reason for change
- Configuration before and after
- Testing performed

### Disaster recovery planning

PITR is a critical component of disaster recovery:

**Recovery Time Objective (RTO):**

Estimate recovery time:
```
RTO = Base backup restore time + WAL replay time + Verification time

Example:
- Base backup restore: 30 minutes
- WAL replay (24 hours): 2 hours
- Verification: 15 minutes
- Total RTO: ~3 hours
```

**Recovery Point Objective (RPO):**

Determine acceptable data loss:
```
RPO = archive_timeout + Maximum archive lag

Example:
- archive_timeout: 5 minutes
- Maximum acceptable lag: 5 minutes
- Total RPO: 10 minutes
```

**Regular drills:**

Conduct disaster recovery drills:
- Quarterly full DR tests
- Annual catastrophic failure simulation
- Include all team members
- Document lessons learned
- Update procedures based on findings

### Capacity planning

Plan for growth:

**Archive storage growth:**

```sql
-- Calculate monthly archive growth
WITH monthly_wal AS (
    SELECT 
        date_trunc('month', last_archived_time) AS month,
        COUNT(*) * 16 * 1024 * 1024 AS bytes -- 16MB per WAL
    FROM pg_stat_archiver
    GROUP BY month
)
SELECT 
    month,
    pg_size_pretty(bytes) AS size,
    pg_size_pretty(bytes * 12) AS projected_yearly
FROM monthly_wal
ORDER BY month DESC;
```

**Scaling considerations:**

As your database grows:
- Archive storage needs will increase proportionally
- Recovery time will increase
- Consider more frequent base backups
- Evaluate compression options
- Consider incremental backup strategies

## Summary

Point-In-Time Recovery represents one of the most sophisticated and powerful disaster recovery mechanisms available in PostgreSQL 18. Throughout this chapter, we've explored the complete spectrum of PITR, from basic concepts through advanced implementation and troubleshooting.

We began by understanding the architectural foundation of PITR, examining how Write-Ahead Logs and continuous archiving combine to create a complete database history. The ability to restore to any specific moment in time provides unprecedented flexibility in disaster recovery scenarios, whether recovering from accidental data deletion, corruption, or other operational incidents.

The configuration of WAL archiving emerged as a critical first step, requiring careful attention to the archive_command, storage considerations, and verification procedures. We explored various archiving strategies, from simple local copies to sophisticated cloud-based solutions, emphasizing that the reliability of the archive directly determines the success of any recovery attempt.

In the practical sections, we walked through complete PITR operations, covering everything from taking base backups to performing recoveries with various target specifications. The advanced techniques sectionâ€"covering named restore points, transaction ID-based recovery, timeline management, and parallel recoveryâ€"demonstrated the depth and flexibility that PostgreSQL 18 provides for handling complex recovery scenarios.

Monitoring and troubleshooting received significant attention, reflecting real-world experience that successful PITR implementations require robust observability and rapid problem resolution. The common issues we documented, along with their solutions, provide a troubleshooting foundation that will prove valuable when facing recovery challenges in production environments.

Finally, we examined best practices and production considerations, recognizing that PITR is not merely a technical implementation but a comprehensive disaster recovery strategy requiring planning, documentation, testing, and ongoing maintenance. The integration with specialized backup tools, security considerations, and capacity planning guidance provide a roadmap for enterprise-grade PITR implementations.

The key takeaway is that PITR is an invaluable tool in the PostgreSQL administrator's arsenal, but its effectiveness depends entirely on proper configuration, regular testing, and thorough understanding. A PITR configuration that is never tested is little better than no backup at all. Make PITR testing a regular practice, document your procedures thoroughly, and ensure your team understands the recovery process before an emergency occurs.

## Verify your knowledge

- Explain why a base backup alone is insufficient for comprehensive disaster recovery, and describe how PITR addresses this limitation. What specific recovery scenarios does PITR enable that simple backups cannot handle?

- You discover that your archive_command has been failing for the past six hours due to a network issue. What are the immediate risks to your database operation, and what steps would you take to assess the impact on your ability to perform PITR? Consider both the short-term operational impact and long-term recovery capabilities.

- Compare and contrast the different recovery target types (time-based, XID-based, LSN-based, and named restore points). In what scenarios would you choose each type? What are the trade-offs in terms of precision, ease of use, and prerequisites?

- Your production database experiences a catastrophic failure at 2:00 PM, and you need to perform PITR. You have daily base backups taken at midnight, and WAL archiving is functioning correctly. Walk through your decision-making process for determining the recovery target time. What factors would you consider? How would you verify the recovery was successful before returning the database to production?

- Describe the relationship between archive_timeout, WAL generation rate, and your Recovery Point Objective (RPO). How would you configure archive_timeout for a high-transaction database where you can tolerate no more than five minutes of data loss? What other considerations affect this decision?

- Timeline management becomes critical when performing multiple PITR operations from the same base backup. Explain what timelines are, why they exist, and describe a scenario where you would need to explicitly specify a recovery_target_timeline. What problems does the timeline mechanism prevent?

- You notice that recovery is taking significantly longer than expected. What diagnostic steps would you take to identify the bottleneck? Discuss at least four potential causes of slow recovery and the specific configuration changes or system-level optimizations you would implement to address each one.

- Design a comprehensive PITR testing strategy for a production database that processes financial transactions. Your strategy should address testing frequency, scope, automation, documentation requirements, and success criteria. How would you balance the need for thorough testing against the operational overhead?

- Explain the security implications of WAL archives and describe a multi-layered security approach for protecting archived WAL files. Consider encryption (at rest and in transit), access controls, audit logging, and geographic distribution. What key management strategies would you implement?

- A developer accidentally dropped a critical table at 10:45 AM, and you need to recover it without impacting other tables that have received legitimate updates since then. What approach would you take? Compare the advantages and disadvantages of at least three different recovery strategies, considering downtime, complexity, and data consistency.

## References

- PostgreSQL 18 official documentation on backup and PITR: https://www.postgresql.org/docs/18/continuous-archiving.html
- PostgreSQL archive_command examples: https://www.postgresql.org/docs/18/runtime-config-wal.html#RUNTIME-CONFIG-WAL-ARCHIVING
- PostgreSQL recovery configuration: https://www.postgresql.org/docs/18/recovery-config.html
- pg_basebackup documentation: https://www.postgresql.org/docs/18/app-pgbasebackup.html
- pg_verifybackup documentation: https://www.postgresql.org/docs/18/app-pgverifybackup.html
- Write-Ahead Logging (WAL) internals: https://www.postgresql.org/docs/18/wal-intro.html
- pgBackRest documentation: https://pgbackrest.org/user-guide.html
- Barman documentation: https://docs.pgbarman.org/
- Wal-G documentation: https://github.com/wal-g/wal-g
- PostgreSQL timeline management: https://www.postgresql.org/docs/18/continuous-archiving.html#BACKUP-TIMELINES
- PostgreSQL monitoring views: https://www.postgresql.org/docs/18/monitoring-stats.html
- High Availability, Load Balancing, and Replication chapter: https://www.postgresql.org/docs/18/high-availability.html

## Answers

**Question 1: Why is a base backup alone insufficient for comprehensive disaster recovery?**

A base backup captures the database state at a single point in time, which creates several significant limitations. First, you can only restore to exactly that moment, meaning any work done after the backup is lost. This is particularly problematic if backups are taken daily at midnight but failure occurs at 11:59 PM the next dayâ€"nearly 24 hours of data would be lost.

Second, a base backup cannot help you recover from logical errors that occurred before the backup was taken. If someone drops a critical table at 2 PM and you don't discover it until after the midnight backup runs, restoring that backup won't help because it already contains the dropped table state.

Third, base backups provide no flexibility in recovery targeting. You cannot stop just before a known bad transaction or recover to a specific application milestone. This all-or-nothing approach severely limits your response options during incidents.

PITR addresses these limitations by combining base backups with continuous WAL archiving. This creates a complete timeline of all database changes from the backup forward. With PITR, you can recover to any specific moment between the base backup and the present, allowing you to stop recovery just before an accidental DROP TABLE, recover to exactly the end of a business day, or target any arbitrary point in time. This temporal flexibility transforms disaster recovery from a binary restore operation into a precise, surgical procedure that can minimize or even eliminate data loss in many scenarios.

**Question 2: Impact of failing archive_command for six hours**

Six hours of archive_command failures creates an immediate risk that your WAL segments in the pg_wal directory will be recycled before archiving succeeds. PostgreSQL keeps only enough WAL segments to ensure crash recovery, typically based on max_wal_size and checkpoint settings. Once segments are recycled, those transactions are permanently lost for PITR purposes, creating a gap in your continuous backup chain.

The immediate assessment steps would be:

First, check pg_stat_archiver to see exactly how many archives have failed:
```sql
SELECT failed_count, last_failed_wal, last_failed_time FROM pg_stat_archiver;
```

Second, determine if any WAL segments have already been recycled by comparing the last archived WAL filename to the current WAL position:
```sql
SELECT pg_current_wal_lsn();
```

If the gap is more than a few WAL segments (each 16 MB), some segments may have been lost. Third, immediately resolve the network issue and allow archiving to catch up. Monitor archived_count increasing and failed_count stopping growth.

The long-term impact depends on whether gaps exist. If some WAL segments were recycled before archiving, you've lost the ability to perform continuous PITR through that time period. Your base backup is still valid, but you can only recover to the point just before the gap. Any recovery attempt that requires WAL segments from the gap period will fail.

To prevent this scenario, implement monitoring alerts on failed_count, ensure max_wal_size is configured generously, and consider using replication slots or wal_keep_size as safety mechanisms.

**Question 3: Comparing recovery target types**

Time-based recovery (recovery_target_time) is the most intuitive and commonly used approach. You specify "restore to 2:30 PM yesterday," which aligns well with business requirements and incident timestamps. It's easy to explain to non-technical stakeholders and maps directly to the "when did the problem occur?" question. The trade-off is that you need accurate timestamps, must account for timezone differences, and the precision is limited to PostgreSQL's timestamp resolution.

Transaction ID-based recovery (recovery_target_xid) provides the most precise control when you know exactly which transaction caused a problem. If you can identify that transaction 12345678 dropped a critical table, you can recover to immediately before that transaction. This is extremely precise but requires either track_commit_timestamp to be enabled or external transaction logging. The transaction ID must be known in advance, which often requires significant investigative work or sophisticated application-level transaction tracking.

LSN-based recovery (recovery_target_lsn) is the most granular option, targeting a specific position in the WAL stream. This is primarily useful in replication scenarios, when coordinating recovery across multiple databases, or when you have monitoring systems that record LSNs. It's less intuitive for typical disaster recovery and requires understanding of LSN progression.

Named restore points (recovery_target_name) offer a middle ground, combining precision with usability. By creating named markers before risky operations (like "before_major_upgrade"), you establish clear recovery targets without needing to track specific timestamps or transaction IDs. This requires proactive planning but provides the clearest recovery semantics. The limitation is that restore points must be created in advance, so they're useless for recovering from unexpected events.

The choice depends on your specific scenario: use time-based recovery for general incidents, XID-based for known problematic transactions, named restore points for planned risky operations, and LSN-based for replication coordination.

**Question 4: PITR after catastrophic failure at 2:00 PM**

The decision-making process for PITR after a catastrophic failure involves several critical steps and considerations.

First, you must establish the recovery target time. The natural inclination is to recover to just before the failure (2:00 PM), but this may not be optimal. You need to understand what caused the failure. Was it a hardware failure (recovery to latest possible moment), a bad transaction (recovery to just before that transaction), or data corruption that existed for some time (recovery to before corruption began)? Consulting application logs, error logs, and monitoring data helps establish this timeline.

Second, consider your base backup taken at midnight. This means you'll need to replay approximately 14 hours of WAL logs, which could take 1-3 hours depending on transaction volume and hardware. If the urgency is extreme, recovering to a closer base backup (if one exists) might be preferable even if it's from a different replica or test system.

Third, examine your WAL archive to verify that all necessary segments exist from midnight to 2:00 PM. Use archive listings and pg_waldump to validate continuity:
```bash
ls -lt /mnt/archive/ | grep $(date +%Y%m%d)
```

Fourth, decide whether to use recovery_target_action of 'pause' or 'promote'. Given the catastrophic nature, 'pause' is safer, allowing verification before promotion.

The verification process before returning to production should be comprehensive: verify key tables exist with correct row counts, run application-specific integrity checks, check for any corruption indicators, validate that critical data is intact, test a representative set of application queries, and verify authentication and permissions are correct.

Only after successful verification should you promote the server and restore application connections. Document the entire incident timeline, recovery target chosen, verification performed, and lessons learned for post-mortem analysis.

**Question 5: archive_timeout, WAL generation rate, and RPO relationship**

The Recovery Point Objective (RPO) defines the maximum acceptable data loss, typically measured in time. archive_timeout directly influences your achievable RPO by controlling how frequently incomplete WAL segments are forced to be archived.

For a high-transaction database with a five-minute RPO, you might initially think setting archive_timeout = 5min would suffice. However, this oversimplifies the relationship. The actual maximum data loss is determined by:

```
Maximum data loss = archive_timeout + archive_lag + disaster_detection_time
```

Archive lag is the time between when a WAL segment is completed and when the archive_command successfully completes. In a high-transaction environment, WAL segments (typically 16 MB) might complete every few seconds naturally, making archive_timeout less relevant. But during low-transaction periods (nights, weekends), archive_timeout becomes critical as it forces segment switches even when incomplete.

For a five-minute RPO, a more realistic configuration would be:
```ini
archive_timeout = 2min  # Provides safety margin
```

This assumes:
- archive_command typically completes within 30 seconds
- Disaster detection is nearly immediate (monitoring alerts)
- Total worst case: 2 minutes (timeout) + 30 seconds (archive lag) + 30 seconds (detection) = 3 minutes

However, other considerations affect this decision. A lower archive_timeout means more WAL segment switches, potentially creating many small, partially-filled segments in your archive. This increases archive storage requirements and management complexity. You must balance RPO requirements against storage costs and archive management overhead.

Additionally, the archive_command performance must be reliable. If archiving to remote storage, network latency and bandwidth become factors. A 2-minute archive_timeout is ineffective if the archive_command takes 3 minutes to complete during peak periods.

Finally, consider monitoring and alerting. Regardless of configuration, you need alerts when archiving falls behind or fails, as this directly impacts your achievable RPO.

**Question 6: Timeline management and multiple PITR operations**

Timelines represent different historical paths through the WAL stream, solving a fundamental problem: when you perform PITR and continue operating, you create an alternate timeline divergent from the original. Without timeline management, you couldn't distinguish between WAL from before and after the recovery, potentially mixing incompatible WAL segments and corrupting the database.

Consider this scenario: You take a base backup at midnight. At 2 PM, someone drops a critical table, so you perform PITR back to 1:55 PM. The database continues operating from 1:55 PM on a new timeline (timeline 2). Meanwhile, the original timeline (timeline 1) contained different transactions from 1:55 PM to 2 PM that are now on a different historical path.

If you later discover you actually needed data from 1:57 PM on the original timeline, you must explicitly specify recovery_target_timeline = 1 to access that data. Without timeline management, PostgreSQL wouldn't know which 1:57 PM you meant—the one on timeline 1 (original) or timeline 2 (post-recovery).

Timeline history files (.history) track these divergences:
```
1  0/4000000  before 2025-11-04 13:55:00+00
```

This indicates timeline 2 branched from timeline 1 at LSN 0/4000000 due to recovery to 1:55 PM.

A more complex scenario: you perform PITR to create a test environment from production. The test environment continues on timeline 2 while production stays on timeline 1. If you later want to restore production from the same base backup but to a different point, you must specify timeline 1 explicitly to follow the production WAL stream, not the test environment's timeline.

Timeline management prevents several catastrophic scenarios: accidentally mixing WAL from different recovery paths, which would corrupt the database; inability to perform multiple recovery attempts from the same base backup; confusion about which historical path a recovered database represents.

**Question 7: Diagnosing and resolving slow recovery**

When recovery is slower than expected, systematic diagnosis is essential. Start by quantifying the slowness—establish baseline expectations using the WAL generation rate and historical recovery tests.

First potential cause: I/O bottleneck. Monitor disk I/O with iostat or iotop during recovery. If iowait is high (>20%) and throughput is saturated, the storage system is the bottleneck. Solutions include:
- Enable recovery_prefetch to reduce random I/O through read-ahead
- Increase maintenance_work_mem for more efficient index rebuilding
- Use faster storage (NVMe vs. SSD vs. HDD makes dramatic differences)
- Ensure the restore_command isn't throttling reads from the archive

Second cause: CPU saturation. Recovery is increasingly parallel in PostgreSQL 18, but if max_recovery_parallelism is 0 or too low, recovery uses a single CPU core. Check CPU usage with top or htop. If one core is saturated while others idle:
- Set recovery_prefetch = on
- Set max_recovery_parallelism = 4 (or more, based on available cores)
- Verify shared_buffers and effective_cache_size are appropriately sized

Third cause: Lock contention. If hot_standby is enabled and queries are running against the recovering database, recovery conflicts can cause delays. Check pg_stat_database_conflicts:
```sql
SELECT * FROM pg_stat_database_conflicts;
```
If conflicts are numerous:
- Set max_standby_archive_delay higher to tolerate longer-running queries
- Kill or cancel conflicting queries
- Disable hot_standby during critical recovery (recovery_target_action can help)

Fourth cause: Inefficient restore_command. If WAL segments are being fetched slowly from the archive:
- Test restore_command manually: time cp /mnt/archive/WAL_FILE /tmp/test
- If remote archiving: check network latency and bandwidth
- If cloud storage: implement parallel fetch or local caching
- If decompressing: verify decompression isn't CPU-bound

Other diagnostic checks include examining the PostgreSQL log for specific slow operations (index rebuilds, constraint checks), verifying that checkpoint_timeout and max_wal_size aren't causing excessive checkpoint activity during recovery, and checking for hardware issues like memory pressure or thermal throttling.

The key is methodical diagnosis: measure first, theorize second, change one variable at a time, and verify the impact of each change.

**Question 8: Comprehensive PITR testing strategy for financial transactions**

A financial transaction database requires an exceptionally robust PITR testing strategy due to regulatory requirements, zero-tolerance for data loss, and the criticality of recovery accuracy.

Testing frequency must be multi-tiered: Daily automated quick tests verify that backups are taken successfully and basic restore functionality works. These tests should restore a recent backup to a test instance, verify basic connectivity, and check that critical tables exist. These tests should complete in under 30 minutes and alert immediately on failure.

Weekly automated functional tests should perform complete PITR to a specific restore point, verify data integrity through checksums or row counts against a known baseline, test recovery to multiple target types (time, XID, named restore point), and validate that recovered financial transactions balance correctly. These tests run overnight and generate detailed reports.

Monthly manual comprehensive tests involve the entire operations team executing the complete disaster recovery runbook. This includes failure simulation, team coordination, recovery execution, verification procedures, documentation, and stakeholder communication. These tests should treat the scenario as a real disaster, measuring actual recovery time and identifying procedural gaps.

The scope must be comprehensive: test recovery from all backup locations (local, remote, cloud), test recovery across database versions if you plan version upgrades, test partial recovery scenarios (selective table recovery), and test recovery after various failure types (hardware, corruption, accidental deletion, malicious actions).

Automation is critical but must be balanced with manual oversight. Automated tests provide rapid feedback and catch obvious failures, but cannot validate business logic or regulatory compliance. Use frameworks like pgBackRest's verify command and custom scripts that check specific business rules (transaction balancing, referential integrity of financial records, audit trail completeness).

Documentation requirements include detailed test plans specifying exactly what is tested and why, test results logged in a tracking system with pass/fail criteria, incident reports for any test failures, corrective action tracking, and trending analysis showing recovery time and success rate over time.

Success criteria must be explicit: database starts successfully, all critical tables present with correct row counts, financial transactions balance (debits = credits), audit logs intact and verifiable, recovery completed within RTO target, no error messages in PostgreSQL logs, and application-level integration tests pass.

The strategy should also include quarterly disaster recovery drills involving executive leadership, failure scenario variations (ensuring different failure types are tested), post-drill retrospectives with documented improvements, and regulatory compliance verification.

**Question 9: Multi-layered security approach for WAL archives**

WAL archives contain complete database history and represent a significant security concern. They can be used to reconstruct all data, including sensitive information, and must be protected with defense-in-depth strategies.

Encryption at rest is fundamental. Use filesystem-level encryption (LUKS, dm-crypt) for local archives, providing transparent encryption for all files. For finer-grained control, implement application-level encryption in the archive_command:

```bash
archive_command = 'openssl enc -aes-256-cbc -pbkdf2 -in %p -out /mnt/archive/%f.enc -pass file:/etc/postgresql/archive.key'
```

Key management is crucial. Never store encryption keys in the archive location or in postgresql.conf. Use a key management system (KMS) like HashiCorp Vault, AWS KMS, or Azure Key Vault. Implement key rotation procedures, requiring re-encryption of archives older than 90 days. Maintain key escrow procedures for disaster recovery, ensuring keys are available even if the primary KMS fails.

Encryption in transit protects data during archiving to remote locations. Use SSH for rsync-based archiving, TLS for cloud storage APIs (S3, Azure Blob, GCS), and VPN tunnels for site-to-site archiving. Never transfer WAL files over unencrypted protocols.

Access controls must be multi-layered. At the filesystem level, restrict archive directories to the postgres user only (chmod 700). Implement mandatory access control (SELinux or AppArmor) policies preventing even root from accessing archives without specific justification. For cloud storage, use IAM policies that grant least-privilege access, require MFA for any manual archive access, and implement time-limited credentials (no permanent keys).

Audit logging tracks all archive access. Enable filesystem audit logging (auditd on Linux) for the archive directory, log all archive_command executions with timestamp and checksum, implement cloud storage access logging, and maintain a tamper-proof audit trail (consider write-once storage or blockchain-based logging for high-security environments).

Geographic distribution provides redundancy and disaster recovery. Maintain three archive copies: primary (local, fast access for recovery), secondary (offsite, different physical location), and tertiary (cloud or long-term archival). Use different encryption keys for geographically distributed copies to limit the impact of a single key compromise.

Additional security measures include regular security audits of archive configuration, automated integrity checking (checksums verified on a schedule), incident response procedures specifically for archive compromise, and regular penetration testing of archive security.

For financial institutions or regulated industries, consider additional measures like Hardware Security Modules (HSMs) for key storage, compliance with specific standards (PCI-DSS, HIPAA, SOX), formal security certification processes, and third-party security audits.

**Question 10: Recovering a single dropped table without impacting other tables**

Recovering a single dropped table without affecting other tables requires a careful approach balancing complexity, downtime, and data consistency.

**Approach 1: Recover to separate instance and extract data**

This is the safest and most commonly recommended approach. Perform PITR to a separate server or test instance, recovering to just before 10:45 AM. Once recovery is complete, extract just the dropped table using pg_dump:

```bash
pg_dump -h recovery-server -U postgres -t dropped_table -Fc database_name > dropped_table.dump
```

Then restore only that table to production:

```bash
pg_restore -h production-server -U postgres -d database_name dropped_table.dump
```

Advantages: No production downtime, no risk to other tables, can verify recovered data before restoration, can be done by junior DBAs with supervision.

Disadvantages: Requires additional hardware/resources for recovery instance, slower (must set up complete recovery instance), potential for data inconsistency if the dropped table had foreign keys to tables that have changed since 10:45 AM, manual reconciliation of referencing data may be needed.

**Approach 2: Logical replication from recovered instance**

Similar to Approach 1, but uses logical replication for the table extraction:

1. Recover to secondary instance
2. Create publication for the dropped table on recovered instance
3. Create empty table structure in production
4. Create subscription in production pointing to recovered instance
5. Let logical replication copy the data
6. Drop subscription when complete

Advantages: Handles large tables more efficiently, can selectively filter data during replication, minimal production impact.

Disadvantages: More complex setup, requires understanding of logical replication, potential for replication lag, recovered instance must remain running during data transfer.

**Approach 3: Tablespace recovery with PITR**

If the table was in a separate tablespace, you could theoretically perform selective recovery:

1. Create new tablespace in production
2. Perform PITR to recovery point in separate PGDATA
3. Copy only the specific tablespace files
4. Attach tablespace and table to production

Advantages: Potentially faster for very large tables, less duplication of data.

Disadvantages: Extremely complex and risky, high chance of corruption if not done perfectly, requires deep understanding of PostgreSQL internals, generally not recommended except by experts in emergency situations, may not work if dropped table had dependencies.

**Recommended approach:**

For a financial database with a dropped critical table, Approach 1 is strongly recommended. Despite being slower, it provides the highest level of safety and verification. The process allows you to:

1. Verify the recovered table data integrity before restoring to production
2. Reconcile foreign key relationships carefully
3. Document exactly what data was recovered
4. Maintain audit trail for compliance
5. Perform the recovery during a maintenance window if needed
6. Test the restoration process in the recovery environment first

The additional time required (possibly an hour longer than direct approaches) is worthwhile given that financial data accuracy is paramount, regulatory compliance may require audit trails of recovery operations, and the risk of introducing inconsistencies or corruption must be minimized.

After recovering the table, thorough validation is essential: verify row counts match expectations from before the drop, check foreign key relationships for consistency, run application-level integrity checks, verify audit logs are complete, and get business stakeholder signoff before declaring recovery complete.