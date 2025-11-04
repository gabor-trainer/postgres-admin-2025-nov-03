# PostgreSQL `pg_catalog` Cheat Sheet

> Purpose: the internal system catalog schema. Tables and views here describe every database object (tables, indexes, columns, types, functions, privileges, dependencies, etc.). Querying `pg_catalog` lets you build powerful introspection queries without relying on `information_schema`.

---

## Concepts

| Concept                                                        | Where to look                         | Notes                                                            |
| -------------------------------------------------------------- | ------------------------------------- | ---------------------------------------------------------------- |
| Schemas                                                        | `pg_namespace`                        | `nspname`, `oid`                                                 |
| Relations (tables, views, matviews, sequences, indexes, TOAST) | `pg_class`                            | `relkind`, `reltype`, `relnamespace`, `reltoastrelid`            |
| Columns                                                        | `pg_attribute`                        | One row per column; `attrelid`, `attname`, `atttypid`, `attnum`  |
| Data types                                                     | `pg_type`                             | `typname`, `typtype`, `typcategory`, `typarray`                  |
| Indexes                                                        | `pg_index` + `pg_class`               | Column/expr positions: `indkey`, `indnkeyatts`, `indoption`      |
| Constraints                                                    | `pg_constraint`                       | `contype`, `conrelid`, `confrelid`, `conkey`, `confkey`          |
| Functions                                                      | `pg_proc`                             | `proname`, `prokind`, `prorettype`, `proargtypes`                |
| Operators                                                      | `pg_operator`                         | `oprname`, `oprleft`, `oprright`, `oprresult`                    |
| Access methods                                                 | `pg_am`, `pg_amop`, `pg_amproc`       | Index AMs, operator classes/families                             |
| Roles & privileges                                             | `pg_roles`/`pg_authid`, `pg_shdepend` | Use `pg_roles` unless superuser                                  |
| Tablespaces                                                    | `pg_tablespace`                       | `spcname`, `spcoptions`                                          |
| Databases (cluster-wide)                                       | `pg_database`                         | `datname`, `datdba`, `encoding`                                  |
| Triggers & rules                                               | `pg_trigger`, `pg_rewrite`            | `tgrelid`, `ev_class`                                            |
| Dependencies                                                   | `pg_depend`                           | `classid/objid` → referenced; `refclassid/refobjid` → depends on |
| Comments                                                       | `pg_description`, `pg_shdescription`  | Text in `description`                                            |
| Settings                                                       | `pg_settings`                         | Name/value/current source                                        |
| Locks                                                          | `pg_locks`                            | Combine with `pg_stat_activity`                                  |

---

## Core Catalogs (essentials)

| Catalog         | What it stores                  | Key columns                                                                                   | Join keys                                                                | Common filters                                                                |
| --------------- | ------------------------------- | --------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------ | ----------------------------------------------------------------------------- |
| `pg_namespace`  | Schemas                         | `oid`, `nspname`                                                                              | `pg_class.relnamespace`, `pg_type.typnamespace`, `pg_proc.pronamespace`  | Exclude system: `nspname NOT LIKE 'pg_%' AND nspname <> 'information_schema'` |
| `pg_class`      | All relations                   | `oid`, `relname`, `relnamespace`, `relkind`, `relowner`, `reltoastrelid`                      | `pg_attribute.attrelid`, `pg_index.indexrelid`, `pg_constraint.conrelid` | `relkind IN ('r','p','v','m','f','S','I')`                                    |
| `pg_attribute`  | Columns                         | `attrelid`, `attname`, `atttypid`, `attnum`, `attnotnull`, `atthasdef`                        | `pg_type.oid = atttypid`                                                 | `attnum > 0 AND NOT attisdropped`                                             |
| `pg_type`       | Types                           | `oid`, `typname`, `typtype`, `typcategory`, `typrelid`, `typarray`                            | `pg_attribute.atttypid`, `pg_proc.prorettype`                            | User types: `typtype IN ('b','e','c','d','r','m')`                            |
| `pg_index`      | Index metadata                  | `indexrelid`, `indrelid`, `indisunique`, `indisprimary`, `indkey`                             | Join to `pg_class` twice (index/table)                                   | `indisvalid`, `indisready`                                                    |
| `pg_constraint` | Constraints                     | `oid`, `contype`, `conrelid`, `conindid`, `conkey`, `confrelid`, `confupdtype`, `confdeltype` | to table: `conrelid=pg_class.oid`, to index: `conindid=pg_class.oid`     | `contype IN ('p','u','f','c','x')`                                            |
| `pg_proc`       | Functions/aggregates/procedures | `oid`, `proname`, `prokind`, `prolang`, `prorettype`, `proargtypes`                           | `pronamespace=pg_namespace.oid`                                          | `prokind IN ('f','p','a','w')`                                                |
| `pg_roles`      | Roles                           | `rolname`, flags                                                                              | `pg_auth_members` for membership                                         | Exclude `pg_` roles                                                           |
| `pg_depend`     | Dependency graph                | `classid, objid` → object; `refclassid, refobjid` → dependency                                | To any catalog via `classid`                                             | `deptype IN ('n','a','i','p','e','x')`                                        |

---

## Handy Enums & Flags

| Field                     | Catalog         | Values (code → meaning)                                                                                                                                                                                 |
| ------------------------- | --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `relkind`                 | `pg_class`      | `r` table, `p` partitioned table, `v` view, `m` matview, `i` index, `I` partitioned index, `S` sequence, `t` TOAST, `f` foreign table                                                                   |
| `relpersistence`          | `pg_class`      | `p` permanent, `u` unlogged, `t` temp                                                                                                                                                                   |
| `typtype`                 | `pg_type`       | `b` base, `c` composite, `d` domain, `e` enum, `m` multirange, `p` pseudo, `r` range                                                                                                                    |
| `typcategory`             | `pg_type`       | `A` array, `B` boolean, `C` composite, `D` date/time, `E` enum, `G` geometric, `I` network, `N` numeric, `P` pseudo, `R` range, `S` string, `T` timespan, `U` user-defined, `V` bit-string, `X` unknown |
| `prokind`                 | `pg_proc`       | `f` function, `p` procedure, `a` aggregate, `w` window                                                                                                                                                  |
| `contype`                 | `pg_constraint` | `p` PK, `u` unique, `f` FK, `c` check, `x` exclusion                                                                                                                                                    |
| `confupdtype/confdeltype` | `pg_constraint` | `a` no action, `r` restrict, `c` cascade, `n` set null, `d` set default                                                                                                                                 |
| `deptype`                 | `pg_depend`     | `n` normal, `a` auto, `i` internal, `p` pin, `e` extension, `x` target extension                                                                                                                        |

---

## reg* OID Pseudotypes (make joins readable)

* Cast OIDs to names on the fly:* `regclass`, `regtype`, `regproc`, `regnamespace`, `regrole`, `regoperator`.

```sql
-- Example: resolve a relation OID to a qualified name
SELECT 12345::regclass;
```

This uses search_path—qualify with schema when ambiguous.

---

## Privileges (ACL) decoding

Privileges are stored in `aclitem[]` (e.g., `relacl`, `proacl`). Use helpers:

```sql
SELECT grantee, privilege_type, is_grantable
FROM   aclexplode(relacl) a
JOIN   pg_class c ON c.oid = 'public.my_table'::regclass;
```

Helpful views: `pg_roles`, `pg_namespace`, `pg_class`.

---

## Common Join Patterns

```sql
-- Tables with columns & types
SELECT n.nspname, c.relname, a.attnum, a.attname, format_type(a.atttypid,a.atttypmod) AS data_type
FROM   pg_class c
JOIN   pg_namespace n ON n.oid = c.relnamespace
JOIN   pg_attribute a ON a.attrelid = c.oid
WHERE  c.relkind IN ('r','p','v','m','f')
  AND  a.attnum > 0 AND NOT a.attisdropped;

-- Indexes on a table (names + definitions)
SELECT i.relname AS index_name, pg_get_indexdef(i.oid) AS ddl
FROM   pg_class t
JOIN   pg_index ix ON ix.indrelid = t.oid
JOIN   pg_class i ON i.oid = ix.indexrelid
WHERE  t.oid = 'public.my_table'::regclass;

-- Constraints on a table
SELECT con.conname, con.contype, pg_get_constraintdef(con.oid, true) AS defn
FROM   pg_constraint con
WHERE  con.conrelid = 'public.my_table'::regclass;

-- Foreign keys with referenced table/cols
SELECT con.conname,
       confrel::regclass AS referenced_table,
       conkey, confkey,
       confupdtype, confdeltype
FROM   pg_constraint con
WHERE  con.contype='f' AND con.conrelid='public.my_table'::regclass;

-- Functions with arg & return types
SELECT n.nspname, p.proname, p.prokind,
       pg_get_function_identity_arguments(p.oid) AS args,
       pg_get_function_result(p.oid) AS returns
FROM   pg_proc p
JOIN   pg_namespace n ON n.oid = p.pronamespace;

-- Object comments
SELECT objoid::regclass AS object, description
FROM   pg_description
WHERE  classoid = 'pg_class'::regclass;

-- Dependency graph (what depends on this table?)
SELECT d.refobjid::regclass AS dependant,
       d.objid::regclass     AS target,
       d.deptype
FROM   pg_depend d
WHERE  d.objid = 'public.my_table'::regclass;
```

---

## Performance/Planner Nuggets

| Topic              | Where                             | Tip                                                           |
| ------------------ | --------------------------------- | ------------------------------------------------------------- |
| Statistics         | `pg_statistic`, `pg_stats` (view) | Prefer `pg_stats` for readable histograms, most-common values |
| Storage parameters | `pg_class.reloptions`             | Use `pg_options_to_table(reloptions)` to explode              |
| Toast              | `pg_class.reltoastrelid`          | Join back to `pg_class` to inspect TOAST table                |
| Partitioning       | `pg_inherits`                     | Parent ↔ child links via `inhrelid`, `inhparent`              |

---

## Safety & Filtering

* User objects: filter out `nspname LIKE 'pg\_%' OR nspname = 'information_schema'`.
* Ignore dropped columns: `attisdropped = false` and `attnum > 0`.
* Use `format_type(atttypid, atttypmod)` for human-friendly column type.
* Many helpers: `pg_get_viewdef`, `pg_get_triggerdef`, `pg_get_serial_sequence`, `pg_get_expr`, `pg_size_pretty(pg_total_relation_size(...))`.

---

## Tiny Recipes

```sql
-- All user tables with row estimates & size
SELECT n.nspname, c.relname,
       c.reltuples::bigint AS est_rows,
       pg_size_pretty(pg_total_relation_size(c.oid)) AS total_size
FROM   pg_class c
JOIN   pg_namespace n ON n.oid = c.relnamespace
WHERE  c.relkind IN ('r','p')
  AND  n.nspname NOT LIKE 'pg\_%' AND n.nspname <> 'information_schema'
ORDER BY pg_total_relation_size(c.oid) DESC;

-- Columns with defaults
SELECT n.nspname, c.relname, a.attname,
       pg_get_expr(ad.adbin, ad.adrelid, true) AS default_expr
FROM   pg_attribute a
JOIN   pg_class c ON c.oid = a.attrelid
JOIN   pg_namespace n ON n.oid = c.relnamespace
JOIN   pg_attrdef ad ON ad.adrelid = a.attrelid AND ad.adnum = a.attnum
WHERE  a.attnum > 0 AND NOT a.attisdropped;

-- Roles and memberships
SELECT m.roleid::regrole AS role, m.member::regrole AS member, m.admin_option
FROM   pg_auth_members m;

-- Find objects owned by a role
SELECT c.oid::regclass AS object, c.relkind
FROM   pg_class c
WHERE  c.relowner = (SELECT oid FROM pg_roles WHERE rolname = 'alice');
```

---

## Version Notes

* Fields and flags can vary across versions; check your `SHOW server_version;`.
* `prokind`, partitioned relkinds (`p`, `I`) and multirange (`m`) exist in newer versions (≥13/14). Adjust queries for older clusters.

---

**Tip:** Prefer `pg_catalog.`-qualified names in system queries to avoid surprises with `search_path`. Example: `pg_catalog.pg_class`.
