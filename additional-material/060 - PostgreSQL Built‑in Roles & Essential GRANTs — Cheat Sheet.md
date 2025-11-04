# PostgreSQL Built‑in Roles & Essential GRANTs — Cheat Sheet
---

## Predefined (Built‑in) Roles — What they allow

| Role                          | Purpose (short)                                                               | Notes / Caution                                                      |
| ----------------------------- | ----------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| `pg_read_all_data`            | Read all tables, views, sequences                                             | Does **not** bypass RLS. Consider `BYPASSRLS` attribute if required. |
| `pg_write_all_data`           | Insert/Update/Delete on all tables, views, sequences                          | Does **not** bypass RLS.                                             |
| `pg_read_all_settings`        | Read all GUCs (config)                                                        | Monitoring/ops use.                                                  |
| `pg_read_all_stats`           | Read all `pg_stat_*` views                                                    | Monitoring/ops use.                                                  |
| `pg_stat_scan_tables`         | Run scanning monitor funcs that may lock                                      | Long `ACCESS SHARE` locks possible.                                  |
| `pg_monitor`                  | Convenience: member of the three above                                        | Grant for read‑only monitoring bundles.                              |
| `pg_signal_backend`           | Cancel/terminate other sessions                                               | Cannot signal superuser sessions. Operationally sensitive.           |
| `pg_signal_autovacuum_worker` | Cancel/terminate autovacuum worker on a table                                 | Operationally sensitive.                                             |
| `pg_use_reserved_connections` | Use slots reserved by `reserved_connections`                                  | Prevents lockout in high load.                                       |
| `pg_database_owner`           | Dynamic role: resolves to *current DB owner*                                  | Not grantable; owns `public` schema initially. Useful in templates.  |
| `pg_read_server_files`        | Read server‑side files via `COPY`/FDWs                                        | Effectively escalates to superuser if misused.                       |
| `pg_write_server_files`       | Write server‑side files                                                       | Same caution as above.                                               |
| `pg_execute_server_program`   | Execute server‑side programs                                                  | Same caution as above.                                               |
| `pg_create_subscription`      | Allow `CREATE SUBSCRIPTION` (logical repl.)                                   | Needs DB `CREATE` privilege too.                                     |
| `pg_checkpoint`               | Execute `CHECKPOINT`                                                          | Operational role.                                                    |
| `pg_maintain`                 | Execute `VACUUM/ANALYZE/REINDEX/CLUSTER/REFRESH MATERIALIZED VIEW/LOCK TABLE` | Acts like having `MAINTAIN` on all relations.                        |

*List reflects current docs for modern releases; older clusters may lack some roles.*

---

## Essential GRANTs by Object Type

| Object                 | Privileges                                                                              | Typical commands                                                                                                                                      |
| ---------------------- | --------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Database**           | `CONNECT`, `CREATE`, `TEMPORARY` (`TEMP`)                                               | `GRANT CONNECT ON DATABASE db TO role;`  •  `GRANT CREATE, TEMP ON DATABASE db TO role;`                                                              |
| **Schema**             | `USAGE`, `CREATE`                                                                       | `GRANT USAGE ON SCHEMA s TO role;`  •  `GRANT CREATE ON SCHEMA s TO role;`                                                                            |
| **Table/View/MatView** | `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `TRUNCATE`, `REFERENCES`, `TRIGGER`, `MAINTAIN` | `GRANT SELECT ON TABLE s.t TO role;`  •  `GRANT INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA s TO role;`  •  `GRANT MAINTAIN ON TABLE s.t TO role;` |
| **Sequence**           | `USAGE`, `SELECT`, `UPDATE`                                                             | `GRANT USAGE, SELECT, UPDATE ON SEQUENCE s.seq TO role;`                                                                                              |
| **Function/Procedure** | `EXECUTE`                                                                               | `GRANT EXECUTE ON FUNCTION s.f(argtypes) TO role;`                                                                                                    |
| **Type/Domain**        | `USAGE`                                                                                 | `GRANT USAGE ON TYPE s.mytype TO role;`                                                                                                               |
| **Language**           | `USAGE`                                                                                 | `GRANT USAGE ON LANGUAGE plpython3u TO role;`                                                                                                         |
| **Large Object**       | `SELECT`, `UPDATE`                                                                      | `GRANT SELECT, UPDATE ON LARGE OBJECT loid TO role;`                                                                                                  |

**With grant option:** append `WITH GRANT OPTION` to let grantees grant further.

---

## Using Built‑in Roles (practical)

```sql
-- Monitoring‑only user
CREATE ROLE monitor LOGIN PASSWORD '...';
GRANT pg_monitor TO monitor;           -- includes read_all_settings/stats/stat_scan
GRANT CONNECT ON DATABASE appdb TO monitor;

-- Read‑only across everything (no RLS bypass)
CREATE ROLE readonly LOGIN;
GRANT pg_read_all_data TO readonly;

-- App writer across everything
CREATE ROLE writer LOGIN;
GRANT pg_write_all_data TO writer;

-- Kill queries but not superusers
GRANT pg_signal_backend TO ops;

-- Maintenance operator without superuser
GRANT pg_maintain TO maint;
```

---

## Default Privileges (future objects)

```sql
-- New tables created by alice in schema s become readable by role r
ALTER DEFAULT PRIVILEGES FOR ROLE alice IN SCHEMA s
  GRANT SELECT ON TABLES TO r;

-- For all schemas owned by alice
ALTER DEFAULT PRIVILEGES FOR ROLE alice
  GRANT EXECUTE ON FUNCTIONS TO r;
```

**Notes**

* Default privileges apply only to objects created *after* the command.
* They are per‑owner, per‑schema, per‑object‑type.

---

## Handy Recipes

```sql
-- Grant full DML on all existing tables in selected schemas
DO $$
DECLARE r text; BEGIN
  FOR r IN SELECT quote_ident(nspname)
           FROM pg_namespace
           WHERE nspname NOT LIKE 'pg\_%' AND nspname <> 'information_schema'
  LOOP
    EXECUTE format('GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA %s TO app_role', r);
    EXECUTE format('GRANT USAGE, SELECT, UPDATE ON ALL SEQUENCES IN SCHEMA %s TO app_role', r);
  END LOOP;
END $$;

-- Revoke public creation on public schema (PG15+ default is already restricted)
REVOKE CREATE ON SCHEMA public FROM PUBLIC;

-- Show effective table privileges
SELECT grantee, table_schema, table_name, privilege_type
FROM information_schema.role_table_grants
WHERE grantee IN ('app_role');

-- Who can terminate sessions?
SELECT r.rolname FROM pg_roles r
JOIN pg_auth_members m ON m.roleid = 'pg_signal_backend'::regrole AND m.member = r.oid;
```

---

## Quick Checks & Tips

* Verify predefined roles on your server:

  * `\du+ pg_*` in `psql`, or `SELECT rolname FROM pg_roles WHERE rolname LIKE 'pg_%' ORDER BY 1;`
* RLS interaction: built‑ins don’t bypass RLS. Use role attribute `BYPASSRLS` sparingly.
* Prefer granting to app roles (groups), not individual users.
* Combine database `CONNECT` with schema `USAGE` and table `SELECT` for read‑only access that actually works.
* For logical replication subscribers, add `pg_create_subscription` and ensure `CREATE` on the database.

---

**Version footnote:** `pg_database_owner`, `pg_read_all_data`, `pg_write_all_data` appear in PG14+. `pg_maintain`, `pg_checkpoint`, `pg_use_reserved_connections`, `pg_signal_autovacuum_worker` are present in newer releases; if your cluster is older, omit them accordingly.
