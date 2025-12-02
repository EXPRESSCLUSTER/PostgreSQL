# PostgreSQL 18 with EXPRESSCLUSTER X on Linux

## About This Guide

This guide describes how to setup PostgreSQL 18 with EXPRESSCLUSTER X (ECX).

## Configuration

Two nodes mirror disk type cluster will be created. The database saved on the ECX data partition will be replicated between 2 servers.

In this guide, a mirror disk type cluster is focused, but shared disk type clusters can also be created.

The tested versions and configurations are as follows:

### Software versions

- RedHat Linux release 9.0
  - Kernel version 5.14.0-70.22.1.el9_0.x86_64
- PostgreSQL 18.x
- EXPRESSCLUSTER X 5.3 for Linux (internal version: 5.3.1)

### Cluster configuration

- Group resource
  - Floating IP address resource
  - Mirror disk resource
  - Exec resource
- Monitor resource
  - Floating IP monitor resource
  - Mirror connect monitor resource
  - Mirror disk monitor resource
  - PostgreSQL monitor resource
  - User-mode monitor resource

---

# Setup

## Setup a basic cluster

### On both servers

1. Install ECX
   Refer:
   https://github.com/EXPRESSCLUSTER/BasicCluster/blob/master/X41/Lin/2nodesMirror_Lin.md

2. Register licenses:
   - EXPRESSCLUSTER X for Linux
   - EXPRESSCLUSTER X Replicator for Linux
   - EXPRESSCLUSTER X Database Agent for Linux
3. Reboot the server

---

### On Primary server

1. Create a cluster
2. Add one failover group with FIP and MD resources

   - Failover group:
     - fip
     - md
       - Mount Point: /postgres
       - Data Partition: /dev/sbd1
       - Cluster Partition: /dev/sdc1

3. Apply configuration
4. Start failover group on Primary server

---

# User settings for PostgreSQL

### On both servers

1. Create a user for PostgreSQL
   **The user ID must be the same on both servers.**

   ```bash
   useradd -u 26 postgres
   ```

2. Confirm identity

   ```bash
   id postgres
   ```

3. Set password

   ```bash
   passwd postgres
   ```

---

# Install PostgreSQL 18

If PostgreSQL 18 is already installed on both servers, skip.

### On both servers

1. Visit:
   https://www.postgresql.org/download/linux/redhat/#yum

2. Select:
   - Version: 18
   - Architecture: x86_64

3. Install PostgreSQL 18:

```bash
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo dnf -qy module disable postgresql
sudo dnf install -y postgresql18-server
sudo /usr/pgsql-18/bin/postgresql-18-setup initdb
sudo systemctl enable postgresql-18
sudo systemctl start postgresql-18
```

---

# PostgreSQL Setup

## On Primary server

1. Create database directory on ECX mirror disk

   ```bash
   mkdir -p /postgres/data
   ```

---

### If using existing PostgreSQL installation

1. Check current data directory:

   ```sql
   psql
   SHOW data_directory;
   ```

2. Stop service

   ```bash
   systemctl stop postgresql-18
   ```

3. Copy data to ECX mirror disk

   ```bash
   cp -r /var/lib/pgsql/18/data/* /postgres/data
   ```

4. Update data location in postgresql.conf

   File: /postgres/data/postgresql.conf

   ```
   data_directory = '/postgres/data/'
   ```

---

## Set permissions

```bash
chown -R postgres:postgres /postgres/data
chmod 700 /postgres/data
```

---

## Update systemd service file (if needed)

File: /usr/lib/systemd/system/postgresql-18.service

```
Environment=PGDATA=/postgres/data
```

Then reload & restart:

```bash
systemctl daemon-reload
systemctl restart postgresql-18.service
```

---

## Initialize PostgreSQL (only for first-time setup)

```bash
sudo su - postgres
/usr/pgsql-18/bin/initdb -D /postgres/data -E UTF8 --no-locale -W
```

After success, PostgreSQL start command:

```bash
/usr/pgsql-18/bin/pg_ctl -D /postgres/data -l logfile start
```

---

## Edit postgresql.conf

Location: /postgres/data/postgresql.conf

### Allow remote connections

**Before**
```
#listen_addresses = 'localhost'
```

**After**
```
listen_addresses = '*'
```

### Port configuration (optional)

```
#port = 5432
```

---

## Edit pg_hba.conf

Location: /postgres/data/pg_hba.conf

**Before**
```
host    all    all    127.0.0.1/32    trust
```

**After**
```
host    all    all    127.0.0.1/32    md5
```

Client subnet example:
```
host    all    all    192.168.1.0/24    md5
```

---

## Start the database

```bash
/usr/pgsql-18/bin/pg_ctl start -D /postgres/data -l /dev/null
```

---

## Create a database (if first setup)

```bash
/usr/pgsql-18/bin/createdb -h localhost -U postgres db1
```

---

## Test connection

```bash
/usr/pgsql-18/bin/psql -h localhost -U postgres db1
```

---

## Stop server

```bash
/usr/pgsql-18/bin/pg_ctl stop -D /postgres/data -m fast
```

---

## Disable automatic startup

```bash
systemctl disable postgresql-18.service
```

---

# On Secondary Server

1. Move failover group to secondary server
2. Check startup type

   ```bash
   systemctl list-unit-files postgresql-18.service
   ```

3. Disable auto-start

   ```bash
   systemctl disable postgresql-18.service
   ```

4. Start Postgres

   ```bash
   /usr/pgsql-18/bin/pg_ctl start -D /postgres/data -l /dev/null
   ```

5. Test connection

   ```bash
   /usr/pgsql-18/bin/psql -h localhost -U postgres db1
   ```

6. Stop DB

   ```bash
   /usr/pgsql-18/bin/pg_ctl stop -D /postgres/data -m fast
   ```

---

# Add PostgreSQL Resources to ECX

### On Primary Server

## Exec resource configuration

| Parameter | Value |
|----------|-------|
| SUUSER | postgres |
| PGINST | /usr/pgsql-18 |
| PGDATA | /postgres/data |
| PGPORT | 5432 |

---

# PostgreSQL Monitor Resource

| Field | Value |
|-------|-------|
| Target Resource | exec resource |
| Database Name | db1 |
| IP Address | 127.0.0.1 |
| Port | 5432 |
| Username | postgres |
| Password | postgres user's password |
| Table | psqlwatch |
| Library Path | /usr/pgsql-18/lib/libpq.so.5.18 |
| Recovery Target | exec resource |

---

# If Database Agent License Not Available

Use customer monitor resource with genw.sh.

| Parameter | Value |
|----------|-------|
| SUUSER | postgres |
| PGINST | /usr/pgsql-18 |
| PGDATA | /postgres/data |

---

# Sample Scripts

## start.sh
```sh
SUUSER="postgres"
PGINST="/usr/pgsql-18"
PGDATA="/postgres/data"
PGPORT="5432"
```
*(Rest of script remains unchanged)*

---

## stop.sh
```sh
SUUSER="postgres"
PGINST="/usr/pgsql-18"
PGDATA="/postgres/data"
```
*(Rest of script remains unchanged)*

---

## genw.sh
```sh
SUUSER="postgres"
PGINST="/usr/pgsql-18"
PGDATA="/postgres/data"
```

---

