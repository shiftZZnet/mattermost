# Mattermost Enterprise Installation (HA)  
**Technical Documentation**

Author:		Joachim Baumgartner, Senior Technical Account Manager
Date:		17/09/2026

---

# Mattermost Read Replica Setup Runbook

**Applies to:** Mattermost Server v11.7.8 · HA deployment · PostgreSQL primary/secondary cluster  
**Goal:** Enable the `DataSourceReplicas` (read replica) and `DataSourceSearchReplicas` features in Mattermost's `config.json`.

---

## Table of Contents

1. [Preliminary Checks](#1-preliminary-checks)
   - 1.1 [Verify the Mattermost process user and home directory](#11-verify-the-mattermost-process-user-and-home-directory)
   - 1.2 [Verify the MMENVIRONMENT file location is upgrade-safe](#12-verify-the-mmenvironment-file-location-is-upgrade-safe)
   - 1.3 [Verify config.json is migrated to the database](#13-verify-configjson-is-migrated-to-the-database)
   - 1.4 [Verify the database DSN is bootstrapped in systemd](#14-verify-the-database-dsn-is-bootstrapped-in-systemd)
   - 1.5 [Verify the local config.json is no longer active](#15-verify-the-local-configjson-is-no-longer-active)
   - 1.6 [Remediation steps if checks fail](#16-remediation-steps-if-checks-fail)
2. [Enable Read Replicas in config.json](#2-enable-read-replicas-in-configjson)
3. [Final Checks](#3-final-checks)
   - 3.1 [Restart Mattermost and verify service health](#31-restart-mattermost-and-verify-service-health)
   - 3.2 [Verify read replica usage in Mattermost logs](#32-verify-read-replica-usage-in-mattermost-logs)
   - 3.3 [Verify query routing on the database side](#33-verify-query-routing-on-the-database-side)

---

## 1. Preliminary Checks

Before enabling the read replica feature, confirm the following on **both** Mattermost application servers:

- The Mattermost process runs as a dedicated system user whose home directory is **not** the binary installation directory.
- The `MMENVIRONMENT` file lives **outside** `/opt/mattermost/` so it cannot be overwritten or deleted during a binary upgrade.
- The `config.json` has been migrated to the database (DB config store is active).
- The systemd unit — or an `MMENVIRONMENT` file it sources — bootstraps the database DSN so Mattermost knows where to load the config from.
- The local `config.json` file on the filesystem has been renamed or removed so it cannot override the DB config.

---

### 1.1 Verify the Mattermost process user and home directory

Check which user the Mattermost process is running as:

```bash
ps aux | grep -E '[m]attermost'
```

Note the username in the first column. Then check that user's home directory:

```bash
getent passwd mattermost | cut -d: -f6
```

**Expected:** The home directory should be something like `/home/mattermost` or `/var/lib/mattermost` — i.e. **not** `/opt/mattermost`.

**Why this matters:** If the user's `$HOME` is set to `/opt/mattermost` (a common default when the user is created alongside the installation), any files stored there — including the `MMENVIRONMENT` file — risk being overwritten or deleted when upgrading the Mattermost binaries, since upgrades typically replace the contents of `/opt/mattermost`.

If the home directory is `/opt/mattermost`, flag this to the customer. While changing `$HOME` mid-deployment is out of scope for this runbook, at minimum the `MMENVIRONMENT` file must be placed outside `/opt/mattermost` as described in the next check.

---

### 1.2 Verify the `MMENVIRONMENT` file location is upgrade-safe

Check where the `MMENVIRONMENT` file currently lives:

```bash
sudo systemctl cat mattermost | grep EnvironmentFile
```

**Expected:** The path should point to a location **outside** `/opt/mattermost/`, for example:

```
EnvironmentFile=/etc/mattermost/config/mattermost.environment
```

**Why `/etc/mattermost/config/` is the right place:**

- `/etc/` is the canonical location for system-wide configuration on Linux and is never touched by application upgrades.
- `/etc/mattermost/` namespaces the config cleanly to the application.
- `/etc/mattermost/config/` mirrors the familiar `config/` convention, making it intuitive for Mattermost admins.
- It provides a dedicated directory that can grow to hold additional config-related files in the future.

If the `EnvironmentFile` path is inside `/opt/mattermost/` (e.g. `/opt/mattermost/config/mattermost.environment`), the file is at risk during upgrades. See [Section 1.6](#16-remediation-steps-if-checks-fail) for how to move it safely.

---

### 1.3 Verify `config.json` is migrated to the database

Run the following on **each** app server:

```bash
# Check which config store Mattermost is using at runtime
sudo systemctl show mattermost -p Environment
```

Look for a `MM_CONFIG` environment variable pointing to a `postgres://` DSN, for example:

```
MM_CONFIG=postgres://mmuser:mmpassword@db-primary.internal:5432/mattermost?sslmode=disable
```

If `MM_CONFIG` is set to a file path (e.g. `/opt/mattermost/config/config.json`) or is absent, the config has **not** been migrated to the database. See [Section 1.6](#16-remediation-steps-if-checks-fail).

You can also verify from the Mattermost side by checking the System Console → **Environment → Configuration** and confirming it shows **"Database"** as the config store, or by querying the database directly:

```sql
-- Run on the primary PostgreSQL node
SELECT COUNT(*) FROM configurations;
```

A non-zero result confirms at least one configuration has been stored in the database.

---

### 1.4 Verify the database DSN is bootstrapped in systemd

The recommended approach is to use an `MMENVIRONMENT` file (cleaner when multiple environment variables are needed) rather than inlining variables in the unit file itself.

#### Expected: Unit file references an EnvironmentFile

Check the active unit file on each app server:

```bash
sudo systemctl cat mattermost
```

**Expected output (using `MMENVIRONMENT` file — preferred):**

```ini
[Unit]
Description=Mattermost
After=network.target postgresql.service

[Service]
Type=notify
User=mattermost
Group=mattermost
WorkingDirectory=/opt/mattermost
EnvironmentFile=/etc/mattermost/config/mattermost.environment
ExecStart=/opt/mattermost/bin/mattermost
Restart=on-failure
RestartSec=10
LimitNOFILE=49152

[Install]
WantedBy=multi-user.target
```

The `MMENVIRONMENT` file (`/opt/mattermost/config/mattermost.environment`) should contain at minimum:

```bash
MM_CONFIG=postgres://mmuser:mmpassword@db-primary.internal:5432/mattermost?sslmode=disable
```

**Alternative (acceptable but less clean): DSN inlined directly in the unit file:**

```ini
[Service]
Environment="MM_CONFIG=postgres://mmuser:mmpassword@db-primary.internal:5432/mattermost?sslmode=disable"
```

If **neither** approach is in place, see [Section 1.6](#16-remediation-steps-if-checks-fail).

---

### 1.5 Verify the local `config.json` is no longer active

When the DB config store is active, the local `config.json` should be renamed or removed so it cannot interfere.

```bash
# Check whether a local config.json still exists
ls -lh /opt/mattermost/config/config.json
```

**Expected:** The file does not exist, or has been renamed (e.g. `config.json.bak`).

If the file is still present and named `config.json`, Mattermost may pick it up depending on how the binary resolves its config path. See [Section 1.4](#14-remediation-steps-if-checks-fail).

---

### 1.6 Remediation steps if checks fail

Only follow these steps if one or more checks above did not pass.

#### Step A — Migrate `config.json` to the database

If the config has not yet been migrated, use the Mattermost CLI to push the local config into the database. Run this on **one** app server only (the primary):

```bash
sudo -u mattermost /opt/mattermost/bin/mattermost config migrate \
  /opt/mattermost/config/config.json \
  "postgres://mmuser:mmpassword@db-primary.internal:5432/mattermost?sslmode=disable"
```

Verify success:

```bash
sudo -u mattermost /opt/mattermost/bin/mattermost config get ServiceSettings.SiteURL \
  --config "postgres://mmuser:mmpassword@db-primary.internal:5432/mattermost?sslmode=disable"
```

This should return your configured site URL, confirming the DB config store is readable.

#### Step B — Configure the `MMENVIRONMENT` file and update the systemd unit

The `MMENVIRONMENT` file must live **outside** `/opt/mattermost/` to survive binary upgrades. The canonical location is `/etc/mattermost/config/mattermost.environment`.

If an `MMENVIRONMENT` file already exists inside `/opt/mattermost/`, move it first before updating the systemd unit:

```bash
# Only run this if the file currently lives inside /opt/mattermost/
sudo mkdir -p /etc/mattermost/config
sudo mv /opt/mattermost/config/mattermost.environment /etc/mattermost/config/mattermost.environment
```

Create or populate the environment file on **each** app server:

```bash
sudo mkdir -p /etc/mattermost/config

sudo tee /etc/mattermost/config/mattermost.environment > /dev/null <<'EOF'
MM_CONFIG=postgres://mmuser:mmpassword@db-primary.internal:5432/mattermost?sslmode=disable
EOF

sudo chmod 640 /etc/mattermost/config/mattermost.environment
sudo chown mattermost:mattermost /etc/mattermost/config/mattermost.environment
```

Edit the systemd unit file to reference the new path. Create a drop-in override (preferred, avoids editing the packaged unit):

```bash
sudo systemctl edit mattermost
```

Add the following in the editor:

```ini
[Service]
EnvironmentFile=/etc/mattermost/config/mattermost.environment
```

Reload systemd to pick up the change:

```bash
sudo systemctl daemon-reload
```

Do this on **both** app servers before restarting either service.

#### Step C — Rename or remove the local `config.json`

On **each** app server:

```bash
sudo mv /opt/mattermost/config/config.json /opt/mattermost/config/config.json.bak
```

> **Note:** Keep the backup until you have confirmed everything is working correctly after the full procedure.

---

## 2. Enable Read Replicas in `config.json`

With the DB config store confirmed active, use the Mattermost CLI or the System Console to add the replica DSN.

> **Important:** All changes to the database-backed config should be made via the CLI or System Console — **do not** edit the `config.json` backup file on disk and attempt to re-migrate it.

### Using the Mattermost CLI (recommended for HA environments)

Run on **one** app server. The change is written to the database and picked up by both nodes.

```bash
# Set the read replica (for general DB reads)
sudo -u mattermost /opt/mattermost/bin/mattermost config set \
  SqlSettings.DataSourceReplicas \
  '["postgres://mmuser:mmpassword@db-secondary.internal:5432/mattermost?sslmode=disable"]'

# Set the search replica (for search queries — can point to the same secondary)
sudo -u mattermost /opt/mattermost/bin/mattermost config set \
  SqlSettings.DataSourceSearchReplicas \
  '["postgres://mmuser:mmpassword@db-secondary.internal:5432/mattermost?sslmode=disable"]'
```

### Verify the resulting SqlSettings block

After setting the values, confirm they are stored correctly:

```bash
sudo -u mattermost /opt/mattermost/bin/mattermost config get SqlSettings
```

The relevant portion of the config should look like this:

```json
"SqlSettings": {
    "DriverName": "postgres",
    "DataSource": "postgres://mmuser:mmpassword@db-primary.internal:5432/mattermost?sslmode=disable",
    "DataSourceReplicas": [
        "postgres://mmuser:mmpassword@db-secondary.internal:5432/mattermost?sslmode=disable"
    ],
    "DataSourceSearchReplicas": [
        "postgres://mmuser:mmpassword@db-secondary.internal:5432/mattermost?sslmode=disable"
    ],
    "MaxIdleConns": 20,
    "ConnMaxLifetimeMilliseconds": 3600000,
    "MaxOpenConns": 300,
    "Trace": false,
    "AtRestEncryptKey": "",
    "QueryTimeout": 30,
    "MigrationsStatementTimeoutSeconds": 100000
}
```

> **Tip:** `DataSourceReplicas` and `DataSourceSearchReplicas` are both arrays. If you have multiple read replicas, add them as additional entries in the array.

---

## 3. Final Checks

### 3.1 Restart Mattermost and verify service health

Restart the service on **both** app servers. In an HA setup, do one node at a time to avoid downtime.

```bash
# Node 1
sudo systemctl restart mattermost
sudo systemctl status mattermost
```

Wait until Node 1 reports `active (running)` before restarting Node 2:

```bash
# Node 2
sudo systemctl restart mattermost
sudo systemctl status mattermost
```

Also verify the Mattermost health endpoint responds on each node:

```bash
curl -sf http://localhost:8065/api/v4/system/ping | python3 -m json.tool
```

Expected response:

```json
{
    "status": "OK",
    "database_status": "OK",
    "filestore_status": "OK"
}
```

---

### 3.2 Verify read replica usage in Mattermost logs

Check the Mattermost application log for confirmation that replicas are being connected to on startup:

```bash
sudo journalctl -u mattermost --since "5 minutes ago" | grep -i replica
```

Or if logging to file:

```bash
sudo grep -i replica /opt/mattermost/logs/mattermost.log | tail -20
```

On a successful start with replicas configured, you should see log entries indicating Mattermost has established connections to the replica DSN(s), for example:

```
{"level":"info","msg":"Pinging SQL replica","driver":"postgres","dataSource":"postgres://mmuser:***@db-secondary.internal:5432/mattermost"}
```

If you see connection errors pointing to the secondary, verify network access and PostgreSQL authentication from the app servers to the secondary node.

---

### 3.3 Verify query routing on the database side

Connect to your **secondary** PostgreSQL node and monitor active connections coming from Mattermost:

```sql
-- Run on db-secondary.internal
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    state,
    query,
    now() - query_start AS query_duration
FROM pg_stat_activity
WHERE usename = 'mmuser'
  AND state = 'active'
ORDER BY query_start DESC
LIMIT 20;
```

Trigger some read activity in Mattermost (e.g. load a channel, perform a search) and confirm that `SELECT` queries from `mmuser` appear on the secondary. Write queries (`INSERT`, `UPDATE`, `DELETE`) should continue to appear only on the **primary** node.

You can also check the total connection count to the secondary to confirm Mattermost has established its connection pool:

```sql
-- Run on db-secondary.internal
SELECT COUNT(*) AS mattermost_connections
FROM pg_stat_activity
WHERE usename = 'mmuser';
```

This count should reflect the `MaxIdleConns` / `MaxOpenConns` values configured in `SqlSettings`.

---

## Summary

| Step | What was done |
|---|---|
| Preliminary checks | Confirmed process user and home directory are safe, `MMENVIRONMENT` at `/etc/mattermost/config/`, DB config store active, systemd bootstrapped, local `config.json` removed |
| Read replica config | Set `DataSourceReplicas` and `DataSourceSearchReplicas` pointing to `db-secondary.internal` |
| Final checks | Service restarted cleanly, health endpoint OK, replica connections visible in logs and `pg_stat_activity` |