# Lab 1: PostgreSQL client tools and database connections

**Estimated time:** 60 minutes  
**Difficulty level:** Beginner  
**PostgreSQL version:** 18.x  
**Platform requirements:** Windows mandatory, Linux optional

---

## 2. Learning objectives

By completing this lab, you will be able to:

- Connect to PostgreSQL instances using psql command-line tool with appropriate connection parameters
- Understand and explain the "one connection to one database" concept in PostgreSQL
- Navigate and execute commands using psql meta-commands (\-commands)
- Use pgAdmin to view object definitions and execute SQL scripts
- Switch between databases in both psql and pgAdmin effectively
- Apply best practices for daily database administration tasks using both tools

---

## 3. Prerequisites

**Knowledge prerequisites:**
- Basic SQL SELECT, INSERT, UPDATE, DELETE syntax
- Understanding of database concepts (tables, rows, columns)
- Familiarity with command-line interfaces (Windows CMD or Linux bash)
- Basic understanding of client-server architecture

**Previous labs:**
- None (this is the first lab)

**Environment confirmation checklist:**
- [ ] Windows PostgreSQL instance running on port 5432
- [ ] (Optional) Linux PostgreSQL instance running on port 15432
- [ ] pgAdmin installed and accessible on Windows
- [ ] Both instances registered in pgAdmin (if Linux available)
- [ ] postgres user password known for both instances
- [ ] Remote desktop connection to IQSoft lab computer active

---

## 4. Lab-specific setup

### Connectivity verification - Windows instance

1. Open Command Prompt (CMD) on Windows
2. Test PostgreSQL Windows service status:
```
sc query postgresql-x64-18
```
Expected output should show "RUNNING"

3. Verify psql is in your PATH:
```
psql --version
```
Expected output: `psql (PostgreSQL) 18.x`

4. Test connection (you'll be prompted for password):
```
psql -h localhost -p 5432 -U postgres
```
Expected: You should see the postgres=# prompt

5. Exit psql:
```
\q
```

### Connectivity verification - Linux instance (optional)

1. Open pgAdmin on Windows
2. Expand "Servers" in the left panel
3. Right-click on the Linux server connection → Connect Server
4. Verify successful connection (green icon)

Or via command line (if accessing Linux terminal):
```
psql -h localhost -p 15432 -U postgres
```

### Create lab database and sample data

We'll create a simple database structure for practice. Connect to Windows instance using psql:

```
psql -h localhost -p 5432 -U postgres
```

Once connected, run these commands:

```sql
-- Create a database for this lab
CREATE DATABASE lab01_practice;

-- List all databases to verify
\l
```

You should see lab01_practice in the list. Note that you're still connected to the postgres database.

Now connect to the new database:
```
\c lab01_practice
```

Create sample tables and data:

```sql
-- Create a customers table
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    customer_name VARCHAR(100),
    email VARCHAR(100),
    city VARCHAR(50)
);

-- Create an orders table
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(customer_id),
    order_date DATE,
    total_amount NUMERIC(10,2)
);

-- Insert sample customers
INSERT INTO customers (customer_name, email, city) VALUES
('Alice Johnson', 'alice@example.com', 'New York'),
('Bob Smith', 'bob@example.com', 'Los Angeles'),
('Carol White', 'carol@example.com', 'Chicago');

-- Insert sample orders
INSERT INTO orders (customer_id, order_date, total_amount) VALUES
(1, '2025-01-15', 150.00),
(1, '2025-02-20', 200.00),
(2, '2025-01-10', 75.50),
(3, '2025-03-05', 300.00);

-- Verify the data
SELECT COUNT(*) FROM customers;
SELECT COUNT(*) FROM orders;
```

Expected output:
- 3 customers
- 4 orders

Exit psql:
```
\q
```

### Setup validation

Verify in pgAdmin:
1. Open pgAdmin
2. Expand Windows server → Databases → lab01_practice
3. Expand Schemas → public → Tables
4. You should see `customers` and `orders` tables

---

## 5. Concept overview

PostgreSQL is a client-server database system. Understanding how clients connect to the server is fundamental to effective database administration. In this lab, you'll learn two primary tools: psql (command-line interface) and pgAdmin (graphical interface).

A critical concept in PostgreSQL is that **every connection is made to exactly one specific database**. You cannot query tables from multiple databases in a single query. This is different from some other database systems. When you connect, you must specify which database you're connecting to (or it defaults to a database with the same name as your username). If you need to work with a different database, you must explicitly switch your connection.

Understanding this connection model helps you avoid common mistakes like trying to query a table but being connected to the wrong database. Both psql and pgAdmin operate under this principle, though they present it differently in their interfaces. This lab will make this concept crystal clear through hands-on practice.

---

## 6. Exercises

### Exercise 1.1: Understanding psql connection parameters

**Platform:** Windows (Linux users can follow along on port 15432)

**Objective:** Master the basic psql connection syntax and understand each connection parameter.

**Scenario:** You're a new DBA who needs to connect to different databases throughout the day. Understanding connection parameters is essential for your daily work.

**Tasks:**

1. Open Command Prompt (Windows CMD)

2. First, try connecting WITHOUT specifying a database:
```
psql -h localhost -p 5432 -U postgres
```
Enter the postgres user password when prompted.

**What you should observe:** You're connected, and the prompt shows `postgres=#`

This means you're connected to the `postgres` database (the default database).

3. Check which database you're connected to:
```
SELECT current_database();
```

Expected output:
```
 current_database 
------------------
 postgres
(1 row)
```

4. Exit psql:
```
\q
```

5. Now connect directly to your lab database using the `-d` parameter:
```
psql -h localhost -p 5432 -U postgres -d lab01_practice
```

**What you should observe:** The prompt now shows `lab01_practice=#`

6. Verify you're in the correct database:
```
SELECT current_database();
```

Expected output:
```
 current_database 
------------------
 lab01_practice
(1 row)
```

7. Try to query your customers table:
```
SELECT * FROM customers;
```

**What you should observe:** You see the 3 customers you created in setup.

8. Now exit and reconnect to the postgres database:
```
\q
psql -h localhost -p 5432 -U postgres -d postgres
```

9. Try to query the customers table again:
```
SELECT * FROM customers;
```

**What you should observe:** An error!
```
ERROR:  relation "customers" does not exist
LINE 1: SELECT * FROM customers;
```

**Critical concept:** The customers table exists in the lab01_practice database, NOT in the postgres database. Your connection is to the postgres database, so you cannot see tables from lab01_practice. **One connection = one database.**

**Verification steps:**
- Your prompt shows the database name you're connected to
- current_database() returns the correct database name
- You can only query tables that exist in your currently connected database

---

### Exercise 1.2: Switching databases within psql

**Platform:** Windows (Linux users can follow along on port 15432)

**Objective:** Learn to switch between databases without closing and reopening psql.

**Scenario:** You're troubleshooting an issue and need to quickly check tables in multiple databases. Exiting and reconnecting each time is inefficient.

**Tasks:**

1. Connect to the postgres database:
```
psql -h localhost -p 5432 -U postgres
```

2. Verify your current connection:
```
\conninfo
```

Expected output:
```
You are connected to database "postgres" as user "postgres" on host "localhost" (address "::1") at port "5432".
```

Notice: The `\conninfo` command shows complete connection details including which database.

3. List all available databases:
```
\l
```

**What you should observe:** A list of all databases including postgres, lab01_practice, template0, template1.

4. Switch to lab01_practice database using the `\c` command:
```
\c lab01_practice
```

Expected output:
```
You are now connected to database "lab01_practice" as user "postgres".
```

**Critical observation:** The prompt changed from `postgres=#` to `lab01_practice=#`

5. Verify the connection switch:
```
\conninfo
```

6. Now you can query the customers table:
```
SELECT customer_name, city FROM customers;
```

**What you should observe:** The query works because you're now connected to the correct database.

7. Switch back to postgres database:
```
\c postgres
```

8. Try querying customers again:
```
SELECT * FROM customers;
```

**What you should observe:** The error returns because you switched databases.

9. Practice switching between databases a few times:
```
\c lab01_practice
SELECT COUNT(*) FROM orders;

\c postgres
\l

\c lab01_practice
\conninfo
```

**Key understanding:** 
- `\c database_name` switches your connection to a different database
- You maintain the same user and host, only the database changes
- The prompt always shows your current database
- Each `\c` command creates a NEW connection and closes the old one

**Verification steps:**
- Prompt shows current database name
- \conninfo confirms connection details
- Tables are accessible only when connected to the correct database

**Exit psql:**
```
\q
```

---

### Exercise 1.3: Essential psql meta-commands

**Platform:** Both (Windows on port 5432, Linux on port 15432)

**Objective:** Master the most commonly used psql backslash commands for daily administration tasks.

**Scenario:** As a DBA, you need quick ways to explore database structures, view table definitions, and navigate between databases. The psql meta-commands (backslash commands) are faster than writing SQL queries for these tasks.

**Tasks:**

1. Connect to lab01_practice:
```
psql -h localhost -p 5432 -U postgres -d lab01_practice
```

2. List all tables in the current database:
```
\dt
```

Expected output shows customers and orders tables with their schema, name, type, and owner.

3. Describe the structure of the customers table:
```
\d customers
```

**What you should observe:** Complete table definition including columns, data types, constraints, and indexes.

4. Get more detailed information about customers (includes storage details):
```
\d+ customers
```

5. List all schemas in the current database:
```
\dn
```

**What you should observe:** At minimum, you'll see the `public` schema.

6. View all databases with sizes:
```
\l+
```

**What you should observe:** Databases listed with their size, tablespace, and description.

7. List all roles (users):
```
\du
```

**What you should observe:** The postgres superuser and potentially other roles.

8. View your current connection info (we used this before):
```
\conninfo
```

9. Display help for psql commands:
```
\?
```

**What you should observe:** A comprehensive list of all backslash commands. Use Space to scroll, 'q' to quit the help.

10. Get help for SQL commands:
```
\h
```

Type 'q' to exit help.

11. Get specific help for CREATE TABLE:
```
\h CREATE TABLE
```

12. Enable query timing (useful for performance testing):
```
\timing
```

Now run a query:
```
SELECT COUNT(*) FROM orders;
```

**What you should observe:** The query result PLUS the time it took to execute (e.g., "Time: 1.234 ms").

13. Toggle timing off:
```
\timing
```

14. Change the output format to expanded display (useful for wide tables):
```
\x
```

Now query customers:
```
SELECT * FROM customers WHERE customer_id = 1;
```

**What you should observe:** Instead of traditional row format, data is displayed vertically (field | value pairs).

15. Toggle back to normal display:
```
\x
```

**Verification steps:**
- \dt shows your tables
- \d table_name reveals table structure
- \l lists all databases
- \timing shows query execution time
- \x toggles between display formats

**Linux users:** Connect to your Linux instance and repeat these commands:
```
psql -h localhost -p 15432 -U postgres -d postgres
\c postgres
\dt
\l
```

The commands work identically on both platforms.

---

### Exercise 1.4: Advanced psql command-line switches and scripting

**Platform:** Both (Windows primary, Linux optional comparison)

**Objective:** Learn command-line options for non-interactive use and scripting scenarios.

**Scenario:** You need to automate routine checks and execute SQL scripts from the command line without interactive sessions. This is essential for scheduled maintenance tasks and automation.

**Tasks:**

1. Execute a single SQL command from the command line (not in psql):

Windows CMD:
```
psql -h localhost -p 5432 -U postgres -d lab01_practice -c "SELECT COUNT(*) FROM customers;"
```

**What you should observe:** The query executes and returns results, then psql exits automatically.

2. Execute multiple commands using multiple -c options:
```
psql -h localhost -p 5432 -U postgres -d lab01_practice -c "SELECT COUNT(*) FROM customers;" -c "SELECT COUNT(*) FROM orders;"
```

3. Create a SQL script file. Open Notepad and create `C:\temp\lab01_query.sql` with this content:

```sql
-- Lab 01 test query script
SELECT 'Customer Report' AS report_type;
SELECT customer_name, email FROM customers ORDER BY customer_name;
SELECT 'Order Report' AS report_type;
SELECT order_id, order_date, total_amount FROM orders ORDER BY order_date;
```

Save the file.

4. Execute the script file from command line:
```
psql -h localhost -p 5432 -U postgres -d lab01_practice -f C:\temp\lab01_query.sql
```

**What you should observe:** All queries in the file execute in sequence.

5. Output results to a file:
```
psql -h localhost -p 5432 -U postgres -d lab01_practice -f C:\temp\lab01_query.sql -o C:\temp\lab01_output.txt
```

6. View the output file:
```
type C:\temp\lab01_output.txt
```

7. Use quiet mode (suppress extra output - useful for scripts):
```
psql -h localhost -p 5432 -U postgres -d lab01_practice -q -c "SELECT customer_name FROM customers;"
```

**What you should observe:** Cleaner output without connection messages.

8. Combine multiple options for production scripts:
```
psql -h localhost -p 5432 -U postgres -d lab01_practice -q -t -A -F"," -c "SELECT customer_name, email FROM customers;" -o C:\temp\customers.csv
```

Explanation of switches:
- `-q`: quiet mode
- `-t`: tuples only (no headers)
- `-A`: unaligned output
- `-F","`: field separator (comma)
- `-o`: output file

9. View the CSV file:
```
type C:\temp\customers.csv
```

**What you should observe:** Clean CSV format suitable for import into other tools.

10. List available databases without entering interactive mode:
```
psql -h localhost -p 5432 -U postgres -l
```

**Verification steps:**
- `-c` executes single commands
- `-f` executes script files
- `-o` redirects output to files
- `-q -t -A -F` options create clean, parseable output
- `-l` lists databases

**Linux users (optional comparison):**

Create script file:
```bash
cat > /tmp/lab01_query.sql << 'EOF'
SELECT 'Customer Report' AS report_type;
SELECT customer_name, email FROM customers ORDER BY customer_name;
EOF
```

Execute it:
```bash
psql -h localhost -p 15432 -U postgres -d lab01_practice -f /tmp/lab01_query.sql
```

Note: The concepts are identical; only file paths differ between Windows and Linux.

---

### Exercise 1.5: pgAdmin interface and the one-connection-one-database concept

**Platform:** Windows

**Objective:** Understand how pgAdmin manages database connections and how to view SQL definitions of existing objects.

**Scenario:** pgAdmin is your primary GUI tool for database administration. Understanding how it handles connections and displays object metadata is crucial for efficient database management.

**Tasks:**

1. Open pgAdmin on Windows.

2. In the left Browser panel, expand:
   - Servers
   - Your Windows PostgreSQL server
   - Databases

**What you should observe:** Multiple databases listed (postgres, lab01_practice, template0, template1).

3. Click on the postgres database (don't expand it yet).

4. Look at the top of the pgAdmin window at the "Dashboard" tab.

**Critical observation:** The dashboard shows metrics for the **postgres database only**. Even though you can see other databases in the tree, you're viewing information about the currently selected database.

5. Now click on lab01_practice database.

**What you should observe:** The dashboard updates to show metrics for lab01_practice. This demonstrates the "one active database context" concept.

6. Expand lab01_practice → Schemas → public → Tables.

7. Right-click on the **customers** table → Properties.

**What you should observe:** A dialog showing table properties including columns, constraints, etc.

8. Close the Properties dialog.

9. Right-click on the **customers** table → View/Edit Data → All Rows.

**What you should observe:** A Query Tool opens showing the data from customers table.

**Critical concept check:** Look at the title of the Query Tool tab. It shows: `lab01_practice/public.customers`

This indicates you're connected to the **lab01_practice** database.

10. In the same Query Tool window, try to query a non-existent table:

In the SQL Editor area, clear any existing query and type:
```sql
SELECT * FROM fake_table;
```

Press F5 to execute.

**What you should observe:** Error: `relation "fake_table" does not exist`

11. Now try to query a table from a DIFFERENT database. Type:
```sql
SELECT * FROM postgres.public.pg_database;
```

Press F5.

**What you should observe:** Error! You cannot use database_name.schema.table syntax in PostgreSQL. **Your connection is to lab01_practice, so you cannot access postgres database tables.**

12. Now let's open a new query tool connected to a different database. 

Close the current Query Tool tab.

13. In the Browser panel, click on the **postgres** database (not lab01_practice).

14. Click Tools menu → Query Tool (or click the Query Tool icon in toolbar).

**What you should observe:** A new Query Tool opens. Look at the title bar - it should indicate connection to `postgres` database.

15. In this new Query Tool, type:
```sql
SELECT * FROM customers;
```

Press F5.

**What you should observe:** Error: `relation "customers" does not exist`

Why? Because this Query Tool is connected to the **postgres** database, and customers table exists in **lab01_practice**.

16. Now demonstrate viewing SQL definitions. In the Browser panel, navigate back to:
lab01_practice → Schemas → public → Tables → customers

17. Right-click on customers → Properties.

18. Click on the **SQL** tab in the Properties dialog.

**What you should observe:** The complete CREATE TABLE statement used to create this table.

```sql
CREATE TABLE IF NOT EXISTS public.customers
(
    customer_id integer NOT NULL DEFAULT nextval('customers_customer_id_seq'::regclass),
    customer_name character varying(100) COLLATE pg_catalog."default",
    email character varying(100) COLLATE pg_catalog."default",
    city character varying(50) COLLATE pg_catalog."default",
    CONSTRAINT customers_pkey PRIMARY KEY (customer_id)
)
...
```

This is incredibly useful for:
- Learning proper PostgreSQL syntax
- Recreating objects in other databases
- Understanding how existing objects were created
- Troubleshooting

19. Close the Properties dialog.

20. Practice with another object. Expand Tables and right-click on **orders** → Properties → SQL tab.

**Verification steps:**
- Dashboard updates based on selected database
- Query Tool title bar shows connected database
- Cannot query tables from different database in same connection
- Properties → SQL tab shows object creation statements
- Each Query Tool window is connected to exactly one database

**Key understanding:** pgAdmin allows you to see multiple databases in the tree view, but when you query or interact with objects, you're always working within ONE database context. This is the same principle as psql but visualized differently.

---

### Exercise 1.6: Running SQL scripts in pgAdmin and best practices

**Platform:** Windows

**Objective:** Learn to write and execute SQL scripts effectively in pgAdmin, understanding transaction behavior and result handling.

**Scenario:** You've received SQL scripts from colleagues or documentation. You need to execute them safely in pgAdmin and verify results.

**Tasks:**

1. Open pgAdmin and ensure you're working with the lab01_practice database.

2. Click Tools → Query Tool to open a new query window connected to lab01_practice.

3. In the SQL Editor (top pane), type the following script:

```sql
-- Script to add new customer and orders
-- Remember: This entire script executes within ONE database connection

-- First, let's add a new customer
INSERT INTO customers (customer_name, email, city) 
VALUES ('David Brown', 'david@example.com', 'Seattle');

-- Get the customer_id of the newly inserted customer
-- (In a real script, you'd use RETURNING or a transaction)
SELECT customer_id, customer_name 
FROM customers 
WHERE customer_name = 'David Brown';

-- Add an order for this customer
-- Note: We're assuming customer_id = 4 based on previous inserts
INSERT INTO orders (customer_id, order_date, total_amount)
VALUES (4, CURRENT_DATE, 125.75);

-- Verify all customers
SELECT COUNT(*) as total_customers FROM customers;

-- Verify all orders
SELECT COUNT(*) as total_orders FROM orders;
```

4. Before executing, let's understand the Query Tool interface:

**Look at the toolbar:**
- Execute button (▶ or F5): Runs the entire script or selected text
- Explain button: Shows query execution plan
- Download button: Save results to CSV
- Clear button: Clears the editor

5. Press **F5** (or click Execute button).

**What you should observe:**
- The script executes all statements in sequence
- The bottom pane (Data Output) shows multiple result tabs
- Each SELECT statement creates a new result tab
- INSERT statements show "INSERT 0 1" in the Messages tab

6. Click through the result tabs at the bottom to see each query result.

**Critical concept:** All statements executed in sequence within the **same connection** to **lab01_practice** database. This is similar to running multiple commands in psql - they all execute against the same database.

7. Now let's demonstrate the importance of database context. Click on the SQL Editor and clear it (Edit → Clear Query or Ctrl+L).

8. Type a script that tries to access multiple databases:

```sql
-- This will fail!
SELECT * FROM lab01_practice.public.customers;  -- Trying to use database.schema.table
```

9. Press F5.

**What you should observe:** Error! PostgreSQL doesn't support `database.schema.table` syntax. You must be connected to the correct database.

10. Clear the editor and let's work with a proper multi-statement script with explanation comments:

```sql
-- Customer and Order Report
-- Connection: lab01_practice database
-- Purpose: Generate summary reports

-- Part 1: Customer summary
SELECT 'Active Customers by City' AS report_section;

SELECT 
    city,
    COUNT(*) as customer_count
FROM customers
GROUP BY city
ORDER BY customer_count DESC;

-- Part 2: Order value analysis
SELECT 'Order Value Summary' AS report_section;

SELECT 
    MIN(total_amount) as minimum_order,
    MAX(total_amount) as maximum_order,
    AVG(total_amount) as average_order,
    SUM(total_amount) as total_revenue
FROM orders;

-- Part 3: Recent orders with customer info
SELECT 'Recent Orders' AS report_section;

SELECT 
    o.order_id,
    c.customer_name,
    o.order_date,
    o.total_amount
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
ORDER BY o.order_date DESC;
```

11. Press F5 to execute the complete script.

**What you should observe:**
- Multiple result tabs appear
- Each SELECT creates a separate result tab
- You can click between tabs to view different results
- This is efficient for running complex reports

12. Let's save this script for future use. Click File → Save (or Ctrl+S).

Save as: `C:\temp\lab01_report.sql`

13. Close the Query Tool tab.

14. Now reopen your saved script. In pgAdmin:
- Tools → Query Tool
- File → Open (or Ctrl+O)
- Browse to `C:\temp\lab01_report.sql`
- Click Open

**What you should observe:** Your script loads into the editor.

15. Execute it again with F5 to verify it works.

16. Let's practice selecting and executing partial scripts. In the editor, use your mouse to **select only these lines:**

```sql
SELECT 
    city,
    COUNT(*) as customer_count
FROM customers
GROUP BY city
ORDER BY customer_count DESC;
```

17. With the text selected, press F5.

**What you should observe:** Only the selected query executes! This is useful for testing individual queries within larger scripts.

18. Final best practice: Let's demonstrate transaction awareness. Clear the editor and type:

```sql
-- Transaction example
BEGIN;

UPDATE customers 
SET city = 'San Francisco' 
WHERE customer_name = 'Alice Johnson';

SELECT customer_name, city FROM customers WHERE customer_name = 'Alice Johnson';

-- Don't commit yet - we're just testing
ROLLBACK;

-- Verify the change was rolled back
SELECT customer_name, city FROM customers WHERE customer_name = 'Alice Johnson';
```

19. Execute with F5.

**What you should observe:**
- First SELECT shows city = 'San Francisco'
- After ROLLBACK, second SELECT shows original city = 'New York'
- The change was not permanently saved

**Verification steps:**
- F5 executes entire script or selected text
- Multiple SELECTs create multiple result tabs
- Scripts can be saved and reopened
- Transactions work as expected
- All commands execute against ONE connected database

**Key takeaways for pgAdmin scripting:**
- One Query Tool = one connection = one database
- To work with different databases, open multiple Query Tool windows
- F5 runs entire script or selected text
- Always verify which database you're connected to (check window title)
- Use BEGIN/COMMIT/ROLLBACK for safe testing

---

## 7. Validation checklist

Verify you've successfully completed the lab:

- [ ] Can connect to PostgreSQL using psql with -h, -p, -U, -d parameters
- [ ] Understand that prompt shows currently connected database
- [ ] Can switch databases using \c command in psql
- [ ] Can list databases with \l and tables with \dt
- [ ] Can view table structure with \d tablename
- [ ] Can execute single commands with psql -c
- [ ] Can execute script files with psql -f
- [ ] Understand that one Query Tool window in pgAdmin = one database connection
- [ ] Can view SQL definitions using Properties → SQL tab
- [ ] Can execute scripts in pgAdmin with F5
- [ ] Can execute partial scripts by selecting text and pressing F5
- [ ] Created lab01_practice database with customers and orders tables

**Validation queries:**

Connect to lab01_practice and run:
```sql
-- Should return 4 (including David Brown added in exercise 1.6)
SELECT COUNT(*) FROM customers;

-- Should return 5 (including order added in exercise 1.6)
SELECT COUNT(*) FROM orders;

-- Should show you're in lab01_practice
SELECT current_database();
```

Expected results:
- 4 customers
- 5 orders
- current_database = lab01_practice

---

## 8. Troubleshooting guide

### Windows-specific issues

**Problem:** "psql: command not found" or "psql is not recognized"

**Solution:**
- PostgreSQL bin directory not in PATH
- Check if psql.exe exists: `dir "C:\Program Files\PostgreSQL\18\bin\psql.exe"`
- Use full path: `"C:\Program Files\PostgreSQL\18\bin\psql" -h localhost -p 5432 -U postgres`
- Or temporarily add to PATH: `set PATH=%PATH%;C:\Program Files\PostgreSQL\18\bin`

**Problem:** "password authentication failed for user postgres"

**Solution:**
- Verify password with instructor
- Check pg_hba.conf allows password authentication (should be pre-configured)
- Ensure you're typing password correctly (it won't display as you type)

**Problem:** "could not connect to server: Connection refused"

**Solution:**
- Check service status: `sc query postgresql-x64-18`
- If stopped, start it: `sc start postgresql-x64-18`
- Verify port: `netstat -an | findstr 5432`
- Check Windows Firewall isn't blocking port 5432

**Problem:** "relation does not exist" when querying a table you know exists

**Solution:**
- Verify which database you're connected to: `SELECT current_database();`
- Check prompt: `postgres=#` vs `lab01_practice=#`
- Switch to correct database: `\c lab01_practice`
- Verify table exists: `\dt`

### Linux-specific issues (optional)

**Problem:** Connection fails on port 15432

**Solution:**
- Linux instance uses non-standard port 15432, not 5432
- Always specify: `psql -h localhost -p 15432 -U postgres`
- Verify service: `sudo systemctl status postgresql`
- Check listening ports: `sudo ss -tlnp | grep 15432`

**Problem:** Permission denied when creating files

**Solution:**
- Use /tmp for temporary files: `/tmp/lab01_query.sql`
- Or ensure you have write permissions to the directory
- Check with: `ls -la /path/to/directory`

### pgAdmin issues

**Problem:** "Could not connect to server" in pgAdmin

**Solution:**
- Right-click server → Properties
- Check host is "localhost"
- Check port is 5432 (Windows) or 15432 (Linux)
- Verify password is correct
- Test connection using psql first to isolate issue

**Problem:** Query Tool shows wrong database in title bar

**Solution:**
- Close Query Tool
- Click on the correct database in Browser panel first
- Then open new Query Tool
- Verify title bar shows correct database

**Problem:** Cannot see SQL tab in Properties dialog

**Solution:**
- Ensure you're right-clicking on the object (table, view, etc.)
- Click Properties, then look for tabs at top of dialog
- SQL tab is usually last tab
- Try closing and reopening Properties dialog

**Problem:** F5 does nothing when pressed in Query Tool

**Solution:**
- Ensure Query Tool window has focus (click in SQL editor)
- Try clicking the Execute button (▶) instead
- Check if script has syntax errors (red underline)
- Try with a simple query: `SELECT 1;`

### General connection issues

**Problem:** "Database does not exist"

**Solution:**
- List available databases: `psql -h localhost -p 5432 -U postgres -l`
- Verify lab01_practice exists
- If missing, recreate using setup section commands
- Check spelling of database name (case-sensitive on Linux)

**Problem:** Slow query execution

**Solution:**
- Enable timing: `\timing` in psql
- Check if connected to wrong instance (Windows vs Linux)
- Verify network connectivity: `ping localhost`
- Check PostgreSQL logs for errors

**Problem:** "Too many connections"

**Solution:**
- Close unused psql sessions with `\q`
- Close unused Query Tool tabs in pgAdmin
- Check active connections: `SELECT count(*) FROM pg_stat_activity;`
- Wait a few minutes for idle connections to timeout

---

## 9. Questions

1. **Database connection architecture:** When you execute a SQL query in PostgreSQL, you must be connected to a specific database. Explain why PostgreSQL enforces this "one connection to one database" model. What are the architectural advantages and potential limitations of this design compared to systems that allow cross-database queries? Consider scenarios where you might need data from multiple databases and discuss how you would approach this in PostgreSQL.

2. **Connection parameter decisions:** You're setting up a script that will run nightly to generate reports from PostgreSQL. The script needs to connect to three different databases on the same server. Explain how you would structure this script considering the one-connection-per-database principle. What connection parameters would you explicitly specify in each connection and why? Discuss the trade-offs between opening multiple connections versus reconnecting as needed.

3. **psql versus pgAdmin decision-making:** As a database administrator, you have both psql command-line tool and pgAdmin GUI available. For each of the following scenarios, explain which tool you would prefer to use and why: (a) Investigating why a scheduled batch job failed at 3 AM, (b) Training a new developer on query optimization, (c) Automating weekly maintenance tasks, (d) Creating a complex query with multiple joins for a one-time report. Justify your choices considering factors like efficiency, reproducibility, and learning curve.

4. **Meta-command efficiency:** The psql meta-commands (like \dt, \d, \l) are convenient shortcuts, but they're ultimately wrappers around SQL queries to system catalogs. Explain why a DBA should learn these meta-commands rather than writing the equivalent SQL queries to pg_catalog tables. When might you prefer to write the SQL query directly instead of using the meta-command? Consider scenarios involving automation, scripting, and troubleshooting.

5. **Script execution analysis:** You have a SQL script file containing 50 INSERT statements and 10 SELECT statements. When you execute this script using `psql -f script.sql`, what happens if the 30th INSERT fails due to a constraint violation? Explain the behavior in terms of transaction management. How would you modify the script execution approach to ensure either all changes succeed or none do? Discuss the implications of implicit versus explicit transaction control.

6. **Object definition exploration:** In pgAdmin, you can view the SQL definition of existing database objects (tables, views, functions) using the Properties → SQL tab. Explain why this capability is valuable for database administration and development work. Describe three specific scenarios where examining these SQL definitions would help you solve real-world problems. How might the generated SQL differ from what you'd write manually, and why?

7. **Performance implications:** You need to extract data from three tables in your database and combine the results for a report. You're deciding between: (a) Opening three separate psql connections and running one query in each, then combining results manually, or (b) Opening one connection and writing a query with joins. Analyze the performance implications of each approach considering connection overhead, query optimization, network latency, and server resource usage. Which would you choose and under what conditions might the other approach be preferable?

8. **Error interpretation:** A junior DBA shows you this error message: "ERROR: relation 'customers' does not exist" but insists that they just created the customers table five minutes ago. Walk through your troubleshooting methodology considering the "one connection, one database" principle. What questions would you ask? What commands would you run to diagnose the issue? Explain how this type of error relates to database context and connection state.

---

## 10. Clean-up

The lab01_practice database and its contents should be kept for potential use in future labs. However, clean up temporary files:

### Windows cleanup

1. Delete temporary files created during the lab:
```
del C:\temp\lab01_query.sql
del C:\temp\lab01_output.txt
del C:\temp\customers.csv
del C:\temp\lab01_report.sql
```

2. Verify cleanup:
```
dir C:\temp\lab01*
```

Expected: "File Not Found"

### Optional Linux cleanup

If you created files on Linux:
```bash
rm /tmp/lab01_query.sql
```

### Database cleanup (optional)

If you want to remove the lab database entirely:

1. Connect to postgres database (not lab01_practice):
```
psql -h localhost -p 5432 -U postgres -d postgres
```

2. Drop the lab database:
```sql
DROP DATABASE lab01_practice;
```

3. Verify:
```
\l
```

**Warning:** Only drop the lab database if instructed. You may need it for future labs.

### Verify clean state

In pgAdmin:
- Check that temporary query tabs are closed
- Servers should still be registered and functional
- Verify you can still connect to postgres database

In psql:
```
psql -h localhost -p 5432 -U postgres -l
```

Expected: List of databases (postgres, template0, template1, and optionally lab01_practice if not dropped)

---

## 11. Key takeaways

- **One connection equals one database**: Every PostgreSQL client connection is tied to exactly one database. You cannot query across databases without switching connections or using advanced techniques like dblink or foreign data wrappers.

- **The prompt is your friend**: In psql, the prompt always shows your current database (e.g., `lab01_practice=#`). In pgAdmin, the Query Tool window title shows the connected database. Always verify before executing queries.

- **psql connection syntax**: `psql -h hostname -p port -U username -d database` - all parameters are important, especially -d to specify the target database.

- **Essential psql meta-commands to remember**:
  - `\l` - list databases
  - `\c database_name` - switch to different database
  - `\dt` - list tables in current database
  - `\d table_name` - describe table structure
  - `\q` - quit psql
  - `\?` - help with psql commands
  - `\conninfo` - show current connection details

- **Non-interactive psql usage**: Use `-c` for single commands, `-f` for script files, `-o` for output redirection. Essential for automation and scheduling.

- **pgAdmin Query Tool management**: Each Query Tool window is a separate connection to ONE database. Opening multiple Query Tools to different databases creates multiple independent connections.

- **View object SQL definitions**: In pgAdmin, right-click any object → Properties → SQL tab reveals the CREATE statement. This is invaluable for learning, debugging, and recreating objects.

- **Script execution**: F5 executes the entire script or selected text in pgAdmin. Always verify which database the Query Tool is connected to before executing.

- **Transaction awareness**: Both psql and pgAdmin operate within transaction contexts. Use BEGIN/COMMIT/ROLLBACK explicitly for safer operations, especially during testing.

- **Platform consistency**: psql commands work identically on Windows and Linux. Only file paths and service management differ. Connection principles are universal.

- **Common mistake to avoid**: Trying to query a table while connected to the wrong database. Always check your connection context first with `SELECT current_database();` or by looking at the prompt/title bar.

---

## 12. Additional resources

### Official PostgreSQL 18 documentation

- **psql command-line tool reference**:  
  https://www.postgresql.org/docs/18/app-psql.html  
  Complete documentation of all psql meta-commands and options

- **Client connection parameters**:  
  https://www.postgresql.org/docs/18/libpq-connect.html  
  Detailed explanation of connection strings and environment variables

- **System catalogs** (what psql meta-commands query):  
  https://www.postgresql.org/docs/18/catalogs.html  
  Understanding pg_catalog tables helps you create custom queries

- **PostgreSQL client programs overview**:  
  https://www.postgresql.org/docs/18/reference-client.html  
  Other useful command-line tools beyond psql

### pgAdmin documentation

- **pgAdmin Query Tool**:  
  https://www.pgadmin.org/docs/pgadmin4/latest/query_tool.html  
  Complete guide to the Query Tool interface and features

- **pgAdmin connection management**:  
  https://www.pgadmin.org/docs/pgadmin4/latest/connecting.html  
  How pgAdmin manages server connections

### Platform-specific resources

- **PostgreSQL on Windows**:  
  https://www.postgresql.org/docs/18/install-windows.html  
  Windows-specific installation and configuration

- **PostgreSQL on Ubuntu/Debian**:  
  https://www.postgresql.org/docs/18/install-debian.html  
  Linux-specific setup and service management

### Best practices articles

- **Working with psql effectively** (Planet PostgreSQL blog):  
  https://www.postgresql.org/about/news/

- **pgAdmin tips and tricks**:  
  Search pgAdmin.org blog for productivity tips

### Connection security

- **Client authentication** (pg_hba.conf):  
  https://www.postgresql.org/docs/18/auth-pg-hba-conf.html  
  Understanding authentication methods and security

---

## 13. Appendices

### Appendix A: Quick reference - psql meta-commands

| Command         | Description                       | Example                        |
| --------------- | --------------------------------- | ------------------------------ |
| `\l`            | List all databases                | `\l` or `\l+` (with details)   |
| `\c dbname`     | Connect to database               | `\c lab01_practice`            |
| `\dt`           | List tables                       | `\dt` or `\dt+` (with details) |
| `\d tablename`  | Describe table                    | `\d customers`                 |
| `\d+ tablename` | Describe table (detailed)         | `\d+ customers`                |
| `\dn`           | List schemas                      | `\dn`                          |
| `\du`           | List roles/users                  | `\du`                          |
| `\dx`           | List extensions                   | `\dx`                          |
| `\df`           | List functions                    | `\df`                          |
| `\dv`           | List views                        | `\dv`                          |
| `\di`           | List indexes                      | `\di`                          |
| `\conninfo`     | Show connection info              | `\conninfo`                    |
| `\q`            | Quit psql                         | `\q`                           |
| `\?`            | Help with psql commands           | `\?`                           |
| `\h`            | Help with SQL commands            | `\h CREATE TABLE`              |
| `\timing`       | Toggle query timing               | `\timing`                      |
| `\x`            | Toggle expanded output            | `\x`                           |
| `\i filename`   | Execute commands from file        | `\i script.sql`                |
| `\o filename`   | Send output to file               | `\o results.txt`               |
| `\! command`    | Execute shell command (Linux/Mac) | `\! ls`                        |

### Appendix B: Connection strings reference

**Windows instance (default configuration):**
```bash
# Full explicit connection
psql -h localhost -p 5432 -U postgres -d postgres

# Connect to specific database
psql -h localhost -p 5432 -U postgres -d lab01_practice

# Using connection URI format
psql postgresql://postgres@localhost:5432/lab01_practice
```

**Linux instance (custom port 15432):**
```bash
# Full explicit connection
psql -h localhost -p 15432 -U postgres -d postgres

# Connect to specific database
psql -h localhost -p 15432 -U postgres -d lab01_practice

# Using connection URI format
psql postgresql://postgres@localhost:15432/lab01_practice
```

**Environment variables (alternative method):**
```bash
# Set connection defaults
set PGHOST=localhost
set PGPORT=5432
set PGUSER=postgres
set PGDATABASE=lab01_practice

# Then simply:
psql
```

### Appendix C: Command-line execution patterns

**Single command execution:**
```bash
psql -h localhost -p 5432 -U postgres -d lab01_practice -c "SELECT COUNT(*) FROM customers;"
```

**Multiple commands:**
```bash
psql -h localhost -p 5432 -U postgres -d lab01_practice -c "SELECT COUNT(*) FROM customers;" -c "SELECT COUNT(*) FROM orders;"
```

**Script file execution:**
```bash
psql -h localhost -p 5432 -U postgres -d lab01_practice -f C:\scripts\report.sql
```

**Output to file:**
```bash
psql -h localhost -p 5432 -U postgres -d lab01_practice -c "SELECT * FROM customers;" -o C:\output\customers.txt
```

**CSV export:**
```bash
psql -h localhost -p 5432 -U postgres -d lab01_practice -q -t -A -F"," -c "SELECT * FROM customers;" -o C:\output\customers.csv
```

**List databases without interactive mode:**
```bash
psql -h localhost -p 5432 -U postgres -l
```

### Appendix D: pgAdmin keyboard shortcuts

| Shortcut     | Action                                | Context      |
| ------------ | ------------------------------------- | ------------ |
| F5           | Execute query/script                  | Query Tool   |
| F7           | Execute EXPLAIN                       | Query Tool   |
| F8           | Execute EXPLAIN ANALYZE               | Query Tool   |
| Ctrl+S       | Save script                           | Query Tool   |
| Ctrl+O       | Open script                           | Query Tool   |
| Ctrl+L       | Clear query editor                    | Query Tool   |
| Ctrl+Space   | Autocomplete                          | Query Tool   |
| Ctrl+/       | Comment/uncomment                     | Query Tool   |
| Ctrl+Shift+C | Copy with headers                     | Results grid |
| F6           | Switch between SQL editor and results | Query Tool   |

### Appendix E: Sample data queries

These queries work if you've completed the lab setup:

```sql
-- Count records
SELECT COUNT(*) FROM customers;
SELECT COUNT(*) FROM orders;

-- Join customers and orders
SELECT 
    c.customer_name,
    c.city,
    COUNT(o.order_id) as order_count,
    COALESCE(SUM(o.total_amount), 0) as total_spent
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.customer_name, c.city
ORDER BY total_spent DESC;

-- Recent orders
SELECT 
    o.order_date,
    c.customer_name,
    o.total_amount
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
ORDER BY o.order_date DESC
LIMIT 5;

-- Customers without orders
SELECT 
    c.customer_name,
    c.email
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_id IS NULL;
```

### Appendix F: Troubleshooting checklist

When encountering connection or query issues, check these in order:

**Connection troubleshooting:**
- [ ] Service running? (Windows: `sc query postgresql-x64-18`, Linux: `systemctl status postgresql`)
- [ ] Correct port? (5432 for Windows, 15432 for Linux)
- [ ] Correct hostname? (localhost or 127.0.0.1)
- [ ] Correct username? (postgres)
- [ ] Correct password?
- [ ] Firewall allowing connection?
- [ ] Check PostgreSQL logs for errors

**Query troubleshooting:**
- [ ] Connected to correct database? (`SELECT current_database();`)
- [ ] Table exists in current database? (`\dt` or `SELECT * FROM pg_tables WHERE tablename='customers';`)
- [ ] Correct schema? (default is public)
- [ ] Correct table name spelling?
- [ ] Sufficient permissions? (`\dt` shows owner)
- [ ] Table has data? (`SELECT COUNT(*) FROM tablename;`)

**pgAdmin specific:**
- [ ] Server connection green (connected)?
- [ ] Query Tool title bar shows correct database?
- [ ] SQL has no syntax errors (no red underlines)?
- [ ] Query Tool window has focus before pressing F5?
- [ ] Not too many Query Tool tabs open (close unused)?

---

## 14. Answers

### Answer to Question 1: Database connection architecture

PostgreSQL's "one connection to one database" model provides strong security isolation and clear transactional boundaries - each database has its own system catalogs, permissions, and transaction logs, preventing accidental cross-database data leakage. This architecture simplifies ACID compliance, enables database-level backup/restore, and allows efficient resource management since locks and memory are scoped to a single database. The main limitation is inability to JOIN tables across databases directly, requiring workarounds like foreign data wrappers (postgres_fdw), dblink extension, or architectural redesign using schemas within one database instead of separate databases. In practice, this design encourages proper database architecture where database boundaries represent true application or tenant isolation, not just organizational convenience.

### Answer to Question 2: Connection parameter decisions

For a script accessing three databases, establish three separate connections sequentially, explicitly specifying all parameters (-h, -p, -U, -d) even when they're identical, to prevent errors from changed defaults or wrong environments. The trade-off between keeping connections open simultaneously versus connect-disconnect-reconnect depends on script duration and server load: simultaneous connections consume more connection slots but save ~30-150ms total connection overhead, while sequential connections free resources but pay authentication cost each time. For a nightly batch job, sequential connections are typically better unless the script is extremely time-sensitive, since connection overhead (10-50ms each) is negligible compared to data processing time. Always implement error handling to decide whether database-specific failures should abort the entire script or allow other databases to process independently.

### Answer to Question 3: psql versus pgAdmin decision-making

For investigating a 3 AM batch job failure, psql is superior because you can SSH remotely, quickly examine logs, query system catalogs, and maintain a complete command history for audit trails without needing a GUI connection. For training a new developer on query optimization, pgAdmin is far better due to visual EXPLAIN plan displays, schema browser, syntax highlighting, and intuitive interface that helps beginners understand database structure without memorizing commands. For automating weekly maintenance, psql is the only practical choice since automation requires scripting, cron jobs, and unattended execution - pgAdmin is a GUI tool not designed for this. For creating complex one-time queries with multiple joins, pgAdmin is often more comfortable for iterative development (syntax highlighting, partial execution, multiple result tabs), though experienced DBAs may prefer psql with \e for editor integration - many develop queries in pgAdmin then migrate to psql scripts for production use.

### Answer to Question 4: Meta-command efficiency

psql meta-commands like \dt, \d, and \l provide enormous productivity benefits because they're concise, consistent, and format output for human readability - the equivalent SQL queries against system catalogs are complex and hard to memorize (e.g., \dt requires querying pg_tables with schema filtering). The consistent pattern (\d commands for describing objects, \l for databases) creates a compact mental model versus dozens of different SQL queries, making new DBAs immediately productive. However, use direct SQL queries against system catalogs for automation scripts that need stable, parseable output (meta-command formatting can change between versions) or for specialized queries beyond common use cases like finding unused indexes or complex foreign key relationships. Understanding both levels - meta-commands for interactive work and system catalogs for programmatic access - gives you maximum flexibility as a DBA.

### Answer to Question 5: Script execution analysis

In psql's default autocommit mode, each statement in your script executes as its own transaction, so if INSERT #30 fails, statements 1-29 have already committed permanently while statements 31-60 continue executing - resulting in partial, inconsistent data. To ensure atomicity (all succeed or none), wrap the entire script in BEGIN/COMMIT or use psql's --single-transaction flag, which treats the whole script as one transaction that rolls back completely if any statement fails. Explicit transaction control is essential for data loading and schema migration scripts where you need all-or-nothing semantics, though very large loads may require batch commits (every 1000 rows) to balance atomicity against lock duration and memory usage. The key distinction: autocommit mode provides statement-level atomicity (fine for interactive work), while explicit transactions provide script-level atomicity (required for production data loads).

### Answer to Question 6: Object definition exploration

Viewing SQL definitions via pgAdmin's Properties → SQL tab shows exactly how PostgreSQL created objects, including all defaults, constraints, and system-generated elements - this is invaluable for learning syntax through real-world examples rather than simplified documentation. Three critical scenarios: (1) discovering optimization techniques you didn't know existed by examining high-performing tables (partial indexes, expression indexes, includes), (2) troubleshooting application errors by revealing exact constraints, triggers, and permissions causing failures, and (3) reproducing database structures precisely in dev/test environments or during disaster recovery. The generated SQL is more verbose than your original creation statement because PostgreSQL makes everything explicit (fully qualified names, all defaults, system-generated constraint names), which actually improves reliability when recreating objects elsewhere since you're not relying on implicit defaults that might differ.

### Answer to Question 7: Performance implications

The single-connection join query is dramatically superior because connection establishment (authentication, memory allocation, catalog cache) costs 10-50ms per connection, you pay network overhead three times, each query is planned independently without cross-query optimization, and you must manually combine results in application code - total overhead easily 100-500ms. The join query pays connection cost once, executes entirely in PostgreSQL's optimized C code using efficient join algorithms and indexes, transfers only the final result across the network, and typically completes in 10-50ms total. The only scenario where multiple connections might help is true parallelism on multi-core servers for independent queries on massive tables, but PostgreSQL's parallel query features usually handle this better. The fundamental principle: do work where it's most efficient - database servers are optimized for set-based operations like joins, while connection management and data serialization are pure overhead to minimize.

### Answer to Question 8: Error interpretation

First ask: "Which database are you connected to?" - run SELECT current_database() or check the psql prompt, since they likely created the table in lab01_practice but are querying while connected to postgres. If database context is correct, check for uncommitted transactions (they ran BEGIN; CREATE TABLE but never COMMIT), verify the exact query syntax for case sensitivity issues or schema qualification problems, and run \dt to see what tables actually exist in the current schema. Less common causes include permissions issues (table exists but they lack access) or the table being in a non-public schema not in their search_path. This error almost always stems from database context confusion - being connected to the wrong database, wrong schema, or having uncommitted work - so verify context assumptions (where am I? what state am I in?) before investigating complex causes.