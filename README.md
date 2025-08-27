# PostgreSQL Upgrade Guide (14 → 17)

This document provides a **step-by-step guide** to safely upgrade a PostgreSQL cluster from **version 14** to **version 17** using the `pg_upgrade` tool.  
It follows best practices and includes **dry-run validation** before performing the actual upgrade.

---

## 🚀 1. Install PostgreSQL 14 & 17

```bash
sudo apt update
sudo apt install postgresql-14 postgresql-17 -y
```

Check:
```bash
pg_lsclusters
```

---

## 📂 2. Prepare PostgreSQL 17 Cluster

```bash
sudo pg_createcluster 17 main --datadir=/var/lib/postgresql/17/main --start-conf=manual
sudo mkdir -p /var/lib/postgresql/17/main
sudo chown -R postgres:postgres /var/lib/postgresql/17
sudo -u postgres /usr/lib/postgresql/17/bin/initdb -D /var/lib/postgresql/17/main
```

---

## 🛑 3. Stop PostgreSQL 14 Cluster

```bash
sudo pg_ctlcluster 14 main stop
```

---

## 🔍 4. Dry-Run Check

```bash
sudo -u postgres /usr/lib/postgresql/17/bin/pg_upgrade   -b /usr/lib/postgresql/14/bin   -B /usr/lib/postgresql/17/bin   -d /var/lib/postgresql/14/main   -D /var/lib/postgresql/17/main   -o "-c config_file=/etc/postgresql/14/main/postgresql.conf       -c hba_file=/etc/postgresql/14/main/pg_hba.conf       -c ident_file=/etc/postgresql/14/main/pg_ident.conf       -c shared_preload_libraries=''"   -O "-c config_file=/etc/postgresql/17/main/postgresql.conf       -c hba_file=/etc/postgresql/17/main/pg_hba.conf       -c ident_file=/etc/postgresql/17/main/pg_ident.conf       -c shared_preload_libraries='timescaledb'       -c unix_socket_permissions=0700       -c unix_socket_directories='/tmp'"   --check
```

---

## ⚡ 5. Actual Upgrade

```bash
sudo -u postgres /usr/lib/postgresql/17/bin/pg_upgrade \
  -b /usr/lib/postgresql/14/bin \
  -B /usr/lib/postgresql/17/bin \
  -d /var/lib/postgresql/14/main \
  -D /var/lib/postgresql/17/main \
  -o "-c config_file=/etc/postgresql/14/main/postgresql.conf \
      -c hba_file=/etc/postgresql/14/main/pg_hba.conf \
      -c ident_file=/etc/postgresql/14/main/pg_ident.conf \
      -c shared_preload_libraries=''" \
  -O "-c config_file=/etc/postgresql/17/main/postgresql.conf \
      -c hba_file=/etc/postgresql/17/main/pg_hba.conf \
      -c ident_file=/etc/postgresql/17/main/pg_ident.conf \
      -c shared_preload_libraries='timescaledb' \
      -c unix_socket_permissions=0700 \
      -c unix_socket_directories='/tmp'" \
  -j "$(nproc)" \
  --link

```

---

## ▶️ 6. Start PostgreSQL 17

```bash
sudo pg_ctlcluster 17 main start
pg_lsclusters
```

---

## 🧪 7. Post-Upgrade

```bash
sudo -u postgres ./analyze_new_cluster.sh
sudo -u postgres ./delete_old_cluster.sh   # after validation
```

---

## 🛠️ Handling TimescaleDB Extension Errors

During `pg_upgrade --check` you might see:

```
Failure, exiting
could not load library "$libdir/timescaledb": FATAL:  extension "timescaledb" must be preloaded
```

### Cause
- TimescaleDB installed in PostgreSQL 14 but not configured in 17.
- Both clusters must use the **same TimescaleDB version**.

### Fix

1. Install the same version on both clusters (example `2.19.3`):
   ```bash
   sudo apt install timescaledb-2-postgresql-14=2.19.3* timescaledb-2-loader-postgresql-14
   sudo apt install timescaledb-2-postgresql-17=2.19.3* timescaledb-2-loader-postgresql-17
   ```

2. Upgrade PG14 extensions:
   ```sql
   ALTER EXTENSION timescaledb UPDATE TO '2.19.3';
   \dx timescaledb
   ```

3. Enable preload in PG17 config:
   ```conf
   shared_preload_libraries = 'timescaledb'
   ```

4. Re-run check with proper flags (old empty, new = timescaledb).

If versions match and preload is enabled, upgrade succeeds.
