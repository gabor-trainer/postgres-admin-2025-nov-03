# Lab 4: PostgreSQL system catalogs and metadata

**Estimated time:** 45 minutes  
**Difficulty level:** Beginner  
**PostgreSQL version:** 18.x  
**Platform requirements:** Any

---

## 1. Learning objectives

By the end of this lab, you will be able to:

- Query pg_catalog system tables to retrieve database metadata
- Use information_schema views for portable metadata queries
- Understand the differences between pg_catalog and information_schema
- Use psql meta-commands as shortcuts for common metadata queries
- Monitor active database connections and sessions

---

## 2. Prerequisites

**Knowledge prerequisites:**
- Basic SQL SELECT queries
- Understanding of databases, schemas, and tables

**Previous labs:**
- Lab 2 (Installing Northwind database) recommended

**Standard environment checklist:**
- [ ] PostgreSQL 18 installed on Windows (port 5432)
- [ ] Can connect as postgres user
- [ ] Northwind database exists (from Lab 2)

---

## 3. Lab-specific setup

### Verify connectivity

```sql
psql -h localhost -p 5432 -U postgres -d postgres
```

### Create test objects for exercises

```sql
-- Ensure we're in postgres database
SELECT current_database();

-- Create a test database
CREATE DATABASE catalog_test;

-- Connect to catalog_test
\c catalog_test

-- Create a custom schema
CREATE SCHEMA sales;

-- Create tables in different schemas
CREATE TABLE public.test_table (
    id INTEGER PRIMARY KEY,
    name VARCHAR(100),
    created_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE sales.orders (
    order_id INTEGER PRIMARY KEY,
    customer_name VARCHAR(100),
    amount DECIMAL(10,2)
);

-- Verify setup
SELECT current_database();
```

---

## 4. Concept overview

PostgreSQL stores all metadata about database objects in system catalogs. There are two main ways to query this metadata: **pg_catalog** (PostgreSQL-specific system tables) and **information_schema** (SQL standard views). The pg_catalog provides complete PostgreSQL-specific information, while information_schema offers portability across different database systems. Additionally, psql provides meta-commands (starting with backslash) as convenient shortcuts for common catalog queries.

---

## 5. Exercises

### Exercise 1.1: Get all databases

**Platform:** Windows  
**Objective:** Learn to list all databases in the PostgreSQL instance  
**Scenario:** You need to see what databases exist on your server for inventory or backup purposes

**Method 1: Using pg_catalog**

```sql
-- Connect to any database (postgres is fine)
\c postgres

-- Query pg_database catalog
SELECT 
    oid,
    datname AS database_name,
    pg_encoding_to_char(encoding) AS encoding,
    datcollate AS collation,
    pg_size_pretty(pg_database_size(datname)) AS size
FROM pg_catalog.pg_database
ORDER BY datname;
```

**Expected output:**
```
  oid  |  database_name  | encoding |      collation      |  size   
-------+-----------------+----------+---------------------+---------
 16388 | catalog_test    | UTF8     | English_United States.1252 | 8377 kB
     5 | postgres        | UTF8     | English_United States.1252 | 8553 kB
     1 | template0       | UTF8     | English_United States.1252 | 8369 kB
     4 | template1       | UTF8     | English_United States.1252 | 8369 kB
(4 rows)
```

**Explanation:**
- `pg_catalog.pg_database`: System table containing all databases
- `oid`: Internal object identifier
- `datname`: Database name
- `pg_encoding_to_char()`: Converts encoding ID to readable name
- `pg_database_size()`: Returns database size in bytes
- `pg_size_pretty()`: Formats bytes as KB/MB/GB

**Method 2: Using information_schema**

```sql
-- information_schema doesn't have a databases view
-- This is a limitation - use pg_catalog or psql meta-command instead
SELECT 'information_schema does not provide database list' AS note;
```

**Note:** information_schema is database-specific and doesn't show other databases. This is by design for security and SQL standard compliance.

**Method 3: Using psql meta-command**

```sql
\l
-- Or with more details:
\l+
```

**Expected output:**
```
                                                                    List of databases
      Name      |  Owner   | Encoding | Locale Provider |          Collate           |           Ctype            | ICU Locale | ICU Rules |   Access privileges   
----------------+----------+----------+-----------------+----------------------------+----------------------------+------------+-----------+-----------------------
 catalog_test   | postgres | UTF8     | libc            | English_United States.1252 | English_United States.1252 |            |           | 
 northwind      | northwind| UTF8     | libc            | English_United States.1252 | English_United States.1252 |            |           | 
 postgres       | postgres | UTF8     | libc            | English_United States.1252 | English_United States.1252 |            |           | 
 template0      | postgres | UTF8     | libc            | English_United States.1252 | English_United States.1252 |            |           | =c/postgres          +
                |          |          |                 |                            |                            |            |           | postgres=CTc/postgres
 template1      | postgres | UTF8     | libc            | English_United States.1252 | English_United States.1252 |            |           | =c/postgres          +
                |          |          |                 |                            |                            |            |           | postgres=CTc/postgres
(5 rows)
```

**Explanation of psql meta-command:**
- `\l`: Lists all databases with basic information
- `\l+`: Adds size and description columns
- This is the quickest way for interactive use

**Verification:**

```sql
-- Count databases
SELECT COUNT(*) AS database_count FROM pg_catalog.pg_database;
```

---

### Exercise 1.2: Get all schemas in a database

**Platform:** Windows  
**Objective:** Learn to list all schemas within a specific database  
**Scenario:** You need to understand the logical organization of a database

**Method 1: Using pg_catalog**

```sql
-- Connect to catalog_test database
\c catalog_test

-- Query pg_namespace catalog
SELECT 
    oid,
    nspname AS schema_name,
    pg_catalog.pg_get_userbyid(nspowner) AS owner,
    CASE 
        WHEN nspname LIKE 'pg_%' THEN 'System schema'
        WHEN nspname = 'information_schema' THEN 'SQL standard schema'
        ELSE 'User schema'
    END AS schema_type
FROM pg_catalog.pg_namespace
ORDER BY 
    CASE 
        WHEN nspname = 'public' THEN 1
        WHEN nspname NOT LIKE 'pg_%' AND nspname != 'information_schema' THEN 2
        ELSE 3
    END,
    nspname;
```

**Expected output:**
```
  oid  |      schema_name       |  owner   |    schema_type     
-------+------------------------+----------+--------------------
    99 | public                 | postgres | User schema
 16389 | sales                  | postgres | User schema
 13175 | information_schema     | postgres | SQL standard schema
 11    | pg_catalog             | postgres | System schema
    99 | pg_toast               | postgres | System schema
 13174 | pg_temp_1              | postgres | System schema
(6+ rows)
```

**Explanation:**
- `pg_catalog.pg_namespace`: System table for schemas (namespaces)
- `nspname`: Schema name
- `nspowner`: Owner's OID (converted to username with pg_get_userbyid)
- `pg_%`: PostgreSQL system schemas
- `public`: Default schema for user objects

**Method 2: Using information_schema**

```sql
-- Query information_schema.schemata view
SELECT 
    catalog_name,
    schema_name,
    schema_owner,
    CASE 
        WHEN schema_name LIKE 'pg_%' THEN 'System schema'
        WHEN schema_name = 'information_schema' THEN 'SQL standard schema'
        ELSE 'User schema'
    END AS schema_type
FROM information_schema.schemata
ORDER BY schema_name;
```

**Expected output:**
```
 catalog_name |      schema_name       | schema_owner |    schema_type     
--------------+------------------------+--------------+--------------------
 catalog_test | information_schema     | postgres     | SQL standard schema
 catalog_test | pg_catalog             | postgres     | System schema
 catalog_test | pg_toast               | postgres     | System schema
 catalog_test | public                 | postgres     | User schema
 catalog_test | sales                  | postgres     | User schema
(5+ rows)
```

**Explanation:**
- `information_schema.schemata`: SQL standard view for schemas
- More portable across database systems
- Contains less PostgreSQL-specific information than pg_namespace

**Method 3: Using psql meta-command**

```sql
\dn
-- Or with more details:
\dn+
```

**Expected output:**
```
        List of schemas
  Name   |       Owner       
---------+-------------------
 public  | postgres
 sales   | postgres
(2 rows)
```

**Explanation:**
- `\dn`: Lists user schemas (excludes system schemas by default)
- `\dn+`: Adds access privileges and description
- `\dn *`: Shows all schemas including system schemas

**Show all schemas including system:**

```sql
\dn *
```

**Verification:**

```sql
-- Count user schemas (excluding system schemas)
SELECT COUNT(*) AS user_schema_count 
FROM pg_catalog.pg_namespace 
WHERE nspname NOT LIKE 'pg_%' 
  AND nspname != 'information_schema';
```

Expected: 2 (public and sales)

---

### Exercise 1.3: Get all tables in a schema

**Platform:** Windows  
**Objective:** Learn to list all tables within a specific schema  
**Scenario:** You need to inventory tables in the sales schema

**Method 1: Using pg_catalog**

```sql
-- Still connected to catalog_test
SELECT current_database();

-- Query pg_class and pg_namespace catalogs
SELECT 
    n.nspname AS schema_name,
    c.relname AS table_name,
    pg_catalog.pg_get_userbyid(c.relowner) AS owner,
    pg_size_pretty(pg_total_relation_size(c.oid)) AS total_size,
    c.reltuples::bigint AS estimated_rows
FROM pg_catalog.pg_class c
JOIN pg_catalog.pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind = 'r'  -- 'r' = regular table
  AND n.nspname NOT IN ('pg_catalog', 'information_schema')
ORDER BY n.nspname, c.relname;
```

**Expected output:**
```
 schema_name | table_name  |  owner   | total_size | estimated_rows 
-------------+-------------+----------+------------+----------------
 public      | test_table  | postgres | 16 kB      |              0
 sales       | orders      | postgres | 16 kB      |              0
(2 rows)
```

**Explanation:**
- `pg_catalog.pg_class`: Contains all database objects (tables, indexes, views, etc.)
- `relkind = 'r'`: Filters for regular tables only
- Other relkind values: 'v' = view, 'i' = index, 'S' = sequence, 'c' = composite type
- `pg_total_relation_size()`: Returns total size including indexes and TOAST
- `reltuples`: Estimated row count (updated by ANALYZE)

**Get tables in specific schema (sales):**

```sql
SELECT 
    c.relname AS table_name,
    pg_catalog.pg_get_userbyid(c.relowner) AS owner,
    pg_size_pretty(pg_relation_size(c.oid)) AS table_size
FROM pg_catalog.pg_class c
JOIN pg_catalog.pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind = 'r'
  AND n.nspname = 'sales'
ORDER BY c.relname;
```

**Expected output:**
```
 table_name |  owner   | table_size 
------------+----------+------------
 orders     | postgres | 8192 bytes
(1 row)
```

**Method 2: Using information_schema**

```sql
-- Query information_schema.tables view
SELECT 
    table_schema,
    table_name,
    table_type
FROM information_schema.tables
WHERE table_schema NOT IN ('pg_catalog', 'information_schema')
ORDER BY table_schema, table_name;
```

**Expected output:**
```
 table_schema | table_name | table_type  
--------------+------------+-------------
 public       | test_table | BASE TABLE
 sales        | orders     | BASE TABLE
(2 rows)
```

**Get tables in specific schema (sales):**

```sql
SELECT 
    table_schema,
    table_name,
    table_type
FROM information_schema.tables
WHERE table_schema = 'sales'
  AND table_type = 'BASE TABLE'
ORDER BY table_name;
```

**Expected output:**
```
 table_schema | table_name | table_type  
--------------+------------+-------------
 sales        | orders     | BASE TABLE
(1 row)
```

**Explanation:**
- `information_schema.tables`: SQL standard view for tables and views
- `table_type = 'BASE TABLE'`: Regular tables (vs 'VIEW', 'FOREIGN TABLE', etc.)
- Does not include size or row count information

**Method 3: Using psql meta-command**

```sql
-- List tables in current search_path
\dt

-- List tables in specific schema
\dt sales.*

-- List all tables in all schemas
\dt *.*

-- With more details (size, description)
\dt+
```

**Expected output for `\dt`:**
```
           List of relations
 Schema |    Name    | Type  |  Owner   
--------+------------+-------+----------
 public | test_table | table | postgres
(1 row)
```

**Expected output for `\dt sales.*`:**
```
          List of relations
 Schema |  Name  | Type  |  Owner   
--------+--------+-------+----------
 sales  | orders | table | postgres
(1 row)
```

**Explanation:**
- `\dt`: Lists tables in schemas on search_path (usually public)
- `\dt schema.*`: Lists tables in specific schema
- `\dt+`: Adds size and description columns

**Verification:**

```sql
-- Count tables in sales schema
SELECT COUNT(*) AS table_count
FROM information_schema.tables
WHERE table_schema = 'sales' AND table_type = 'BASE TABLE';
```

Expected: 1

---

### Exercise 1.4: Get all columns in a table

**Platform:** Windows  
**Objective:** Learn to retrieve column metadata for a specific table  
**Scenario:** You need to understand the structure of the sales.orders table

**Method 1: Using pg_catalog**

```sql
-- Still connected to catalog_test
SELECT 
    a.attnum AS column_number,
    a.attname AS column_name,
    pg_catalog.format_type(a.atttypid, a.atttypmod) AS data_type,
    a.attnotnull AS not_null,
    pg_catalog.pg_get_expr(d.adbin, d.adrelid) AS default_value,
    col_description(c.oid, a.attnum) AS column_description
FROM pg_catalog.pg_attribute a
JOIN pg_catalog.pg_class c ON c.oid = a.attrelid
JOIN pg_catalog.pg_namespace n ON n.oid = c.relnamespace
LEFT JOIN pg_catalog.pg_attrdef d ON (d.adrelid = a.attrelid AND d.adnum = a.attnum)
WHERE c.relname = 'orders'
  AND n.nspname = 'sales'
  AND a.attnum > 0  -- Exclude system columns
  AND NOT a.attisdropped  -- Exclude dropped columns
ORDER BY a.attnum;
```

**Expected output:**
```
 column_number |  column_name  |     data_type     | not_null | default_value | column_description 
---------------+---------------+-------------------+----------+---------------+--------------------
             1 | order_id      | integer           | t        |               | 
             2 | customer_name | character varying(100) | f   |               | 
             3 | amount        | numeric(10,2)     | f        |               | 
(3 rows)
```

**Explanation:**
- `pg_catalog.pg_attribute`: Contains column information
- `attnum`: Column position (positive numbers are user columns)
- `atttypid`: Data type OID
- `format_type()`: Converts type OID to readable format
- `attnotnull`: True if column has NOT NULL constraint
- `pg_attrdef`: Stores default values
- `col_description()`: Retrieves column comments

**Method 2: Using information_schema**

```sql
-- Query information_schema.columns view
SELECT 
    table_schema,
    table_name,
    ordinal_position,
    column_name,
    data_type,
    character_maximum_length,
    numeric_precision,
    numeric_scale,
    is_nullable,
    column_default
FROM information_schema.columns
WHERE table_schema = 'sales'
  AND table_name = 'orders'
ORDER BY ordinal_position;
```

**Expected output:**
```
 table_schema | table_name | ordinal_position | column_name   |     data_type     | character_maximum_length | numeric_precision | numeric_scale | is_nullable | column_default 
--------------+------------+------------------+---------------+-------------------+--------------------------+-------------------+---------------+-------------+----------------
 sales        | orders     |                1 | order_id      | integer           |                          |                32 |             0 | NO          | 
 sales        | orders     |                2 | customer_name | character varying |                      100 |                   |               | YES         | 
 sales        | orders     |                3 | amount        | numeric           |                          |                10 |             2 | YES         | 
(3 rows)
```

**Explanation:**
- `information_schema.columns`: SQL standard view for columns
- `ordinal_position`: Column order in table
- `character_maximum_length`: For varchar/char types
- `numeric_precision` and `numeric_scale`: For numeric types
- `is_nullable`: 'YES' or 'NO' instead of boolean

**Get columns for public.test_table:**

```sql
SELECT 
    column_name,
    data_type,
    is_nullable,
    column_default
FROM information_schema.columns
WHERE table_schema = 'public'
  AND table_name = 'test_table'
ORDER BY ordinal_position;
```

**Expected output:**
```
 column_name  |          data_type          | is_nullable |     column_default      
--------------+-----------------------------+-------------+-------------------------
 id           | integer                     | NO          | 
 name         | character varying(100)      | YES         | 
 created_date | timestamp without time zone | YES         | CURRENT_TIMESTAMP
(3 rows)
```

**Method 3: Using psql meta-command**

```sql
-- Describe table structure
\d sales.orders

-- Or more concise
\d+ sales.orders
```

**Expected output:**
```
                                      Table "sales.orders"
    Column     |          Type          | Collation | Nullable | Default | Storage  | Stats target | Description 
---------------+------------------------+-----------+----------+---------+----------+--------------+-------------
 order_id      | integer                |           | not null |         | plain    |              | 
 customer_name | character varying(100) |           |          |         | extended |              | 
 amount        | numeric(10,2)          |           |          |         | main     |              | 
Indexes:
    "orders_pkey" PRIMARY KEY, btree (order_id)
```

**Explanation:**
- `\d table_name`: Describes table structure (columns, indexes, constraints)
- `\d+`: Adds storage information and statistics targets
- Shows more context than just columns (indexes, foreign keys, etc.)

**Verification:**

```sql
-- Count columns in sales.orders
SELECT COUNT(*) AS column_count
FROM information_schema.columns
WHERE table_schema = 'sales' AND table_name = 'orders';
```

Expected: 3

---

### Exercise 1.5: Get all active connections

**Platform:** Windows  
**Objective:** Learn to monitor active database connections and sessions  
**Scenario:** You need to see who is connected to your databases and what they're doing

**Method 1: Using pg_catalog**

```sql
-- Connect to postgres database (to see all connections)
\c postgres

-- Query pg_stat_activity catalog view
SELECT 
    pid,
    usename AS username,
    datname AS database,
    client_addr AS client_address,
    client_port,
    application_name,
    backend_start,
    state,
    state_change,
    query_start,
    LEFT(query, 50) AS current_query
FROM pg_catalog.pg_stat_activity
WHERE pid <> pg_backend_pid()  -- Exclude current session
ORDER BY backend_start DESC;
```

**Expected output:**
```
  pid  | username | database |   client_address   | client_port | application_name |         backend_start         | state  |         state_change          |          query_start          |               current_query                
-------+----------+----------+--------------------+-------------+------------------+-------------------------------+--------+-------------------------------+-------------------------------+--------------------------------------------
 12345 | postgres | northwind| 127.0.0.1          |       54321 | psql             | 2025-11-04 10:30:15.123456-05 | idle   | 2025-11-04 10:31:20.654321-05 | 2025-11-04 10:31:20.654321-05 | SELECT * FROM customers WHERE customer_id
 12344 | postgres | postgres | 127.0.0.1          |       54320 | pgAdmin 4        | 2025-11-04 10:25:10.789012-05 | idle   | 2025-11-04 10:30:05.123456-05 | 2025-11-04 10:30:05.123456-05 | SELECT version()
(2 rows)
```

**Explanation:**
- `pg_catalog.pg_stat_activity`: Real-time view of current database connections
- `pid`: Process ID of the backend
- `usename`: Connected user
- `datname`: Connected database
- `client_addr`: IP address of client (NULL for local connections on some systems)
- `application_name`: Connecting application (psql, pgAdmin, etc.)
- `backend_start`: When connection was established
- `state`: Current state (active, idle, idle in transaction, etc.)
- `query`: Current or last query executed
- `pg_backend_pid()`: Returns current session's PID

**Get only active queries:**

```sql
SELECT 
    pid,
    usename,
    datname,
    state,
    NOW() - query_start AS query_duration,
    query
FROM pg_catalog.pg_stat_activity
WHERE state = 'active'
  AND pid <> pg_backend_pid()
ORDER BY query_start;
```

**Get connections by database:**

```sql
SELECT 
    datname AS database,
    COUNT(*) AS connection_count,
    COUNT(*) FILTER (WHERE state = 'active') AS active_queries,
    COUNT(*) FILTER (WHERE state = 'idle') AS idle_connections
FROM pg_catalog.pg_stat_activity
WHERE datname IS NOT NULL
GROUP BY datname
ORDER BY connection_count DESC;
```

**Expected output:**
```
   database   | connection_count | active_queries | idle_connections 
--------------+------------------+----------------+------------------
 postgres     |                2 |              1 |                1
 northwind    |                1 |              0 |                1
 catalog_test |                1 |              0 |                1
(3 rows)
```

**Get long-running queries:**

```sql
SELECT 
    pid,
    usename,
    datname,
    NOW() - query_start AS duration,
    state,
    LEFT(query, 60) AS query_preview
FROM pg_catalog.pg_stat_activity
WHERE state = 'active'
  AND query_start < NOW() - INTERVAL '5 seconds'
  AND pid <> pg_backend_pid()
ORDER BY query_start;
```

**Method 2: Using information_schema**

```sql
-- information_schema does not provide connection information
-- This is a limitation - use pg_stat_activity instead
SELECT 'information_schema does not provide connection monitoring' AS note;
```

**Note:** Connection monitoring is PostgreSQL-specific functionality not covered by the SQL standard, so information_schema doesn't have an equivalent view.

**Method 3: Using psql meta-command**

There is no direct psql meta-command for connections, but you can create a shortcut:

```sql
-- No built-in meta-command, but \watch can help monitor
SELECT 
    pid,
    usename,
    datname,
    state,
    LEFT(query, 40) AS query
FROM pg_stat_activity
WHERE pid <> pg_backend_pid()
ORDER BY backend_start DESC;

-- Add \watch 5 to refresh every 5 seconds
-- Press Ctrl+C to stop
```

**Or create a custom psql variable:**

```sql
-- In psql, you can use \gset to create reusable queries
-- But for monitoring, direct query is typical
```

**Verification:**

```sql
-- Count total connections
SELECT COUNT(*) AS total_connections 
FROM pg_catalog.pg_stat_activity;

-- Count connections to specific database
SELECT COUNT(*) AS catalog_test_connections
FROM pg_catalog.pg_stat_activity
WHERE datname = 'catalog_test';
```

**Terminate a specific connection (if needed):**

```sql
-- Find the PID first
SELECT pid, usename, datname, state 
FROM pg_stat_activity 
WHERE datname = 'catalog_test';

-- Terminate gracefully
SELECT pg_terminate_backend(12345);  -- Replace 12345 with actual PID

-- Or cancel just the query
SELECT pg_cancel_backend(12345);  -- Replace 12345 with actual PID
```

**Expected output:**
```
 pg_terminate_backend 
----------------------
 t
(1 row)
```

---

## 6. Validation checklist

After completing all exercises, verify:

- [ ] Can list all databases using pg_database and \l
- [ ] Can list schemas using pg_namespace and information_schema.schemata
- [ ] Can list tables using pg_class and information_schema.tables
- [ ] Can list columns using pg_attribute and information_schema.columns
- [ ] Can monitor connections using pg_stat_activity
- [ ] Understand differences between pg_catalog and information_schema
- [ ] Know equivalent psql meta-commands for common queries

**Final validation script:**

```sql
-- Connect to postgres
\c postgres

-- Check catalogs exist
SELECT 
    'pg_database' AS catalog,
    COUNT(*) AS rows
FROM pg_catalog.pg_database
UNION ALL
SELECT 'pg_namespace', COUNT(*) FROM pg_catalog.pg_namespace
UNION ALL
SELECT 'pg_class', COUNT(*) FROM pg_catalog.pg_class
UNION ALL
SELECT 'pg_attribute', COUNT(*) FROM pg_catalog.pg_attribute
UNION ALL
SELECT 'pg_stat_activity', COUNT(*) FROM pg_catalog.pg_stat_activity;
```

---

## 7. Troubleshooting guide

### Windows

**Problem:** "permission denied for table pg_stat_activity"
- **Solution:** Connect as postgres or a superuser. Regular users see only their own connections.

**Problem:** "schema 'sales' does not exist"
- **Solution:** Verify you created the test schema in setup. Run: `CREATE SCHEMA sales;`

**Problem:** "relation does not exist" when querying information_schema
- **Solution:** Ensure you're connected to the correct database. information_schema is database-specific.

**Problem:** psql meta-commands not working
- **Solution:** Meta-commands only work in psql interactive sessions, not in SQL scripts or other clients.

**Problem:** Empty results from pg_stat_activity
- **Solution:** Check if you're filtering out your own session. Remove `WHERE pid <> pg_backend_pid()` to see all connections including yours.

**Problem:** Cannot see all databases in information_schema
- **Solution:** This is by design. Use pg_catalog.pg_database or \l instead. information_schema is intentionally database-scoped.

---

## 8. Questions

1. When would you prefer using information_schema over pg_catalog? What are the trade-offs?

2. Why does pg_stat_activity show query text but information_schema doesn't provide connection monitoring?

3. What's the difference between pg_cancel_backend() and pg_terminate_backend() when managing connections?

---

## 9. Clean-up

To remove test objects:

```sql
-- Connect to postgres
\c postgres postgres

-- Terminate any connections to catalog_test
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE datname = 'catalog_test' AND pid <> pg_backend_pid();

-- Drop the test database
DROP DATABASE IF EXISTS catalog_test;

-- Verify cleanup
SELECT datname FROM pg_database WHERE datname = 'catalog_test';
```

Expected: zero rows

---

## 10. Key takeaways

- pg_catalog contains PostgreSQL-specific system tables with complete metadata
- information_schema provides SQL standard views for cross-database compatibility
- information_schema cannot show all databases or connections (by design)
- psql meta-commands (\l, \dn, \dt, \d) are shortcuts for common catalog queries
- pg_stat_activity is essential for monitoring active connections and queries
- pg_catalog provides more detail; information_schema provides portability
- System columns (attnum < 0) are excluded from normal queries

---

## 11. Additional resources

**Official PostgreSQL 18 documentation:**
- [System Catalogs](https://www.postgresql.org/docs/18/catalogs.html)
- [Information Schema](https://www.postgresql.org/docs/18/information-schema.html)
- [Monitoring Views](https://www.postgresql.org/docs/18/monitoring-stats.html)
- [psql meta-commands](https://www.postgresql.org/docs/18/app-psql.html)

---

## 12. Appendices

### Appendix A: Quick reference - Catalog queries

**Databases:**
```sql
-- pg_catalog
SELECT datname FROM pg_database;
-- psql
\l
```

**Schemas:**
```sql
-- pg_catalog
SELECT nspname FROM pg_namespace WHERE nspname !~ '^pg_';
-- information_schema
SELECT schema_name FROM information_schema.schemata;
-- psql
\dn
```

**Tables:**
```sql
-- pg_catalog
SELECT relname FROM pg_class WHERE relkind='r' AND relnamespace=2200;
-- information_schema
SELECT table_name FROM information_schema.tables WHERE table_schema='public';
-- psql
\dt
```

**Columns:**
```sql
-- pg_catalog
SELECT attname FROM pg_attribute WHERE attrelid='tablename'::regclass AND attnum>0;
-- information_schema
SELECT column_name FROM information_schema.columns WHERE table_name='tablename';
-- psql
\d tablename
```

**Connections:**
```sql
-- pg_catalog only
SELECT * FROM pg_stat_activity;
```

### Appendix B: Useful pg_catalog tables

| Table               | Purpose                              |
| ------------------- | ------------------------------------ |
| pg_database         | All databases                        |
| pg_namespace        | All schemas                          |
| pg_class            | All objects (tables, indexes, views) |
| pg_attribute        | All columns                          |
| pg_type             | All data types                       |
| pg_proc             | All functions/procedures             |
| pg_index            | All indexes                          |
| pg_constraint       | All constraints                      |
| pg_stat_activity    | Active connections                   |
| pg_stat_user_tables | Table statistics                     |

### Appendix C: Comparing pg_catalog vs information_schema

| Feature            | pg_catalog                            | information_schema                              |
| ------------------ | ------------------------------------- | ----------------------------------------------- |
| **Portability**    | PostgreSQL only                       | SQL standard (works on MySQL, SQL Server, etc.) |
| **Completeness**   | All PostgreSQL features               | Subset of features                              |
| **Performance**    | Direct system tables (faster)         | Views over system tables (slightly slower)      |
| **Detail level**   | Low-level (OIDs, internal structures) | High-level (user-friendly names)                |
| **Cross-database** | Yes (can see all databases)           | No (database-scoped)                            |
| **Use case**       | PostgreSQL-specific admin tasks       | Portable applications                           |

### Appendix D: Common relkind values in pg_class

| relkind | Description       |
| ------- | ----------------- |
| 'r'     | Regular table     |
| 'i'     | Index             |
| 'S'     | Sequence          |
| 'v'     | View              |
| 'm'     | Materialized view |
| 'c'     | Composite type    |
| 'f'     | Foreign table     |
| 'p'     | Partitioned table |

---

## 13. Answers

### Answer to Question 1

Use information_schema when you need portability across different database systems (MySQL, SQL Server, Oracle) or when writing application code that might run on multiple database platforms. It provides a standard interface and user-friendly column names. Use pg_catalog when you need PostgreSQL-specific features (like OIDs, table sizes, replication status), require cross-database queries, need better performance, or are writing PostgreSQL-specific administration scripts. The trade-off is portability versus completeness: information_schema is portable but limited, while pg_catalog gives complete access but ties you to PostgreSQL.

### Answer to Question 2

information_schema follows the SQL standard, which defines database metadata as database-scoped for security and isolation. Each database's information_schema only shows objects within that specific database, intentionally preventing visibility into other databases or active connections. pg_stat_activity is a PostgreSQL-specific monitoring view that provides server-wide visibility into all connections, query execution, and performance metrics. This capability is essential for database administration but falls outside the SQL standard's scope, so information_schema doesn't include it. For connection monitoring, you must use pg_stat_activity.

### Answer to Question 3

pg_cancel_backend(pid) interrupts the currently running query for that connection but keeps the connection alive—the client receives a query cancellation error but remains connected and can run new queries. pg_terminate_backend(pid) forcibly closes the entire connection, disconnecting the client completely. Use pg_cancel_backend when you want to stop a long-running query but keep the user connected (less disruptive). Use pg_terminate_backend when you need to drop the connection entirely, such as before maintenance or when dropping a database. pg_terminate_backend is more aggressive and should be used carefully as it disrupts the client session.