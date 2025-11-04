# Lab 5: Logical backups with pg_dump and pg_restore

**Estimated time:** 75 minutes  
**Difficulty level:** Beginner to Intermediate  
**PostgreSQL version:** 18.x  
**Platform requirements:** Windows only

---

## 1. Learning objectives

By the end of this lab, you will be able to:

- Create logical backups of a single database using pg_dump in multiple formats
- Restore databases from plain SQL and custom format backups
- Use compression to reduce backup file sizes
- Perform selective backups (schema only, data only, specific tables)
- Choose between COPY and INSERT statements for data portability
- Backup an entire cluster using pg_dumpall
- Perform selective restores using pg_restore

---

## 2. Prerequisites

**Knowledge prerequisites:**
- Basic understanding of PostgreSQL databases and tables
- Familiarity with command-line tools
- Understanding of file system navigation

**Previous labs:**
- Lab 2 (Installing Northwind database) must be completed

**Standard environment checklist:**
- [ ] PostgreSQL 18 installed on Windows (port 5432)
- [ ] Northwind database exists with all data populated
- [ ] Can connect as postgres user
- [ ] Command prompt access
- [ ] At least 500 MB free disk space for backups

---

## 3. Lab-specific setup

### Create a backup working directory

```
mkdir C:\pg_backups
cd C:\pg_backups
```

### Verify Northwind database exists and has data

```sql
psql -h localhost -p 5432 -U postgres -d northwind
```

```sql
-- Check table count
SELECT COUNT(*) FROM information_schema.tables 
WHERE table_schema = 'public' AND table_type = 'BASE TABLE';

-- Check data exists
SELECT 
    'customers' AS table_name, COUNT(*) AS row_count FROM customers
UNION ALL
SELECT 'orders', COUNT(*) FROM orders
UNION ALL
SELECT 'products', COUNT(*) FROM products
UNION ALL
SELECT 'employees', COUNT(*) FROM employees;

-- Exit psql
\q
```

Expected output: 13+ tables, 91 customers, 830 orders, 77 products, 9 employees

### Create a test directory structure

```
mkdir C:\pg_backups\full
mkdir C:\pg_backups\partial
mkdir C:\pg_backups\custom
mkdir C:\pg_backups\restore_test
```

---

## 4. Concept overview

Logical backups extract database content as SQL statements or data files, working through PostgreSQL's query interface like any other client application. Unlike physical backups that copy raw data files, logical backups produce human-readable (or structured) output that can be restored by executing SQL commands. PostgreSQL provides pg_dump for single database backups, pg_dumpall for cluster-wide backups, and pg_restore for restoring non-plain-text formats. Logical backups are portable across PostgreSQL versions and operating systems, making them ideal for migrations, though they can be slower than physical backups for very large databases.

---

## 5. Exercises

### Exercise 1.1: Basic pg_dump - plain SQL format

**Platform:** Windows  
**Objective:** Learn to create a basic logical backup in plain SQL format and restore it  
**Scenario:** You need to create a backup of the Northwind database for disaster recovery purposes

**Tasks:**

1. **Navigate to backup directory:**

   ```
   cd C:\pg_backups\full
   ```

2. **Create a basic pg_dump backup (plain SQL format):**

   ```
   pg_dump -h localhost -p 5432 -U postgres -d northwind -f northwind_basic.sql
   ```

   **Explanation:**
   - `pg_dump`: The backup utility for single databases
   - `-h localhost -p 5432`: Connect to local server on port 5432
   - `-U postgres`: Connect as postgres superuser
   - `-d northwind`: Database to backup
   - `-f northwind_basic.sql`: Output file (plain SQL text)

3. **Examine the backup file:**

   ```
   more northwind_basic.sql
   ```

   Press spacebar to scroll. Look for:
   - SET statements at the beginning (configuring restore environment)
   - CREATE TABLE statements (schema definitions)
   - COPY statements (data loading - PostgreSQL optimized)
   - ALTER TABLE statements (constraints and indexes)

4. **Check the backup file size:**

   ```
   dir northwind_basic.sql
   ```

   Expected size: approximately 300-400 KB

5. **View the backup with verbose output:**

   ```
   pg_dump -h localhost -p 5432 -U postgres -d northwind -f northwind_verbose.sql -v
   ```

   **Explanation:**
   - `-v`: Verbose mode - shows what pg_dump is doing
   - You'll see messages like "dumping contents of table customers"

   Expected output:
   ```
   pg_dump: last built-in OID is 16383
   pg_dump: reading extensions
   pg_dump: identifying extension members
   pg_dump: reading schemas
   pg_dump: reading user-defined tables
   pg_dump: reading user-defined functions
   pg_dump: reading user-defined types
   pg_dump: reading procedural languages
   pg_dump: reading user-defined aggregate functions
   pg_dump: reading user-defined operators
   pg_dump: reading user-defined access methods
   pg_dump: reading user-defined operator classes
   pg_dump: reading user-defined operator families
   pg_dump: reading user-defined text search parsers
   pg_dump: reading user-defined text search templates
   pg_dump: reading user-defined text search dictionaries
   pg_dump: reading user-defined text search configurations
   pg_dump: reading user-defined foreign-data wrappers
   pg_dump: reading user-defined foreign servers
   pg_dump: reading default privileges
   pg_dump: reading user-defined collations
   pg_dump: reading user-defined conversions
   pg_dump: reading type casts
   pg_dump: reading transforms
   pg_dump: reading table inheritance information
   pg_dump: reading event triggers
   pg_dump: finding extension tables
   pg_dump: finding inheritance relationships
   pg_dump: reading column info for interesting tables
   pg_dump: flagging inherited columns in subtables
   pg_dump: reading partitioning data
   pg_dump: reading indexes
   pg_dump: flagging indexes in partitioned tables
   pg_dump: reading extended statistics
   pg_dump: reading constraints
   pg_dump: reading triggers
   pg_dump: reading rewrite rules
   pg_dump: reading policies
   pg_dump: reading row-level security policies
   pg_dump: reading publication membership
   pg_dump: reading subscription membership
   pg_dump: reading large objects
   pg_dump: reading dependency data
   pg_dump: saving encoding = UTF8
   pg_dump: saving standard_conforming_strings = on
   pg_dump: saving search_path = 
   pg_dump: dumping contents of table public.categories
   pg_dump: dumping contents of table public.customers
   pg_dump: dumping contents of table public.employees
   pg_dump: dumping contents of table public.orders
   pg_dump: dumping contents of table public.products
   ...
   ```

**Verification steps:**

1. Verify backup file exists and has content:
   ```
   dir C:\pg_backups\full\northwind_basic.sql
   ```

2. Count SQL statements in backup:
   ```
   find /C "CREATE TABLE" northwind_basic.sql
   find /C "COPY" northwind_basic.sql
   ```

   Expected: 13+ CREATE TABLE, 13+ COPY statements

---

### Exercise 1.2: Restoring from plain SQL backup

**Platform:** Windows  
**Objective:** Learn to restore a database from a plain SQL backup file  
**Scenario:** You need to create a test copy of Northwind for development purposes

**Tasks:**

1. **Create a new empty database for restoration:**

   ```
   psql -h localhost -p 5432 -U postgres -d postgres -c "CREATE DATABASE northwind_restore WITH OWNER northwind;"
   ```

   Expected output:
   ```
   CREATE DATABASE
   ```

2. **Restore the backup using psql:**

   ```
   psql -h localhost -p 5432 -U postgres -d northwind_restore -f C:\pg_backups\full\northwind_basic.sql
   ```

   **Explanation:**
   - Connect to the NEW database (northwind_restore)
   - Execute all SQL statements from the backup file
   - Plain SQL backups are restored with psql, not pg_restore

   You'll see output like:
   ```
   SET
   SET
   SET
   CREATE TABLE
   ALTER TABLE
   CREATE TABLE
   ...
   COPY 8
   COPY 91
   COPY 9
   ...
   ALTER TABLE
   ```

3. **Verify restoration succeeded:**

   ```
   psql -h localhost -p 5432 -U postgres -d northwind_restore
   ```

   ```sql
   -- List all tables
   \dt

   -- Check row counts match original
   SELECT 
       'customers' AS table_name, COUNT(*) AS row_count FROM customers
   UNION ALL
   SELECT 'orders', COUNT(*) FROM orders
   UNION ALL
   SELECT 'products', COUNT(*) FROM products;
   ```

   Expected output:
   ```
                List of relations
    Schema |         Name          | Type  |   Owner   
   --------+-----------------------+-------+-----------
    public | categories            | table | northwind
    public | customer_customer_demo| table | northwind
    public | customers             | table | northwind
    public | employees             | table | northwind
    public | orders                | table | northwind
    public | products              | table | northwind
    ...
   (13 rows)

    table_name | row_count 
   ------------+-----------
    customers  |        91
    orders     |       830
    products   |        77
   (3 rows)
   ```

4. **Test data integrity by querying:**

   ```sql
   -- Query from orders joined with customers
   SELECT o.order_id, c.company_name, o.order_date, o.freight
   FROM orders o
   JOIN customers c ON o.customer_id = c.customer_id
   ORDER BY o.order_id
   LIMIT 5;
   ```

   Expected output:
   ```
    order_id |     company_name      | order_date | freight 
   ----------+-----------------------+------------+---------
       10248 | Vins et alcools Chevalier |1996-07-04|  32.38
       10249 | Toms Spezialitäten    | 1996-07-05 |  11.61
       10250 | Hanari Carnes         | 1996-07-08 |  65.83
       10251 | Victuailles en stock  | 1996-07-08 |  41.34
       10252 | Suprêmes délices      | 1996-07-09 |  51.30
   (5 rows)
   ```

5. **Exit psql:**

   ```sql
   \q
   ```

**Verification steps:**

- All 13+ tables exist in northwind_restore
- Row counts match the original northwind database
- Joins work correctly (foreign keys intact)
- Table owners are set to northwind role

---

### Exercise 1.3: Using custom format for selective restore

**Platform:** Windows  
**Objective:** Learn to create custom format backups and perform selective restoration  
**Scenario:** You need flexibility to restore only specific tables later

**Tasks:**

1. **Create a custom format backup:**

   ```
   cd C:\pg_backups\custom
   pg_dump -h localhost -p 5432 -U postgres -d northwind -F c -f northwind_custom.backup
   ```

   **Explanation:**
   - `-F c`: Custom format (PostgreSQL-specific, compressed, allows selective restore)
   - Output is binary, not readable text
   - Requires pg_restore to restore (not psql)

2. **Create a directory format backup:**

   ```
   pg_dump -h localhost -p 5432 -U postgres -d northwind -F d -f northwind_dir
   ```

   **Explanation:**
   - `-F d`: Directory format - creates a directory with one file per table
   - Each table's data is compressed separately
   - Also requires pg_restore

3. **Examine the directory format:**

   ```
   dir northwind_dir
   ```

   Expected output:
   ```
   Directory of C:\pg_backups\custom\northwind_dir

   toc.dat
   3456.dat.gz
   3457.dat.gz
   3458.dat.gz
   ...
   ```

   Each .dat.gz file contains one table's data (compressed)

4. **Compare file sizes:**

   ```
   dir northwind_basic.sql
   dir northwind_custom.backup
   dir northwind_dir
   ```

   Expected observations:
   - Custom format: ~150-200 KB (compressed)
   - Directory format: similar total size, split across files
   - Plain SQL: ~300-400 KB (uncompressed)

5. **List contents of custom backup without restoring:**

   ```
   pg_restore -l northwind_custom.backup > backup_toc.txt
   more backup_toc.txt
   ```

   **Explanation:**
   - `-l`: List table of contents
   - Shows what's in the backup without restoring
   - Each line has an ID number used for selective restore

   Expected output:
   ```
   ;
   ; Archive created at 2025-11-04 15:30:45 EST
   ;     dbname: northwind
   ;     TOC Entries: 145
   ;     Compression: -1
   ;     Dump Version: 1.14-0
   ;     Format: CUSTOM
   ;     Integer: 4 bytes
   ;     Offset: 8 bytes
   ;     Dumped from database version: 18.0
   ;     Dumped by pg_dump version: 18.0
   ;
   ;
   ; Selected TOC Entries:
   ;
   3456; 1259 16385 TABLE public categories northwind
   3457; 1259 16386 SEQUENCE public categories_category_id_seq northwind
   3458; 1259 16388 TABLE public customers northwind
   3459; 1259 16390 TABLE public employees northwind
   ...
   3500; 0 16385 TABLE DATA public categories northwind
   3501; 0 16388 TABLE DATA public customers northwind
   ...
   ```

6. **Perform a selective restore - orders table only:**

   First, create a test database:
   ```
   psql -h localhost -p 5432 -U postgres -c "CREATE DATABASE northwind_partial OWNER northwind;"
   ```

   Restore only the orders table structure and data:
   ```
   pg_restore -h localhost -p 5432 -U postgres -d northwind_partial -t orders northwind_custom.backup
   ```

   **Explanation:**
   - `-t orders`: Restore only the orders table
   - pg_restore figures out dependencies automatically

7. **Verify selective restore:**

   ```
   psql -h localhost -p 5432 -U postgres -d northwind_partial
   ```

   ```sql
   -- Should show only orders table
   \dt

   -- Count orders
   SELECT COUNT(*) FROM orders;

   -- Try to query customers (should fail - not restored)
   SELECT COUNT(*) FROM customers;
   ```

   Expected:
   ```
           List of relations
    Schema |  Name  | Type  |   Owner   
   --------+--------+-------+-----------
    public | orders | table | northwind
   (1 row)

    count 
   -------
      830
   (1 row)

   ERROR:  relation "customers" does not exist
   ```

8. **Restore additional tables to the same database:**

   ```sql
   \q
   ```

   ```
   pg_restore -h localhost -p 5432 -U postgres -d northwind_partial -t customers -t products northwind_custom.backup
   ```

   Verify:
   ```
   psql -h localhost -p 5432 -U postgres -d northwind_partial -c "\dt"
   ```

   Expected:
   ```
           List of relations
    Schema |   Name    | Type  |   Owner   
   --------+-----------+-------+-----------
    public | customers | table | northwind
    public | orders    | table | northwind
    public | products  | table | northwind
   (3 rows)
   ```

**Verification steps:**

- Custom and directory format backups created successfully
- Table of contents can be listed with pg_restore -l
- Selective restore works (only specified tables restored)
- Can incrementally add more tables to existing database

---

### Exercise 1.4: Schema-only and data-only backups

**Platform:** Windows  
**Objective:** Learn to backup and restore database structure separately from data  
**Scenario:** You need to recreate database structure on a new server without data, or export data for analysis

**Tasks:**

1. **Create a schema-only backup (structure without data):**

   ```
   cd C:\pg_backups\partial
   pg_dump -h localhost -p 5432 -U postgres -d northwind -s -f northwind_schema.sql
   ```

   **Explanation:**
   - `-s`: Schema only - includes CREATE TABLE, indexes, constraints, but no data
   - Useful for setting up empty databases with identical structure

2. **Examine the schema-only backup:**

   ```
   more northwind_schema.sql
   ```

   Look for:
   - CREATE TABLE statements: YES
   - COPY statements: NO (no data)
   - CREATE INDEX statements: YES
   - ALTER TABLE for foreign keys: YES

   ```
   find /C "CREATE TABLE" northwind_schema.sql
   find /C "COPY" northwind_schema.sql
   ```

   Expected: 13+ CREATE TABLE, 0 COPY statements

3. **Create a data-only backup (data without structure):**

   ```
   pg_dump -h localhost -p 5432 -U postgres -d northwind -a -f northwind_data.sql
   ```

   **Explanation:**
   - `-a`: Data only - includes COPY statements but no CREATE TABLE
   - Assumes tables already exist in target database
   - Useful for refreshing data in existing structure

4. **Examine the data-only backup:**

   ```
   find /C "CREATE TABLE" northwind_data.sql
   find /C "COPY" northwind_data.sql
   ```

   Expected: 0 CREATE TABLE, 13+ COPY statements

5. **Test two-step restore (schema then data):**

   Create empty database:
   ```
   psql -h localhost -p 5432 -U postgres -c "CREATE DATABASE northwind_twostep OWNER northwind;"
   ```

   Restore schema first:
   ```
   psql -h localhost -p 5432 -U postgres -d northwind_twostep -f northwind_schema.sql
   ```

   Verify tables exist but are empty:
   ```
   psql -h localhost -p 5432 -U postgres -d northwind_twostep
   ```

   ```sql
   \dt
   SELECT COUNT(*) FROM customers;
   SELECT COUNT(*) FROM orders;
   \q
   ```

   Expected:
   ```
   (13 rows of tables listed)
   
    count 
   -------
        0
   (1 row)
   ```

6. **Load the data:**

   ```
   psql -h localhost -p 5432 -U postgres -d northwind_twostep -f northwind_data.sql
   ```

   Verify data loaded:
   ```
   psql -h localhost -p 5432 -U postgres -d northwind_twostep -c "SELECT COUNT(*) FROM customers;"
   ```

   Expected:
   ```
    count 
   -------
       91
   (1 row)
   ```

7. **Backup specific tables only:**

   ```
   pg_dump -h localhost -p 5432 -U postgres -d northwind -t customers -t orders -f customers_orders.sql
   ```

   **Explanation:**
   - `-t tablename`: Include only this table (can repeat for multiple tables)
   - Useful for partial backups or data exports

8. **Verify specific table backup:**

   ```
   find /C "CREATE TABLE" customers_orders.sql
   ```

   Expected: 2 CREATE TABLE statements (customers and orders only)

9. **Exclude tables from backup:**

   ```
   pg_dump -h localhost -p 5432 -U postgres -d northwind -T employees -T employee_territories -f northwind_no_hr.sql
   ```

   **Explanation:**
   - `-T tablename`: Exclude this table (can repeat for multiple tables)
   - Useful when you want most tables but not sensitive ones

10. **Verify exclusion:**

    ```
    find /C "employees" northwind_no_hr.sql
    ```

    Expected: Should not find employee-related CREATE TABLE or COPY

**Verification steps:**

- Schema-only backup contains no COPY statements
- Data-only backup contains no CREATE TABLE statements
- Two-step restore (schema + data) produces complete database
- Selective table backup includes only specified tables
- Table exclusion works correctly

---

### Exercise 1.5: Using INSERT instead of COPY for portability

**Platform:** Windows  
**Objective:** Learn to create portable backups using standard SQL INSERT statements  
**Scenario:** You need to migrate Northwind data to a different database system (MySQL, SQL Server)

**Tasks:**

1. **Create backup with INSERT statements:**

   ```
   cd C:\pg_backups\full
   pg_dump -h localhost -p 5432 -U postgres -d northwind --inserts -f northwind_inserts.sql
   ```

   **Explanation:**
   - `--inserts`: Use INSERT statements instead of COPY
   - More portable (works on other databases)
   - Slower to restore than COPY
   - INSERT statements don't include column names

2. **Create backup with column names in INSERT:**

   ```
   pg_dump -h localhost -p 5432 -U postgres -d northwind --column-inserts -f northwind_column_inserts.sql
   ```

   **Explanation:**
   - `--column-inserts`: INSERT statements include column names
   - Most portable format
   - Works even if target table has different column order
   - Slowest to restore

3. **Compare the output formats:**

   COPY format (from earlier):
   ```
   more C:\pg_backups\full\northwind_basic.sql
   ```
   Look for:
   ```
   COPY public.categories (category_id, category_name, description, picture) FROM stdin;
   1	Beverages	Soft drinks, coffees, teas, beers, and ales	\x
   2	Condiments	Sweet and savory sauces, relishes, spreads, and seasonings	\x
   \.
   ```

   INSERT format (without columns):
   ```
   more northwind_inserts.sql
   ```
   Look for:
   ```
   INSERT INTO public.categories VALUES (1, 'Beverages', 'Soft drinks, coffees, teas, beers, and ales', '\x');
   INSERT INTO public.categories VALUES (2, 'Condiments', 'Sweet and savory sauces, relishes, spreads, and seasonings', '\x');
   ```

   INSERT format (with columns):
   ```
   more northwind_column_inserts.sql
   ```
   Look for:
   ```
   INSERT INTO public.categories (category_id, category_name, description, picture) VALUES (1, 'Beverages', 'Soft drinks, coffees, teas, beers, and ales', '\x');
   INSERT INTO public.categories (category_id, category_name, description, picture) VALUES (2, 'Condiments', 'Sweet and savory sauces, relishes, spreads, and seasonings', '\x');
   ```

4. **Compare file sizes:**

   ```
   dir northwind_basic.sql
   dir northwind_inserts.sql
   dir northwind_column_inserts.sql
   ```

   Expected observations:
   - COPY format: smallest (~300 KB)
   - INSERT format: larger (~400-500 KB)
   - Column INSERT format: largest (~500-600 KB)

5. **Test restore performance (timed):**

   COPY format restore:
   ```
   psql -h localhost -p 5432 -U postgres -c "DROP DATABASE IF EXISTS test_copy;"
   psql -h localhost -p 5432 -U postgres -c "CREATE DATABASE test_copy OWNER northwind;"
   
   echo Starting COPY restore...
   powershell -Command "Measure-Command {psql -h localhost -p 5432 -U postgres -d test_copy -f C:\pg_backups\full\northwind_basic.sql -q}"
   ```

   INSERT format restore:
   ```
   psql -h localhost -p 5432 -U postgres -c "DROP DATABASE IF EXISTS test_insert;"
   psql -h localhost -p 5432 -U postgres -c "CREATE DATABASE test_insert OWNER northwind;"
   
   echo Starting INSERT restore...
   powershell -Command "Measure-Command {psql -h localhost -p 5432 -U postgres -d test_insert -f northwind_inserts.sql -q}"
   ```

   Expected observation:
   - COPY restore: faster (2-5 seconds)
   - INSERT restore: slower (5-15 seconds)
   - Difference increases with larger databases

6. **Verify both restores are identical:**

   ```
   psql -h localhost -p 5432 -U postgres -d test_copy -c "SELECT COUNT(*) FROM orders;" -t
   psql -h localhost -p 5432 -U postgres -d test_insert -c "SELECT COUNT(*) FROM orders;" -t
   ```

   Both should return: 830

**Verification steps:**

- INSERT format backups are larger than COPY format
- Column INSERT format is most verbose but most portable
- COPY format restores faster than INSERT format
- Both formats produce identical restored databases
- Understand trade-off: portability vs. performance

---

### Exercise 1.6: Cluster-wide backup with pg_dumpall

**Platform:** Windows  
**Objective:** Learn to backup entire PostgreSQL cluster including all databases and roles  
**Scenario:** You need to migrate or backup your entire PostgreSQL installation

**Tasks:**

1. **Create a full cluster backup:**

   ```
   cd C:\pg_backups
   pg_dumpall -h localhost -p 5432 -U postgres -f cluster_full.sql
   ```

   **Explanation:**
   - `pg_dumpall`: Backs up ALL databases in the cluster
   - Includes global objects (roles, tablespaces)
   - Always produces plain SQL format (no custom format option)
   - Must connect as superuser

   This will take a minute as it backs up:
   - postgres database
   - template0, template1 databases
   - northwind database
   - All created roles (northwind, accounting, hr, john, bill from previous labs)

2. **Examine the cluster backup:**

   ```
   more cluster_full.sql
   ```

   Look for these sections:
   - CREATE ROLE statements at the beginning
   - Multiple CREATE DATABASE statements
   - \connect statements switching between databases
   - Complete content of each database

3. **Extract only global objects (roles and tablespaces):**

   ```
   pg_dumpall -h localhost -p 5432 -U postgres -g -f cluster_globals.sql
   ```

   **Explanation:**
   - `-g`: Globals only - just roles and tablespaces
   - Does NOT include any databases or their content
   - Useful for setting up roles on a new server

4. **Examine the globals-only backup:**

   ```
   more cluster_globals.sql
   ```

   Look for:
   ```
   CREATE ROLE northwind;
   ALTER ROLE northwind WITH NOSUPERUSER INHERIT NOCREATEROLE NOCREATEDB LOGIN;
   CREATE ROLE accounting;
   CREATE ROLE john;
   ...
   ```

5. **Backup only roles (no databases):**

   ```
   pg_dumpall -h localhost -p 5432 -U postgres -r -f cluster_roles.sql
   ```

   **Explanation:**
   - `-r`: Roles only (similar to -g but excludes tablespaces)

6. **Compare file sizes:**

   ```
   dir cluster_full.sql
   dir cluster_globals.sql
   dir cluster_roles.sql
   ```

   Expected:
   - cluster_full.sql: several MB (all databases)
   - cluster_globals.sql: few KB (just roles/tablespaces)
   - cluster_roles.sql: few KB (just roles)

7. **Verify cluster backup includes all databases:**

   ```
   find /C "CREATE DATABASE" cluster_full.sql
   find /C "\\connect" cluster_full.sql
   ```

   Expected: Multiple occurrences (postgres, northwind, test databases)

8. **Simulate disaster recovery scenario:**

   **WARNING:** Only do this on a test system!

   View current roles:
   ```
   psql -h localhost -p 5432 -U postgres -c "\du"
   ```

   To test restoration of roles (non-destructive):
   ```
   psql -h localhost -p 5432 -U postgres -c "CREATE ROLE test_recovery LOGIN PASSWORD 'test';"
   psql -h localhost -p 5432 -U postgres -c "\du" | find "test_recovery"
   psql -h localhost -p 5432 -U postgres -c "DROP ROLE test_recovery;"
   ```

   The cluster_globals.sql would recreate all roles on a fresh PostgreSQL installation

**Verification steps:**

- pg_dumpall creates a single SQL file with all databases
- Cluster backup includes CREATE ROLE statements
- Globals-only backup contains no database content
- Cluster backup is significantly larger than single database backup
- Understand when to use pg_dumpall vs pg_dump

---

### Exercise 1.7: Compression and backup automation

**Platform:** Windows  
**Objective:** Learn to compress backups and understand automation concepts  
**Scenario:** You have limited storage and want to schedule automated backups

**Tasks:**

1. **Create compressed backup using gzip-style compression:**

   ```
   cd C:\pg_backups\full
   pg_dump -h localhost -p 5432 -U postgres -d northwind -F c -Z 9 -f northwind_compressed.backup
   ```

   **Explanation:**
   - `-F c`: Custom format (already uses some compression)
   - `-Z 9`: Maximum compression level (0=none, 9=maximum)
   - Trade-off: more compression = slower backup

2. **Create uncompressed custom format for comparison:**

   ```
   pg_dump -h localhost -p 5432 -U postgres -d northwind -F c -Z 0 -f northwind_uncompressed.backup
   ```

   **Explanation:**
   - `-Z 0`: No additional compression
   - Faster backup but larger file

3. **Compare compressed vs uncompressed:**

   ```
   dir northwind_compressed.backup
   dir northwind_uncompressed.backup
   ```

   Expected:
   - Compressed: ~100-150 KB
   - Uncompressed: ~250-350 KB
   - Compression ratio: ~50-60%

4. **Test restore performance of compressed backup:**

   ```
   psql -h localhost -p 5432 -U postgres -c "DROP DATABASE IF EXISTS test_compressed;"
   psql -h localhost -p 5432 -U postgres -c "CREATE DATABASE test_compressed OWNER northwind;"
   
   pg_restore -h localhost -p 5432 -U postgres -d test_compressed northwind_compressed.backup
   ```

   Verify:
   ```
   psql -h localhost -p 5432 -U postgres -d test_compressed -c "SELECT COUNT(*) FROM orders;"
   ```

   Expected: 830 orders (decompression happens automatically)

5. **Create a directory format backup with parallel dumps:**

   ```
   pg_dump -h localhost -p 5432 -U postgres -d northwind -F d -f northwind_parallel -j 4
   ```

   **Explanation:**
   - `-j 4`: Use 4 parallel jobs to backup
   - Faster on multi-core systems
   - Only works with directory format (-F d)
   - Each table is dumped by a separate process

6. **Compare parallel vs serial backup times:**

   Serial:
   ```
   powershell -Command "Measure-Command {pg_dump -h localhost -p 5432 -U postgres -d northwind -F d -f northwind_serial}"
   ```

   Parallel:
   ```
   powershell -Command "Measure-Command {pg_dump -h localhost -p 5432 -U postgres -d northwind -F d -f northwind_parallel2 -j 4}"
   ```

   Expected: Parallel is faster (50-75% of serial time) for larger databases

7. **Create a Windows batch script for automated backup:**

   Create file: `C:\pg_backups\backup_northwind.bat`

   ```batch
   @echo off
   REM Automated Northwind Backup Script
   REM Run this script with Task Scheduler for automated backups
   
   SET BACKUP_DIR=C:\pg_backups\automated
   SET TIMESTAMP=%DATE:~-4%%DATE:~-10,2%%DATE:~-7,2%_%TIME:~0,2%%TIME:~3,2%
   SET TIMESTAMP=%TIMESTAMP: =0%
   SET BACKUP_FILE=%BACKUP_DIR%\northwind_%TIMESTAMP%.backup
   
   echo Starting backup at %DATE% %TIME%
   echo Backup file: %BACKUP_FILE%
   
   REM Create backup directory if not exists
   if not exist "%BACKUP_DIR%" mkdir "%BACKUP_DIR%"
   
   REM Perform backup
   "C:\Program Files\PostgreSQL\18\bin\pg_dump.exe" -h localhost -p 5432 -U postgres -d northwind -F c -Z 6 -f "%BACKUP_FILE%"
   
   if %ERRORLEVEL% EQU 0 (
       echo Backup completed successfully
   ) else (
       echo Backup failed with error code %ERRORLEVEL%
   )
   
   REM Clean up backups older than 7 days
   forfiles /P "%BACKUP_DIR%" /M northwind_*.backup /D -7 /C "cmd /c del @path" 2>nul
   
   echo Backup process finished at %DATE% %TIME%
   ```

8. **Test the backup script:**

   ```
   mkdir C:\pg_backups\automated
   C:\pg_backups\backup_northwind.bat
   ```

   Verify backup was created:
   ```
   dir C:\pg_backups\automated
   ```

   Expected: File named like `northwind_20251104_1530.backup`

9. **Create a backup rotation script:**

   This concept (implemented in the batch script above):
   - Keeps backups from last 7 days
   - Automatically deletes older backups
   - Prevents disk from filling up

10. **Document backup strategy:**

    Create file: `C:\pg_backups\BACKUP_STRATEGY.txt`

    ```
    Northwind Database Backup Strategy
    ===================================
    
    Backup Schedule:
    - Daily: Full backup at 2 AM (using Task Scheduler)
    - Weekly: Cluster-wide backup on Sundays
    - Monthly: Archive backup to external storage
    
    Backup Types:
    - Daily: Custom format, compressed (-F c -Z 6)
    - Weekly: pg_dumpall cluster backup
    - Monthly: Directory format for long-term archive
    
    Retention:
    - Daily backups: 7 days
    - Weekly backups: 4 weeks
    - Monthly backups: 1 year
    
    Storage Locations:
    - Daily: C:\pg_backups\automated
    - Weekly: C:\pg_backups\weekly
    - Monthly: External drive or cloud storage
    
    Recovery Procedures:
    - Daily recovery: pg_restore from custom format
    - Disaster recovery: pg_dumpall cluster restore
    
    Testing:
    - Test restore monthly to verify backup integrity
    - Document restore times for RTO planning
    ```

**Verification steps:**

- Compressed backups are smaller than uncompressed
- Compression level affects file size and backup time
- Parallel backups work with directory format
- Batch script creates properly named backup files
- Understand backup rotation and retention concepts

---

## 6. Validation checklist

After completing all exercises, verify:

- [ ] Can create plain SQL backup with pg_dump
- [ ] Can restore from plain SQL using psql
- [ ] Can create custom format backup (-F c)
- [ ] Can create directory format backup (-F d)
- [ ] Can list backup contents with pg_restore -l
- [ ] Can perform selective restore with pg_restore -t
- [ ] Can create schema-only backup (-s)
- [ ] Can create data-only backup (-a)
- [ ] Understand COPY vs INSERT trade-offs
- [ ] Can create cluster-wide backup with pg_dumpall
- [ ] Can create compressed backups with -Z
- [ ] Can perform parallel backups with -j
- [ ] Have created backup automation script

**Final validation query:**

```sql
-- Connect to original and restored database
psql -h localhost -p 5432 -U postgres -d northwind

-- Get checksums of all tables
SELECT 
    schemaname,
    tablename,
    n_live_tup AS row_count
FROM pg_stat_user_tables
ORDER BY tablename;

\q

-- Compare with restored database
psql -h localhost -p 5432 -U postgres -d northwind_restore

SELECT 
    schemaname,
    tablename,
    n_live_tup AS row_count
FROM pg_stat_user_tables
ORDER BY tablename;
```

Row counts should match between original and restored databases.

---

## 7. Troubleshooting guide

### Windows

**Problem:** "pg_dump: command not found"
- **Solution:** Add PostgreSQL bin to PATH or use full path: `"C:\Program Files\PostgreSQL\18\bin\pg_dump.exe"`

**Problem:** "password authentication failed"
- **Solution:** Use PGPASSWORD environment variable: `set PGPASSWORD=yourpassword` before running pg_dump
- **Or:** Create .pgpass file in `%APPDATA%\postgresql\pgpass.conf`

**Problem:** "database 'northwind' does not exist"
- **Solution:** Verify database name: `psql -l` and check spelling

**Problem:** "permission denied to create database"
- **Solution:** Connect as superuser (postgres) or grant CREATEDB to your role

**Problem:** Restore fails with "relation already exists"
- **Solution:** Drop and recreate database before restoring, or use `--clean` option with pg_dump
- **Or:** Use pg_restore with `-c` to clean before restore

**Problem:** "COPY failed: invalid byte sequence for encoding"
- **Solution:** Ensure backup and restore use same encoding
- **Check:** `psql -l` shows database encoding

**Problem:** Custom format backup cannot be read as text
- **Solution:** Use pg_restore, not psql, for custom/directory/tar formats
- **Remember:** Only plain SQL format works with psql

**Problem:** Parallel backup fails
- **Solution:** Parallel (-j) only works with directory format (-F d)
- **Use:** `pg_dump -F d -j 4` not `pg_dump -F c -j 4`

**Problem:** Backup file is huge
- **Solution:** Use compression: `-F c -Z 6` or higher
- **Or:** Use selective backup: `-t` for specific tables

**Problem:** pg_restore says "no matching tables found"
- **Solution:** Check table name spelling and case
- **List contents:** `pg_restore -l backup.file` to see available tables

---

## 8. Questions

1. When would you choose custom format (-F c) over plain SQL format for pg_dump backups?

2. Why does the book recommend testing backups by actually restoring them rather than just verifying the file exists?

3. Explain the trade-off between using COPY statements versus INSERT statements in your backup file.

4. When would you use pg_dumpall instead of pg_dump?

5. How does parallel backup (-j option) improve performance, and what are its limitations?

---

## 9. Clean-up

To remove test databases created during exercises:

```sql
-- Connect to postgres database
psql -h localhost -p 5432 -U postgres -d postgres

-- Drop test databases
DROP DATABASE IF EXISTS northwind_restore;
DROP DATABASE IF EXISTS northwind_partial;
DROP DATABASE IF EXISTS northwind_twostep;
DROP DATABASE IF EXISTS test_copy;
DROP DATABASE IF EXISTS test_insert;
DROP DATABASE IF EXISTS test_compressed;

-- Verify cleanup
\l
```

To remove backup files:

```
REM Remove all backup directories (WARNING: deletes all backups)
rmdir /S /Q C:\pg_backups\full
rmdir /S /Q C:\pg_backups\partial
rmdir /S /Q C:\pg_backups\custom
rmdir /S /Q C:\pg_backups\restore_test
rmdir /S /Q C:\pg_backups\automated

REM Or keep backups for reference
REM They're just files and won't affect PostgreSQL
```

**Note:** Keep the original northwind database intact for future labs.

---

## 10. Key takeaways

- pg_dump creates logical backups of single databases; pg_dumpall backs up entire clusters
- Plain SQL format works with psql; custom/directory/tar formats require pg_restore
- Custom format (-F c) is compressed and allows selective restore
- Schema-only (-s) and data-only (-a) backups enable flexible restore strategies
- COPY is faster than INSERT but less portable across database systems
- Compression (-Z) reduces backup size at cost of CPU time
- Parallel backups (-j) speed up large database backups on multi-core systems
- Always test backups by restoring them - a backup you can't restore is useless
- Automate backups with scripts and implement retention policies

---

## 11. Additional resources

**Official PostgreSQL 18 documentation:**
- [pg_dump](https://www.postgresql.org/docs/18/app-pgdump.html)
- [pg_dumpall](https://www.postgresql.org/docs/18/app-pg-dumpall.html)
- [pg_restore](https://www.postgresql.org/docs/18/app-pgrestore.html)
- [Backup and Restore](https://www.postgresql.org/docs/18/backup.html)

---

## 12. Appendices

### Appendix A: Quick reference commands

**Basic backup:**
```
pg_dump -h localhost -p 5432 -U postgres -d dbname -f backup.sql
```

**Custom format (compressed, selective restore):**
```
pg_dump -F c -Z 6 -f backup.backup dbname
```

**Schema only:**
```
pg_dump -s -f schema.sql dbname
```

**Data only:**
```
pg_dump -a -f data.sql dbname
```

**Specific tables:**
```
pg_dump -t table1 -t table2 -f tables.sql dbname
```

**Restore plain SQL:**
```
psql -d dbname -f backup.sql
```

**Restore custom format:**
```
pg_restore -d dbname backup.backup
```

**Selective restore:**
```
pg_restore -d dbname -t tablename backup.backup
```

**Cluster backup:**
```
pg_dumpall -f cluster.sql
```

**Parallel backup:**
```
pg_dump -F d -j 4 -f backup_dir dbname
```

### Appendix B: Backup format comparison

| Format    | Extension | Restore Tool | Compressed | Selective | Parallel |
| --------- | --------- | ------------ | ---------- | --------- | -------- |
| Plain SQL | .sql      | psql         | No*        | No        | No       |
| Custom    | .backup   | pg_restore   | Yes        | Yes       | No       |
| Directory | (dir)     | pg_restore   | Yes        | Yes       | Yes      |
| Tar       | .tar      | pg_restore   | No**       | Yes       | No       |

\* Can pipe through gzip manually  
\** Can compress entire tar file externally

### Appendix C: Common pg_dump options

| Option           | Purpose                 | Example                      |
| ---------------- | ----------------------- | ---------------------------- |
| -F c             | Custom format           | `pg_dump -F c -f db.backup`  |
| -F d             | Directory format        | `pg_dump -F d -f backup_dir` |
| -s               | Schema only             | `pg_dump -s -f schema.sql`   |
| -a               | Data only               | `pg_dump -a -f data.sql`     |
| -t               | Include table           | `pg_dump -t users -t orders` |
| -T               | Exclude table           | `pg_dump -T logs -T audit`   |
| -Z               | Compression level       | `pg_dump -Z 9 -F c`          |
| -j               | Parallel jobs           | `pg_dump -j 4 -F d`          |
| -v               | Verbose output          | `pg_dump -v`                 |
| --inserts        | Use INSERT              | `pg_dump --inserts`          |
| --column-inserts | INSERT with columns     | `pg_dump --column-inserts`   |
| --create         | Include CREATE DATABASE | `pg_dump --create`           |
| --clean          | Include DROP statements | `pg_dump --clean`            |

### Appendix D: Backup size estimates for Northwind

| Backup Type                    | Approximate Size |
| ------------------------------ | ---------------- |
| Plain SQL (COPY)               | 300-400 KB       |
| Plain SQL (INSERT)             | 400-500 KB       |
| Plain SQL (column INSERT)      | 500-600 KB       |
| Custom format (compressed)     | 100-150 KB       |
| Custom format (no compression) | 250-350 KB       |
| Directory format               | 150-200 KB total |
| pg_dumpall (cluster)           | Several MB       |

---

## 13. Answers

### Answer to Question 1

Choose custom format (-F c) when you need selective restore capabilities, automatic compression, or plan to use pg_restore features like parallel restore or partial restoration. Custom format is ideal for production backups because it's compressed (saving storage), allows restoring specific tables without modifying the backup file, and supports features like --jobs for faster restoration. Choose plain SQL when you need human-readable backups, want to edit the backup manually, need to restore using only psql (no pg_restore available), or require portability to non-PostgreSQL databases. Plain SQL is better for documentation, troubleshooting, or version control systems where you want to see what changed between backups.

### Answer to Question 2

The book emphasizes testing backups by restoring them because a backup file might exist and appear valid but could be corrupted, incomplete, or incompatible with your restore environment. Storage media can fail silently, producing files that look correct but contain corrupted data. Network transfers might truncate files, pg_dump might encounter errors that produce partial backups, or you might have incorrect permissions that prevent restoration. Testing restoration confirms not just that you have a file, but that you can actually recover your data when needed. Most data loss disasters occur because backups were never tested, and organizations only discover their backups are unusable when they desperately need them during a real emergency.

### Answer to Question 3

COPY statements are PostgreSQL-specific, load data much faster (optimized bulk loading), and produce smaller backup files because they use a condensed format. However, they're not portable to other database systems. INSERT statements are standard SQL, work on any database system (MySQL, SQL Server, Oracle), and allow manual editing of individual records. However, they're significantly slower to restore (each INSERT is a separate operation) and produce larger backup files. Use COPY for PostgreSQL-to-PostgreSQL migrations or routine backups where performance matters. Use INSERT (especially with --column-inserts) when migrating to different database systems, when backup files need to be human-readable, or when you need maximum portability across platforms.

### Answer to Question 4

Use pg_dumpall when you need to backup the entire PostgreSQL cluster including all databases, roles, tablespaces, and other global objects. This is essential for complete disaster recovery, migrating an entire PostgreSQL installation to new hardware, or setting up a development server that exactly mirrors production. Use pg_dumpall with -g to backup only global objects (roles and tablespaces) when setting up users on a new server before restoring individual database backups. Use pg_dump when you only need a single database, want selective restore capabilities (custom format), need parallel backup/restore, or want to backup different databases on different schedules with different retention policies.

### Answer to Question 5

Parallel backup (-j option) uses multiple CPU cores by assigning different tables to separate worker processes, dramatically reducing backup time for large databases on multi-core systems. Each worker dumps one or more tables independently, utilizing all available CPU cores instead of just one. However, parallel backup only works with directory format (-F d), not custom or plain SQL formats, because it needs to write multiple files simultaneously. The performance improvement depends on having multiple large tables to parallelize—backing up one huge table won't benefit from parallelism. Additionally, parallel backup increases network/disk I/O load, which can impact concurrent database operations, so you should tune the number of jobs based on system resources and production load.