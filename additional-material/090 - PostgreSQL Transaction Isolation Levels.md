# PostgreSQL Transaction Isolation Levels

## Setup

```sql
-- Run in pgAdmin to verify starting state
SELECT product_id, product_name, unit_price, units_in_stock 
FROM products 
WHERE product_id IN (1, 2, 3)
ORDER BY product_id;
```

**Expected output:**
```
product_id | product_name | unit_price | units_in_stock
-----------+--------------+------------+---------------
1          | Chai         | 18         | 39
2          | Chang        | 19         | 17
3          | Aniseed Syrup| 10         | 13
```

---

## Demo 1: Read committed - modifications not visible until commit

**pgAdmin (Connection 1):**
```sql
BEGIN;
UPDATE products SET unit_price = 25.00 WHERE product_id = 1;
-- Do NOT commit yet
SELECT unit_price FROM products WHERE product_id = 1;
```

**Expected output in pgAdmin:**
```
unit_price
----------
25
```

**psql (Connection 2):**
```sql
-- Check default isolation level
SHOW transaction_isolation;

-- Read the same row
SELECT unit_price FROM products WHERE product_id = 1;
```

**Expected output in psql:**
```
transaction_isolation
---------------------
read committed

unit_price
----------
18
```

**Why**: Connection 2 sees the old value (18) because Connection 1 hasn't committed. Read Committed only sees committed data.

**pgAdmin (Connection 1):**
```sql
COMMIT;
```

**psql (Connection 2):**
```sql
-- Read again after commit
SELECT unit_price FROM products WHERE product_id = 1;
```

**Expected output in psql:**
```
unit_price
----------
25
```

**Why**: After commit, Connection 2 now sees the new value.

---

## Demo 2: Read committed - blocking on concurrent modifications

**pgAdmin (Connection 1):**
```sql
BEGIN;
UPDATE products SET units_in_stock = 100 WHERE product_id = 2;
-- Do NOT commit yet
```

**psql (Connection 2):**
```sql
BEGIN;
-- Try to update the same row
UPDATE products SET units_in_stock = 200 WHERE product_id = 2;
-- This will BLOCK (hang) waiting for Connection 1
```

**Expected behavior in psql:**
```
-- Command hangs, no output yet
-- psql appears frozen
```

**Why**: Connection 2 must wait for Connection 1 to commit or rollback. PostgreSQL prevents lost updates by locking modified rows.

**pgAdmin (Connection 1):**
```sql
COMMIT;
```

**Expected output in psql (immediately after Connection 1 commits):**
```
UPDATE 1
```

**psql (Connection 2):**
```sql
-- Verify what happened
SELECT units_in_stock FROM products WHERE product_id = 2;
COMMIT;
```

**Expected output in psql:**
```
units_in_stock
--------------
200
```

**Why**: Connection 2's UPDATE executed after Connection 1 committed. Connection 2 saw the committed value (100) and updated it to 200.

---

## Demo 3: Read committed vs Repeatable Read - the key difference

**pgAdmin (Connection 1):**
```sql
BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- Read 1: Initial read
SELECT unit_price FROM products WHERE product_id = 3;
```

**Expected output:**
```
unit_price
----------
10
```

**psql (Connection 2):**
```sql
-- Another user updates and commits
UPDATE products SET unit_price = 15.00 WHERE product_id = 3;
```

**Expected output:**
```
UPDATE 1
```

**pgAdmin (Connection 1):**
```sql
-- Read 2: Read again in same transaction
SELECT unit_price FROM products WHERE product_id = 3;
COMMIT;
```

**Expected output:**
```
unit_price
----------
15
```

**Why**: Read Committed sees the new committed value (15) even within the same transaction. This is a **non-repeatable read**.

---

**Now with Repeatable Read:**

**pgAdmin (Connection 1):**
```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Read 1: Initial read
SELECT unit_price FROM products WHERE product_id = 3;
```

**Expected output:**
```
unit_price
----------
15
```

**psql (Connection 2):**
```sql
-- Another user updates and commits
UPDATE products SET unit_price = 20.00 WHERE product_id = 3;
```

**Expected output:**
```
UPDATE 1
```

**pgAdmin (Connection 1):**
```sql
-- Read 2: Read again in same transaction
SELECT unit_price FROM products WHERE product_id = 3;
COMMIT;
```

**Expected output:**
```
unit_price
----------
15
```

**Why**: Repeatable Read provides a consistent snapshot. Connection 1 sees the same value (15) throughout its transaction, ignoring concurrent commits.

---

## Demo 4: Repeatable Read - phantom read prevention

**pgAdmin (Connection 1):**
```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Count products in category 1
SELECT COUNT(*) FROM products WHERE category_id = 1;
```

**Expected output:**
```
count
-----
12
```

**psql (Connection 2):**
```sql
-- Insert a new product in category 1
INSERT INTO products (product_id, product_name, category_id, unit_price, units_in_stock, units_on_order, reorder_level, discontinued)
VALUES (78, 'New Beverage', 1, 10, 50, 0, 10, 0);
```

**Expected output:**
```
INSERT 0 1
```

**pgAdmin (Connection 1):**
```sql
-- Count again in same transaction
SELECT COUNT(*) FROM products WHERE category_id = 1;
COMMIT;
```

**Expected output:**
```
count
-----
12
```

**Why**: Repeatable Read in PostgreSQL prevents phantom reads. The same query returns the same result set throughout the transaction.

---

**Compare with Read Committed:**

**pgAdmin (Connection 1):**
```sql
BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- Count products in category 1
SELECT COUNT(*) FROM products WHERE category_id = 1;
```

**Expected output:**
```
count
-----
13
```

**psql (Connection 2):**
```sql
-- Insert another product
INSERT INTO products (product_id, product_name, category_id, unit_price, units_in_stock, units_on_order, reorder_level, discontinued)
VALUES (79, 'Another Beverage', 1, 12, 30, 0, 10, 0);
```

**Expected output:**
```
INSERT 0 1
```

**pgAdmin (Connection 1):**
```sql
-- Count again
SELECT COUNT(*) FROM products WHERE category_id = 1;
COMMIT;
```

**Expected output:**
```
count
-----
14
```

**Why**: Read Committed sees new rows (phantom reads allowed). The count changed from 13 to 14 within the same transaction.

---

## Demo 5: Serializable isolation level

**pgAdmin (Connection 1):**
```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

SELECT SUM(units_in_stock) FROM products WHERE category_id = 2;
```

**Expected output:**
```
sum
----
79
```

**psql (Connection 2):**
```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

SELECT SUM(units_in_stock) FROM products WHERE category_id = 2;
-- Then update based on that sum
UPDATE products SET units_in_stock = units_in_stock + 10 WHERE category_id = 2;
COMMIT;
```

**Expected output:**
```
sum
----
79

UPDATE 4
COMMIT
```

**pgAdmin (Connection 1):**
```sql
-- Try to update based on our read
UPDATE products SET units_in_stock = units_in_stock + 5 WHERE category_id = 2;
```

**Expected output:**
```
ERROR:  could not serialize access due to concurrent update
```

**Why**: Serializable detects conflicts. Connection 1's transaction would produce different results if executed serially, so PostgreSQL aborts it to maintain serializability.

---

## Does PostgreSQL support Read Uncommitted?

```sql
-- Try to set Read Uncommitted
BEGIN TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SHOW transaction_isolation;
```

**Expected output:**
```
transaction_isolation
---------------------
read uncommitted
```

**However, test the actual behavior:**

**pgAdmin (Connection 1):**
```sql
BEGIN;
UPDATE products SET unit_price = 99.99 WHERE product_id = 1;
-- Do NOT commit
```

**psql (Connection 2):**
```sql
BEGIN TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT unit_price FROM products WHERE product_id = 1;
COMMIT;
```

**Expected output:**
```
unit_price
----------
25
```

**Why**: PostgreSQL accepts the `READ UNCOMMITTED` syntax but **implements it as READ COMMITTED**. PostgreSQL does not support dirty reads. You cannot see uncommitted changes from other transactions.

**pgAdmin (Connection 1):**
```sql
ROLLBACK;
```

---

## Cleanup

```sql
-- Reset modified data
UPDATE products SET unit_price = 18 WHERE product_id = 1;
UPDATE products SET units_in_stock = 17 WHERE product_id = 2;
UPDATE products SET unit_price = 10 WHERE product_id = 3;
DELETE FROM products WHERE product_id IN (78, 79);
```

---

## Isolation level summary

| Isolation Level          | Dirty Read   | Non-Repeatable Read | Phantom Read  | PostgreSQL Support           |
| ------------------------ | ------------ | ------------------- | ------------- | ---------------------------- |
| Read Uncommitted         | Possible     | Possible            | Possible      | **Mapped to Read Committed** |
| Read Committed (default) | Not possible | Possible            | Possible      | ✓ Supported                  |
| Repeatable Read          | Not possible | Not possible        | Not possible* | ✓ Supported                  |
| Serializable             | Not possible | Not possible        | Not possible  | ✓ Supported                  |

*PostgreSQL's Repeatable Read is stronger than the SQL standard—it prevents phantom reads using MVCC.