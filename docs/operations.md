# Operations & Runbook

[← Back to Home](index.md) • [Deployment Guide](deployment.md) • [Configuration Reference](configuration.md) • [API Reference](api.md)

---

## 1. Process Lifecycle & Signal Handling

`webhook-async` handles standard POSIX signals for lifecycle events:

| Signal | Action | Behavior |
| :--- | :--- | :--- |
| `SIGHUP` | Reload Hooks | Re-reads and re-validates hook definition files. Current active hooks remain untouched if parsing or validation fails. |
| `SIGUSR1` | Reload Hooks | Upstream-compatible reload signal. |
| `SIGTERM` | Graceful Shutdown | Stops accepting new HTTP requests, drains workers for `shutdown_grace_seconds`, and cleanly unlinks PID/socket files. |
| `SIGINT` | Graceful Shutdown | Intercepts `Ctrl+C` for clean termination. |

### Sending Reload Signals
```bash
# Using kill with PID file
kill -HUP $(cat /var/run/webhook-async.pid)

# Using systemd
sudo systemctl reload webhook-async
```

---

## 2. Dynamic Hot Reloading

When `--hotreload` is specified (or `server.hot_reload = true` in `config.toml`), `webhook-async` polls the configured hook files for modifications every 2.5 seconds.

### Invariant Guarantees
- **Atomic Swap**: The internal routing table is updated atomically using a thread-safe read-write lock (`Arc<RwLock<...>>`).
- **Immutable Job Snapshots**: In-flight and queued jobs retain the exact `ExecutionSpec` snapshot captured when they were accepted. Reloading hooks never mutates running tasks.
- **Fail-Safe**: If a modified hook file contains invalid syntax or unknown fields, an error is logged and the server continues running with the previous valid configuration.

---

## 3. Zero-Downtime SQLite Hot Backups

Because `webhook-async` uses SQLite in `WAL` (Write-Ahead Logging) mode with multi-connection concurrency and busy timeouts, backups can be taken while the server is accepting requests without table locks.

### Online Backup Command
```bash
sqlite3 /var/lib/webhook-async/data.db "VACUUM INTO '/var/backups/webhook-async-$(date +%F-%H%M).db'"
```

### Automated Nightly Backup Cron Script
Create `/etc/cron.daily/webhook-async-backup`:
```bash
#!/bin/bash
set -euo pipefail

BACKUP_DIR="/var/backups/webhook-async"
DB_PATH="/var/lib/webhook-async/data.db"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

mkdir -p "$BACKUP_DIR"

# Perform online atomic backup
sqlite3 "$DB_PATH" "VACUUM INTO '$BACKUP_DIR/backup-$TIMESTAMP.db'"

# Retain backups for 14 days
find "$BACKUP_DIR" -type f -name "backup-*.db" -mtime +14 -delete
```
Make executable:
```bash
sudo chmod +x /etc/cron.daily/webhook-async-backup
```

### Restoring from Backup
1. Stop the service:
   ```bash
   sudo systemctl stop webhook-async
   ```
2. Replace the database:
   ```bash
   sudo cp /var/backups/webhook-async/backup-20261113_020000.db /var/lib/webhook-async/data.db
   sudo rm -f /var/lib/webhook-async/data.db-wal /var/lib/webhook-async/data.db-shm
   sudo chown webhook:webhook /var/lib/webhook-async/data.db
   ```
3. Start the service:
   ```bash
   sudo systemctl start webhook-async
   ```

---

## 4. Valkey / Redis Persistence & Backups

For distributed deployments using Valkey or Redis:

### Persistence Requirements
1. Always enable Append-Only File (AOF) in `valkey.conf` / `redis.conf`:
   ```text
   appendonly yes
   appendfsync everysec
   ```
2. Configure RDB snapshots:
   ```text
   save 900 1
   save 300 10
   save 60 10000
   ```

### Triggering Manual Backup
```bash
# Trigger background RDB snapshot
valkey-cli bgsave

# Copy the generated dump.rdb
cp /var/lib/valkey/dump.rdb /var/backups/valkey-dump-$(date +%F).rdb
```

---

## 5. Logging, Diagnostics & Crash Safety

### Structured Logging
Configure logging outputs:
- Standard output: `--verbose` or `--debug`.
- Dedicated log file: `--logfile /var/log/webhook-async/service.log`.

### Panic Immunity (`--nopanic`)
In production, pass `--nopanic` (or `-nopanic`). This installs a custom panic hook that logs the stack trace without aborting the server process, preventing individual malformed requests from causing an outage.

### Storage Health Inspection via CLI
```bash
webhook-async storage status --config /etc/webhook-async/config.toml
```
Output:
```text
Storage Backend: SQLite (Embedded)
Capabilities:
  Priority Ordering: true
  Multi-Host Shared: false
  Idempotency TTL:   true
Connectivity:       Connected
Counts:
  Queued:           0
  Running:          2
  Succeeded:        1420
  Failed:           3
  Dead Letter:      0
Total Pending:      2
```
