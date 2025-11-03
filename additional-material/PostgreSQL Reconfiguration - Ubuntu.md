## PostgreSQL Reconfiguration - Ubuntu

### Prerequisites
```bash
# Verify PostgreSQL is installed and running
sudo systemctl status postgresql

# Find your PostgreSQL version
ls /etc/postgresql/
# Example output: 14, 15, or 18
```

---

### Step 1: Test Default Connection

```bash
# Connect as postgres Unix user (default peer authentication)
sudo -u postgres psql

# Should connect successfully with prompt:
# postgres=#

# Check current settings
\conninfo
# Shows: socket connection, port 5432, user postgres

# Exit
\q
```

---

### Step 2: Change Port to 15432

**Edit postgresql.conf:**
```bash
# Find config file (replace 18 with your version)
sudo nano /etc/postgresql/18/main/postgresql.conf
```

**Find and modify:**
```ini
# Around line 63-64
port = 5432
```

**Change to:**
```ini
port = 15432
```

**Save:** `Ctrl+O`, `Enter`, `Ctrl+X`

---

### Step 3: Configure Password Authentication

**Edit pg_hba.conf:**
```bash
sudo nano /etc/postgresql/18/main/pg_hba.conf
```

**Find this section (near bottom):**
```
# "local" is for Unix domain socket connections only
local   all             all                                     peer
# IPv4 local connections:
host    all             all             127.0.0.1/32            scram-sha-256
# IPv6 local connections:
host    all             all             ::1/128                 scram-sha-256
```

**Change to:**
```
# "local" is for Unix domain socket connections only
local   all             postgres                                md5
local   all             all                                     peer
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5
# IPv6 local connections:
host    all             all             ::1/128                 md5
```

**Save:** `Ctrl+O`, `Enter`, `Ctrl+X`

---

### Step 4: Set postgres User Password

```bash
# Connect before restarting (still using old port/auth)
sudo -u postgres psql

# Set password
ALTER USER postgres WITH PASSWORD 'password';

# Verify
\du
# Should show postgres with Superuser role

# Exit
\q
```

---

### Step 5: Restart PostgreSQL

```bash
# Restart service to apply all changes
sudo systemctl restart postgresql

# Verify it's running on new port
sudo ss -tlnp | grep 15432
# Should show: LISTEN 0 244 127.0.0.1:15432

# Check service status
sudo systemctl status postgresql
```

---

### Step 6: Test New Configuration

**Test with password authentication:**
```bash
# Method 1: Connect to localhost with password prompt
psql -U postgres -h localhost -p 15432
# Enter password: password

# Method 2: Inline password (for scripts only)
PGPASSWORD=password psql -U postgres -h localhost -p 15432

# Method 3: Still works with sudo (unix socket)
sudo -u postgres psql -p 15432
```

**Verify connection info:**
```sql
\conninfo
-- Should show: TCP/IP connection, port 15432, user postgres

\q
```

---

### Verification Checklist

```bash
# ✓ Service running
sudo systemctl is-active postgresql

# ✓ Listening on correct port
sudo ss -tlnp | grep postgres | grep 15432

# ✓ Password auth works
psql -U postgres -h localhost -p 15432 -c "SELECT version();"

# ✓ Socket auth works
sudo -u postgres psql -p 15432 -c "SELECT current_user;"
```

---

### Common Issues & Fixes

**Issue: Port already in use**
```bash
# Check what's using the port
sudo ss -tlnp | grep 15432
# Kill process or choose different port
```

**Issue: "FATAL: password authentication failed"**
```bash
# Verify password was set
sudo -u postgres psql -p 15432
ALTER USER postgres WITH PASSWORD 'password';
\q

# Verify pg_hba.conf has md5, not peer
sudo grep "local.*postgres" /etc/postgresql/18/main/pg_hba.conf
```

**Issue: Connection refused**
```bash
# Check PostgreSQL is running
sudo systemctl status postgresql

# Check logs
sudo journalctl -u postgresql -n 50
```

**Reset to defaults:**
```bash
# Restore original configs
sudo cp /etc/postgresql/18/main/postgresql.conf.bak /etc/postgresql/18/main/postgresql.conf
sudo cp /etc/postgresql/18/main/pg_hba.conf.bak /etc/postgresql/18/main/pg_hba.conf
sudo systemctl restart postgresql
```

---

### Quick Reference Card

```bash
# Connect methods after configuration:
psql -U postgres -h localhost -p 15432              # With password prompt
PGPASSWORD=password psql -U postgres -h 127.0.0.1 -p 15432  # Inline password
sudo -u postgres psql -p 15432                      # Socket (no password)

# Config files:
/etc/postgresql/18/main/postgresql.conf             # Port settings
/etc/postgresql/18/main/pg_hba.conf                 # Authentication

# Service management:
sudo systemctl restart postgresql
sudo systemctl status postgresql
sudo journalctl -u postgresql -f                    # Live logs
```