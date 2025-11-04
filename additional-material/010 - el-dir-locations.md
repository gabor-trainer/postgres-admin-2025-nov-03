# Directories in a PostgreSQL installation on EL

#### **Default Binary Locations**

- Executables (e.g., `psql`, `pg_ctl`, `initdb`):
  - `/usr/bin/`
  - Example: `/usr/bin/psql`

#### **Configuration Files**

- Main configuration files (`postgresql.conf`, `pg_hba.conf`, `pg_ident.conf`):
  - `/var/lib/pgsql/data/`
  - Example: `/var/lib/pgsql/data/postgresql.conf`

#### **Data Directory**

- Default data directory (where the database files are stored):
  - `/var/lib/pgsql/data/`
  - Example: `/var/lib/pgsql/data/base/`

#### **Log Files**

- Default location for PostgreSQL log files:
  - `/var/lib/pgsql/pg_log/` (if enabled in configuration)
  - Logs may also be managed by `systemd` or the system logger (`journalctl`).

#### **Systemd Service Files**

- Service configuration:
  - `/usr/lib/systemd/system/postgresql.service`

#### **Socket Files**

- Default location for the PostgreSQL socket:
  - `/var/run/postgresql/` or `/tmp/`
