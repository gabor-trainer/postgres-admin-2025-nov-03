# Lab 3: Roles and permissions in PostgreSQL

**Estimated time:** 60 minutes  
**Difficulty level:** Beginner  
**PostgreSQL version:** 18.x  
**Platform requirements:** Any

---

## 1. Learning objectives

By the end of this lab, you will be able to:

- Create group roles (non-login) to organize permissions
- Create user roles (login) and assign them to groups
- Grant table-level SELECT and DML permissions (INSERT, UPDATE, DELETE)
- Test permissions using different user connections
- Understand role inheritance and membership in PostgreSQL

---

## 2. Prerequisites

**Knowledge prerequisites:**
- Basic understanding of PostgreSQL roles
- Familiarity with SQL SELECT, INSERT, UPDATE, DELETE statements
- Understanding of database ownership

**Previous labs:**
- Lab 2 (Installing Northwind database) must be completed

**Standard environment checklist:**
- [ ] PostgreSQL 18 installed on Windows (port 5432)
- [ ] Northwind database exists with all tables populated
- [ ] Can connect as postgres user
- [ ] pgAdmin and psql available

---

## 3. Lab-specific setup

### Verify Northwind database exists

```sql
-- Connect as postgres
psql -h localhost -p 5432 -U postgres -d postgres
```

```sql
-- Verify northwind exists
SELECT datname FROM pg_database WHERE datname = 'northwind';

-- Verify tables exist
\c northwind
\dt
```

Expected output: You should see 13+ tables including customers, orders, employees, products, etc.

### Pre-lab cleanup

If you've attempted this lab before, clean up first:

```sql
-- Connect to postgres database
\c postgres

-- Terminate any active connections
SELECT pg_terminate_backend(pid) 
FROM pg_stat_activity 
WHERE datname = 'northwind' AND pid <> pg_backend_pid();

-- Drop test users if they exist
DROP ROLE IF EXISTS john;
DROP ROLE IF EXISTS bill;
DROP ROLE IF EXISTS accounting;
DROP ROLE IF EXISTS hr;
```

---

## 4. Concept overview

PostgreSQL implements a sophisticated role-based access control (RBAC) system where roles can represent both users (with LOGIN privilege) and groups (without LOGIN privilege). This allows you to organize permissions hierarchically: grant permissions to group roles, then add users as members of those groups.

This lab demonstrates a common business scenario: the accounting department needs to manage orders and customer data, while HR manages employee information. Rather than granting permissions to each user individually, we create department roles and assign users to them. This approach scales well and simplifies permission management.

---

## 5. Exercises

### Exercise 1.1: Create group roles and user roles

**Platform:** Windows  
**Objective:** Learn to create non-login group roles and login user roles  
**Scenario:** Your company needs separate accounting and HR departments with different database access patterns

**Tasks:**

1. **Connect to the northwind database as postgres:**

   ```
   psql -h localhost -p 5432 -U postgres -d northwind
   ```

2. **Create the accounting group role (no login):**

   ```sql
   CREATE ROLE accounting 
       NOINHERIT 
       NOLOGIN;
   ```

   **Explanation:**
   - `CREATE ROLE`: Creates a new database role
   - `accounting`: The role name
   - `NOINHERIT`: This role will NOT automatically inherit privileges from roles it's a member of (we'll override this for users)
   - `NOLOGIN`: This is a group role; users cannot connect as "accounting" directly

3. **Create the hr group role (no login):**

   ```sql
   CREATE ROLE hr 
       NOINHERIT 
       NOLOGIN;
   ```

4. **Create the john user role (with login):**

   ```sql
   CREATE ROLE john 
       LOGIN 
       INHERIT 
       PASSWORD 'john123';
   ```

   **Explanation:**
   - `LOGIN`: This role can connect to the database
   - `INHERIT`: John will automatically inherit privileges from any groups he's a member of
   - `PASSWORD 'john123'`: Sets the login password (use strong passwords in production)

5. **Create the bill user role (with login):**

   ```sql
   CREATE ROLE bill 
       LOGIN 
       INHERIT 
       PASSWORD 'bill123';
   ```

6. **Verify roles were created:**

   ```sql
   SELECT rolname, rolcanlogin, rolinherit 
   FROM pg_roles 
   WHERE rolname IN ('accounting', 'hr', 'john', 'bill')
   ORDER BY rolname;
   ```

   Expected output:
   ```
     rolname   | rolcanlogin | rolinherit 
   ------------+-------------+------------
    accounting | f           | f
    bill       | t           | t
    hr         | f           | f
    john       | t           | t
   (4 rows)
   ```

**Verification steps:**

1. Check that accounting and hr cannot login (rolcanlogin = false)
2. Check that john and bill can login (rolcanlogin = true)
3. Check that john and bill have inherit enabled (rolinherit = true)

---

### Exercise 1.2: Assign users to groups

**Platform:** Windows  
**Objective:** Learn to add user roles as members of group roles  
**Scenario:** Assign john to the accounting department and bill to the HR department

**Tasks:**

1. **Add john as a member of the accounting role:**

   ```sql
   GRANT accounting TO john;
   ```

   **Explanation:**
   - `GRANT accounting`: Makes john a member of the accounting group
   - `TO john`: The user receiving membership
   - Because john has INHERIT enabled, he automatically gets all privileges granted to accounting

2. **Add bill as a member of the hr role:**

   ```sql
   GRANT hr TO bill;
   ```

3. **Verify role memberships:**

   ```sql
   SELECT 
       r.rolname AS role,
       m.rolname AS member
   FROM pg_roles r
   JOIN pg_auth_members am ON r.oid = am.roleid
   JOIN pg_roles m ON am.member = m.oid
   WHERE r.rolname IN ('accounting', 'hr')
   ORDER BY r.rolname, m.rolname;
   ```

   Expected output:
   ```
       role    | member 
   ------------+--------
    accounting | john
    hr         | bill
   (2 rows)
   ```

4. **Check what roles john currently has:**

   ```sql
   -- Connect as john
   \c northwind john
   -- Password: john123
   
   SELECT current_user, session_user;
   ```

   Expected output:
   ```
    current_user | session_user 
   --------------+--------------
    john         | john
   (1 row)
   ```

5. **See all effective roles for john:**

   ```sql
   SELECT * FROM pg_roles WHERE oid IN (SELECT * FROM pg_roles_in_use());
   ```

   Or simpler:
   ```sql
   \du john
   ```

**Verification steps:**

- john is a member of accounting
- bill is a member of hr
- Both users can connect to the database

---

### Exercise 1.3: Grant permissions to the accounting role

**Platform:** Windows  
**Objective:** Grant SELECT on all tables and write permissions on business-critical tables for accounting  
**Scenario:** Accounting needs to read all data but only modify orders, customers, and related tables (not employees)

**Tasks:**

1. **Reconnect as postgres:**

   ```sql
   \c northwind postgres
   ```

2. **Grant SELECT on all tables to accounting:**

   ```sql
   GRANT SELECT ON ALL TABLES IN SCHEMA public TO accounting;
   ```

   **Explanation:**
   - `GRANT SELECT`: Gives read permission
   - `ON ALL TABLES IN SCHEMA public`: Applies to every table in the public schema
   - `TO accounting`: The accounting group receives these privileges
   - This allows accounting to query any table in northwind

3. **Grant write permissions on orders table:**

   ```sql
   GRANT INSERT, UPDATE, DELETE ON orders TO accounting;
   ```

   **Explanation:**
   - `INSERT, UPDATE, DELETE`: The three write operations (collectively called DML)
   - `ON orders`: Only applies to the orders table
   - Accounting can now modify order records

4. **Grant write permissions on customers table:**

   ```sql
   GRANT INSERT, UPDATE, DELETE ON customers TO accounting;
   ```

5. **Grant write permissions on order_details (related to orders):**

   ```sql
   GRANT INSERT, UPDATE, DELETE ON order_details TO accounting;
   ```

   **Explanation:**
   - order_details has a foreign key to orders
   - Accounting needs to modify line items when managing orders

6. **Grant write permissions on products (purchased in orders):**

   ```sql
   GRANT INSERT, UPDATE, DELETE ON products TO accounting;
   ```

7. **Grant write permissions on categories (related to products):**

   ```sql
   GRANT INSERT, UPDATE, DELETE ON categories TO accounting;
   ```

8. **Grant write permissions on suppliers (related to products):**

   ```sql
   GRANT INSERT, UPDATE, DELETE ON suppliers TO accounting;
   ```

9. **Grant write permissions on shippers (related to orders):**

   ```sql
   GRANT INSERT, UPDATE, DELETE ON shippers TO accounting;
   ```

10. **Verify accounting permissions:**

    ```sql
    SELECT 
        table_name,
        privilege_type
    FROM information_schema.table_privileges
    WHERE grantee = 'accounting'
    ORDER BY table_name, privilege_type;
    ```

    Expected output (partial):
    ```
       table_name    | privilege_type 
    -----------------+----------------
     categories      | DELETE
     categories      | INSERT
     categories      | SELECT
     categories      | UPDATE
     customers       | DELETE
     customers       | INSERT
     customers       | SELECT
     customers       | UPDATE
     employees       | SELECT
     order_details   | DELETE
     order_details   | INSERT
     order_details   | SELECT
     order_details   | UPDATE
     orders          | DELETE
     orders          | INSERT
     orders          | SELECT
     orders          | UPDATE
     products        | DELETE
     ...
    ```

    **Notice:** employees has only SELECT, not INSERT/UPDATE/DELETE

**Verification steps:**

- accounting has SELECT on ALL tables
- accounting has INSERT, UPDATE, DELETE on: orders, customers, order_details, products, categories, suppliers, shippers
- accounting does NOT have write access to employees

---

### Exercise 1.4: Grant permissions to the hr role

**Platform:** Windows  
**Objective:** Grant SELECT on all tables and write permissions only on employees table for hr  
**Scenario:** HR needs to read all data for reporting but only modify employee records

**Tasks:**

1. **Ensure you're connected as postgres:**

   ```sql
   SELECT current_user;
   ```

   If not postgres, reconnect:
   ```sql
   \c northwind postgres
   ```

2. **Grant SELECT on all tables to hr:**

   ```sql
   GRANT SELECT ON ALL TABLES IN SCHEMA public TO hr;
   ```

3. **Grant write permissions on employees table:**

   ```sql
   GRANT INSERT, UPDATE, DELETE ON employees TO hr;
   ```

4. **Grant write permissions on employee_territories (related to employees):**

   ```sql
   GRANT INSERT, UPDATE, DELETE ON employee_territories TO hr;
   ```

   **Explanation:**
   - employee_territories links employees to sales territories
   - HR needs to manage employee assignments

5. **Grant write permissions on territories (related to employee_territories):**

   ```sql
   GRANT INSERT, UPDATE, DELETE ON territories TO hr;
   ```

6. **Grant write permissions on region (related to territories):**

   ```sql
   GRANT INSERT, UPDATE, DELETE ON region TO hr;
   ```

7. **Verify hr permissions:**

   ```sql
   SELECT 
       table_name,
       privilege_type
   FROM information_schema.table_privileges
   WHERE grantee = 'hr'
    ORDER BY table_name, privilege_type;
   ```

    Expected output (partial):
    ```
          table_name      | privilege_type 
    ----------------------+----------------
     categories           | SELECT
     customers            | SELECT
     employee_territories | DELETE
     employee_territories | INSERT
     employee_territories | SELECT
     employee_territories | UPDATE
     employees            | DELETE
     employees            | INSERT
     employees            | SELECT
     employees            | UPDATE
     orders               | SELECT
     products             | SELECT
     region               | DELETE
     region               | INSERT
     region               | SELECT
     region               | UPDATE
     territories          | DELETE
     territories          | INSERT
     territories          | SELECT
     territories          | UPDATE
     ...
    ```

    **Notice:** orders and customers have only SELECT, not write permissions

**Verification steps:**

- hr has SELECT on ALL tables
- hr has INSERT, UPDATE, DELETE on: employees, employee_territories, territories, region
- hr does NOT have write access to orders, customers, or products

---

### Exercise 1.5: Test accounting permissions as john

**Platform:** Windows  
**Objective:** Verify that john can perform authorized operations and is blocked from unauthorized ones  
**Scenario:** Test that john (accounting) can modify orders but not employees

**Tasks:**

1. **Connect as john:**

   ```sql
   \c northwind john
   -- Password: john123
   ```

2. **Test SELECT permission (should succeed):**

   ```sql
   SELECT customer_id, company_name, city 
   FROM customers 
   LIMIT 3;
   ```

   Expected output:
   ```
    customer_id |       company_name        |    city     
   -------------+---------------------------+-------------
    ALFKI       | Alfreds Futterkiste       | Berlin
    ANATR       | Ana Trujillo Emparedados y helados | México D.F.
    ANTON       | Antonio Moreno Taquería   | México D.F.
   (3 rows)
   ```

3. **Test SELECT on employees (should succeed):**

   ```sql
   SELECT employee_id, first_name, last_name 
   FROM employees 
   LIMIT 3;
   ```

   Expected output:
   ```
    employee_id | first_name | last_name 
   -------------+------------+-----------
              1 | Nancy      | Davolio
              2 | Andrew     | Fuller
              3 | Janet      | Leverling
   (3 rows)
   ```

4. **Test INSERT on customers (should succeed):**

   ```sql
   INSERT INTO customers (customer_id, company_name, contact_name, city, country)
   VALUES ('TESTT', 'Test Company', 'John Doe', 'New York', 'USA');
   ```

   Expected output:
   ```
   INSERT 0 1
   ```

5. **Verify the insert worked:**

   ```sql
   SELECT * FROM customers WHERE customer_id = 'TESTT';
   ```

6. **Test UPDATE on orders (should succeed):**

   ```sql
   -- First, see an existing order
   SELECT order_id, customer_id, freight 
   FROM orders 
   WHERE order_id = 10248;
   
   -- Update the freight cost
   UPDATE orders 
   SET freight = 50.00 
   WHERE order_id = 10248;
   
   -- Verify the update
   SELECT order_id, customer_id, freight 
   FROM orders 
   WHERE order_id = 10248;
   ```

   Expected output: freight should now be 50.00

7. **Test INSERT on employees (should FAIL):**

   ```sql
   INSERT INTO employees (employee_id, last_name, first_name)
   VALUES (99, 'Test', 'Employee');
   ```

   Expected output:
   ```
   ERROR:  permission denied for table employees
   ```

   **Explanation:** john (accounting) has SELECT on employees but not INSERT

8. **Test UPDATE on employees (should FAIL):**

   ```sql
   UPDATE employees 
   SET first_name = 'Johnny' 
   WHERE employee_id = 1;
   ```

   Expected output:
   ```
   ERROR:  permission denied for table employees
   ```

9. **Test DELETE on employees (should FAIL):**

   ```sql
   DELETE FROM employees WHERE employee_id = 99;
   ```

   Expected output:
   ```
   ERROR:  permission denied for table employees
   ```

**Verification steps:**

- john can SELECT from all tables ✓
- john can INSERT, UPDATE, DELETE on customers and orders ✓
- john CANNOT INSERT, UPDATE, DELETE on employees ✓

---

### Exercise 1.6: Test hr permissions as bill

**Platform:** Windows  
**Objective:** Verify that bill can modify employees but not orders  
**Scenario:** Test that bill (hr) can update employee records but not customer orders

**Tasks:**

1. **Connect as bill:**

   ```sql
   \c northwind bill
   -- Password: bill123
   ```

2. **Test SELECT on orders (should succeed):**

   ```sql
   SELECT order_id, customer_id, order_date 
   FROM orders 
   LIMIT 3;
   ```

   Expected output:
   ```
    order_id | customer_id | order_date 
   ----------+-------------+------------
       10248 | VINET       | 1996-07-04
       10249 | TOMSP       | 1996-07-05
       10250 | HANAR       | 1996-07-08
   (3 rows)
   ```

3. **Test SELECT on employees (should succeed):**

   ```sql
   SELECT employee_id, first_name, last_name, title 
   FROM employees 
   LIMIT 3;
   ```

4. **Test UPDATE on employees (should succeed):**

   ```sql
   -- Update an employee title
   UPDATE employees 
   SET title = 'Senior Sales Representative' 
   WHERE employee_id = 1;
   
   -- Verify the update
   SELECT employee_id, first_name, last_name, title 
   FROM employees 
   WHERE employee_id = 1;
   ```

   Expected output: title should be updated

5. **Test INSERT on employees (should succeed):**

   ```sql
   INSERT INTO employees (employee_id, last_name, first_name, title)
   VALUES (99, 'Smith', 'Sarah', 'HR Manager');
   ```

   Expected output:
   ```
   INSERT 0 1
   ```

6. **Verify the insert:**

   ```sql
   SELECT employee_id, first_name, last_name, title 
   FROM employees 
   WHERE employee_id = 99;
   ```

7. **Test DELETE on the new employee (should succeed):**

   ```sql
   DELETE FROM employees WHERE employee_id = 99;
   ```

   Expected output:
   ```
   DELETE 1
   ```

8. **Test INSERT on customers (should FAIL):**

   ```sql
   INSERT INTO customers (customer_id, company_name, city, country)
   VALUES ('BILL1', 'Bills Company', 'Chicago', 'USA');
   ```

   Expected output:
   ```
   ERROR:  permission denied for table customers
   ```

9. **Test UPDATE on orders (should FAIL):**

   ```sql
   UPDATE orders 
   SET freight = 99.99 
   WHERE order_id = 10248;
   ```

   Expected output:
   ```
   ERROR:  permission denied for table orders
   ```

10. **Test DELETE on customers (should FAIL):**

    ```sql
    DELETE FROM customers WHERE customer_id = 'TESTT';
    ```

    Expected output:
    ```
    ERROR:  permission denied for table customers
    ```

**Verification steps:**

- bill can SELECT from all tables ✓
- bill can INSERT, UPDATE, DELETE on employees ✓
- bill CANNOT INSERT, UPDATE, DELETE on customers or orders ✓

---

## 6. Validation checklist

After completing all exercises, verify:

- [ ] Four roles exist: accounting (no login), hr (no login), john (login), bill (login)
- [ ] john is a member of accounting
- [ ] bill is a member of hr
- [ ] accounting has SELECT on all tables
- [ ] accounting has write access to orders, customers, and related tables
- [ ] accounting does NOT have write access to employees
- [ ] hr has SELECT on all tables
- [ ] hr has write access to employees and related tables
- [ ] hr does NOT have write access to orders or customers
- [ ] john can modify orders but not employees
- [ ] bill can modify employees but not orders

**Final validation script (run as postgres):**

```sql
\c northwind postgres

-- Check role creation
SELECT rolname, rolcanlogin 
FROM pg_roles 
WHERE rolname IN ('accounting', 'hr', 'john', 'bill')
ORDER BY rolname;

-- Check memberships
SELECT r.rolname AS role, m.rolname AS member
FROM pg_roles r
JOIN pg_auth_members am ON r.oid = am.roleid
JOIN pg_roles m ON am.member = m.oid
WHERE r.rolname IN ('accounting', 'hr');

-- Check accounting permissions on employees (should be SELECT only)
SELECT privilege_type
FROM information_schema.table_privileges
WHERE grantee = 'accounting' AND table_name = 'employees';

-- Check hr permissions on orders (should be SELECT only)
SELECT privilege_type
FROM information_schema.table_privileges
WHERE grantee = 'hr' AND table_name = 'orders';
```

---

## 7. Troubleshooting guide

### Windows

**Problem:** "role 'john' already exists"
- **Solution:** Drop existing roles first using the cleanup script in section 3
- **Or:** Use `DROP ROLE IF EXISTS john;` before creating

**Problem:** "permission denied for table X" when you expect it to work
- **Solution:** Check if you're connected as the right user: `SELECT current_user;`
- **Verify group membership:** `\du username`
- **Check if permissions were granted to the group:** Query information_schema.table_privileges

**Problem:** "cannot drop role because it owns objects"
- **Solution:** Reassign ownership first: `REASSIGN OWNED BY john TO postgres;`
- **Then drop:** `DROP OWNED BY john; DROP ROLE john;`

**Problem:** User can access tables they shouldn't
- **Solution:** Check for PUBLIC grants: `SELECT * FROM information_schema.table_privileges WHERE grantee = 'PUBLIC';`
- **Revoke if needed:** `REVOKE ALL ON ALL TABLES IN SCHEMA public FROM PUBLIC;`

**Problem:** Permissions not working even though granted
- **Solution:** Verify the user has INHERIT enabled: `SELECT rolname, rolinherit FROM pg_roles WHERE rolname = 'john';`
- **If false, alter:** `ALTER ROLE john INHERIT;`

**Problem:** Can't connect as john or bill
- **Solution:** Check if LOGIN privilege exists: `SELECT rolname, rolcanlogin FROM pg_roles WHERE rolname = 'john';`
- **Verify password:** Try resetting: `ALTER ROLE john PASSWORD 'john123';`

---

## 8. Questions

1. Why create group roles (accounting, hr) instead of granting permissions directly to user roles (john, bill)? What happens when you need to add a third accountant?

2. What is the difference between NOINHERIT on group roles and INHERIT on user roles? Why did we configure them differently?

3. If john needs temporary access to modify employee records for a special project, how would you grant this without changing the accounting group's permissions?

---

## 9. Clean-up

To remove all test data and roles:

```sql
-- Connect as postgres
\c postgres postgres

-- Terminate connections
SELECT pg_terminate_backend(pid) 
FROM pg_stat_activity 
WHERE datname = 'northwind' 
  AND usename IN ('john', 'bill')
  AND pid <> pg_backend_pid();

-- Connect to northwind
\c northwind postgres

-- Remove test customer record
DELETE FROM customers WHERE customer_id = 'TESTT';

-- Reset freight on order 10248
UPDATE orders SET freight = 32.38 WHERE order_id = 10248;

-- Reset employee title
UPDATE employees 
SET title = 'Sales Representative' 
WHERE employee_id = 1;

-- Connect back to postgres
\c postgres postgres

-- Revoke permissions and drop roles
REVOKE ALL PRIVILEGES ON ALL TABLES IN SCHEMA public FROM accounting;
REVOKE ALL PRIVILEGES ON ALL TABLES IN SCHEMA public FROM hr;

DROP ROLE IF EXISTS john;
DROP ROLE IF EXISTS bill;
DROP ROLE IF EXISTS accounting;
DROP ROLE IF EXISTS hr;
```

Verify cleanup:
```sql
SELECT rolname FROM pg_roles WHERE rolname IN ('accounting', 'hr', 'john', 'bill');
```

Should return zero rows.

---

## 10. Key takeaways

- Group roles (NOLOGIN) organize permissions; user roles (LOGIN) inherit from groups
- INHERIT on user roles allows automatic privilege inheritance from group membership
- Grant permissions to groups, not individual users, for easier management
- SELECT can be granted broadly; write permissions should be restricted to specific tables
- Always test permissions with actual user connections, not just as postgres
- Use information_schema.table_privileges to audit permissions
- Role-based access control scales better than user-specific permissions

---

## 11. Additional resources

**Official PostgreSQL 18 documentation:**
- [Role membership](https://www.postgresql.org/docs/18/role-membership.html)
- [GRANT command](https://www.postgresql.org/docs/18/sql-grant.html)
- [ALTER ROLE](https://www.postgresql.org/docs/18/sql-alterrole.html)
- [Privileges](https://www.postgresql.org/docs/18/ddl-priv.html)

---

## 12. Appendices

### Appendix A: Quick reference commands

**Create roles:**
```sql
-- Group role (no login)
CREATE ROLE groupname NOLOGIN;

-- User role (login)
CREATE ROLE username LOGIN PASSWORD 'password';
```

**Assign membership:**
```sql
GRANT groupname TO username;
```

**Grant permissions:**
```sql
-- Read access
GRANT SELECT ON ALL TABLES IN SCHEMA public TO rolename;

-- Write access to specific table
GRANT INSERT, UPDATE, DELETE ON tablename TO rolename;
```

**Check permissions:**
```sql
-- View role properties
\du rolename

-- View table privileges
SELECT * FROM information_schema.table_privileges WHERE grantee = 'rolename';

-- Check current user
SELECT current_user, session_user;
```

**Revoke permissions:**
```sql
REVOKE ALL ON tablename FROM rolename;
REVOKE ALL ON ALL TABLES IN SCHEMA public FROM rolename;
```

### Appendix B: Permission matrix

| Table                | accounting                     | hr                             |
| -------------------- | ------------------------------ | ------------------------------ |
| customers            | SELECT, INSERT, UPDATE, DELETE | SELECT only                    |
| orders               | SELECT, INSERT, UPDATE, DELETE | SELECT only                    |
| order_details        | SELECT, INSERT, UPDATE, DELETE | SELECT only                    |
| products             | SELECT, INSERT, UPDATE, DELETE | SELECT only                    |
| categories           | SELECT, INSERT, UPDATE, DELETE | SELECT only                    |
| suppliers            | SELECT, INSERT, UPDATE, DELETE | SELECT only                    |
| shippers             | SELECT, INSERT, UPDATE, DELETE | SELECT only                    |
| employees            | SELECT only                    | SELECT, INSERT, UPDATE, DELETE |
| employee_territories | SELECT only                    | SELECT, INSERT, UPDATE, DELETE |
| territories          | SELECT only                    | SELECT, INSERT, UPDATE, DELETE |
| region               | SELECT only                    | SELECT, INSERT, UPDATE, DELETE |

### Appendix C: Common permission patterns

**Read-only user:**
```sql
CREATE ROLE readonly LOGIN PASSWORD 'pass';
GRANT CONNECT ON DATABASE northwind TO readonly;
GRANT USAGE ON SCHEMA public TO readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;
```

**Full access (non-superuser):**
```sql
CREATE ROLE poweruser LOGIN PASSWORD 'pass';
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO poweruser;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO poweruser;
```

**Application role:**
```sql
CREATE ROLE app_user LOGIN PASSWORD 'secure_pass';
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_user;
GRANT INSERT, UPDATE, DELETE ON orders, customers TO app_user;
```

---

## 13. Answers

### Answer to Question 1

Creating group roles instead of granting permissions directly to users provides scalability and maintainability. When you grant permissions to the accounting group once, any new member automatically receives those permissions. If you need to add a third accountant (e.g., create user 'alice'), you simply execute `GRANT accounting TO alice`, and alice immediately has all accounting permissions without additional GRANT statements. This approach follows the DRY (Don't Repeat Yourself) principle and reduces administrative overhead. Additionally, if accounting needs new permissions later (e.g., write access to a new invoices table), you grant it once to the accounting group, and all members (john, alice, and future accountants) receive it automatically.

### Answer to Question 2

NOINHERIT on group roles (accounting, hr) means these roles don't automatically inherit from roles they might be members of, which isn't relevant here since they aren't members of other roles. INHERIT on user roles (john, bill) means these users automatically receive all privileges granted to their group roles without needing to explicitly SET ROLE. We configured them differently because groups organize permissions (they don't need to inherit), while users need to inherit to actually use those permissions. If john had NOINHERIT, he would need to execute `SET ROLE accounting` before accessing accounting's privileges, adding unnecessary complexity to every database session.

### Answer to Question 3

To grant john temporary employee access without modifying the accounting group, you would grant permissions directly to john: `GRANT INSERT, UPDATE, DELETE ON employees TO john;`. This supplements his inherited permissions from accounting. User-specific grants override or add to group permissions. When the project ends, revoke the specific grant: `REVOKE INSERT, UPDATE, DELETE ON employees FROM john;`. This approach keeps the accounting group's permissions unchanged while allowing temporary individual exceptions. Alternatively, you could create a temporary role (e.g., hr_project), grant it the needed permissions, add john to that role during the project, then remove the membership afterward.