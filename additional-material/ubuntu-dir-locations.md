# Directories in a PostgreSQL installation on Ubuntu

### 1\. **Configuration Files**

- **Path:** `/etc/postgresql/<version>/main/`
- **Description:** Contains the main configuration files:
  - `postgresql.conf`: Main server configuration.
  - `pg_hba.conf`: Client authentication settings.
  - `pg_ident.conf`: User mapping file.

### 2\. **Database Cluster Data**

- **Path:** `/var/lib/postgresql/<version>/main/`
- **Description:** The directory where the PostgreSQL database cluster (actual data) is stored.

### 3\. **Logs**

- **Path:** `/var/log/postgresql/`
- **Description:** Contains PostgreSQL log files.

### 4\. **Binaries**

- **Path:** `/usr/lib/postgresql/<version>/bin/`
- **Description:** The directory where PostgreSQL binary files (e.g., `psql`, `postgres`) are located.

### 5\. **Service Management**

- **Path:** `/lib/systemd/system/postgresql.service`
- **Description:** The systemd service file for managing the PostgreSQL service.

### 6\. **User Home Directory**

- **Path:** `/var/lib/postgresql/`
- **Description:** The home directory for the `postgres` system user.

### 7\. **Extensions and Shared Libraries**

- **Path:** `/usr/share/postgresql/<version>/`
- **Description:** Contains extensions, templates, and shared files for PostgreSQL.

---

### Checking Active Paths on Your System:

To verify the actual paths used on your system, you can use the following commands:

1.  **Locate the configuration files:**

    ```
    sudo -u postgres psql -c "SHOW config_file;"

    ```

2.  **Locate the data directory:**

    ```
    sudo -u postgres psql -c "SHOW data_directory;"

    ```

3.  **Locate the log directory:**

    ```
    sudo -u postgres psql -c "SHOW log_directory;"
    ```
