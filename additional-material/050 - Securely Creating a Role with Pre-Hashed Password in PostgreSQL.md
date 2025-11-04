# Securely Creating a Role with Pre-Hashed Password in PostgreSQL

## Step 1: Generate the encrypted password hash

```sql
-- ============================================================================
-- DBA INTERACTIVE STEP: Generate password hash
-- ============================================================================
-- The DBA executes this query to generate the SCRAM-SHA-256 hash
-- for the plaintext password 'password'
-- This is done interactively so the plaintext password never appears in scripts

SELECT encode(
    digest(
        'password' || 'northwind',  -- password + username (salt)
        'sha256'
    ),
    'hex'
);

-- BETTER METHOD: Use PostgreSQL's built-in SCRAM-SHA-256 function
-- This generates the proper SCRAM-SHA-256 hash that PostgreSQL expects
-- Execute this in psql:

SELECT 
    pg_catalog.pg_get_userbyid(0),  -- Just to test function availability
    'Use the method below instead...';

-- RECOMMENDED: Use psql to generate the hash
-- Connect to database and run:
-- \set password 'password'
-- SELECT pg_catalog.scram_sha_256_password(:'password', 'northwind');

-- Or use this programmatic approach:
DO $$
DECLARE
    hashed_password text;
BEGIN
    -- Generate SCRAM-SHA-256 hash for password 'password'
    hashed_password := pg_catalog.scram_sha_256_password('password', 'northwind');
    RAISE NOTICE 'Generated hash: %', hashed_password;
END $$;
```

## Step 2: Copy the hash and use it in CREATE ROLE

```sql
-- ============================================================================
-- CREATE ROLE STATEMENT WITH PRE-HASHED PASSWORD
-- ============================================================================
-- After generating the hash interactively, the DBA copies the hash value
-- and pastes it into this statement. This ensures:
--   1. The plaintext password 'password' never appears in version control
--   2. The script can be safely stored in repositories
--   3. The password is already in SCRAM-SHA-256 format

CREATE ROLE northwind WITH
  LOGIN
  NOSUPERUSER
  INHERIT
  NOCREATEDB
  NOCREATEROLE
  NOREPLICATION
  NOBYPASSRLS
  -- The DBA copies the generated hash below (starts with 'SCRAM-SHA-256$...')
  -- This hash represents the encrypted password 'password'
  ENCRYPTED PASSWORD 'SCRAM-SHA-256$4096:G2FfNLWXeXiz/58N/54bzA==$lEqoD3UA7umJW8At+WurMiEeYCscPYb0KsiABa+gQPI=:q2wCms4eN0rH+AR6aoHw76Mu7cPBeZDcafDr3ujnrb0=';

-- Note: When PostgreSQL sees a password string starting with 'SCRAM-SHA-256$',
-- it recognizes this as an already-hashed password and stores it as-is
-- without re-hashing it.
```

## Complete DBA workflow with comments:

```sql
-- ============================================================================
-- SECURE PASSWORD HASH GENERATION WORKFLOW
-- ============================================================================
-- Best practice for DBAs when creating roles with passwords
--
-- WHY THIS APPROACH?
-- - Plaintext passwords should NEVER be committed to version control
-- - Passwords should be hashed on a secure workstation before scripting
-- - This allows role creation scripts to be safely shared and stored
-- ============================================================================

-- ----------------------------------------------------------------------------
-- STEP 1: DBA executes this interactively on a secure workstation
-- ----------------------------------------------------------------------------
-- Option A: Use psql variables (RECOMMENDED)
\set password 'password'
SELECT 'SCRAM-SHA-256$' || encode(
    digest(:'password' || 'northwind', 'sha256'), 
    'base64'
);

-- Option B: Generate using PostgreSQL's internal function (PostgreSQL 14+)
-- Note: This requires appropriate extensions/functions
SELECT pg_catalog.scram_sha_256_password('password', 'northwind');

-- The DBA then:
-- 1. Copies the output hash (entire 'SCRAM-SHA-256$...' string)
-- 2. Securely stores or immediately pastes into script below
-- 3. Clears terminal history if needed for security

-- ----------------------------------------------------------------------------
-- STEP 2: DBA pastes the hash into this script
-- ----------------------------------------------------------------------------
-- This script can now be safely committed to version control

CREATE ROLE northwind WITH
  LOGIN
  NOSUPERUSER
  INHERIT
  NOCREATEDB
  NOCREATEROLE
  NOREPLICATION
  NOBYPASSRLS
  -- Hash generated in Step 1 pasted here:
  ENCRYPTED PASSWORD 'SCRAM-SHA-256$4096:G2FfNLWXeXiz/58N/54bzA==$lEqoD3UA7umJW8At+WurMiEeYCscPYb0KsiABa+gQPI=:q2wCms4eN0rH+AR6aoHw76Mu7cPBeZDcafDr3ujnrb0=';

-- ----------------------------------------------------------------------------
-- VERIFICATION
-- ----------------------------------------------------------------------------
-- Verify the role was created successfully
\du northwind

-- Test the password works (from psql command line):
-- psql -h localhost -p 5432 -U northwind -d postgres
-- (Enter password: password)
```

## Summary of the DBA workflow:

1. **Generate hash interactively** - DBA runs the hash generation query on their secure workstation
2. **Copy the hash** - DBA copies the entire `SCRAM-SHA-256$...` string from the output
3. **Paste into script** - DBA pastes the hash into the CREATE ROLE statement
4. **Commit safely** - The script now contains only the hash, never the plaintext password
5. **Execute script** - The script can be run in any environment without exposing the password

This is a security best practice for production environments where scripts are stored in version control or shared among team members.