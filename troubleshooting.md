# Troubleshooting Matrix

[← Back to Home](index.html) • [Deployment Guide](deployment.html) • [Configuration Reference](configuration.html) • [API Reference](api.html)

---

## HTTP Status Codes & Solutions

| Status | Cause | Resolution |
| :--- | :--- | :--- |
| **`400 Hook rules were not satisfied.`** | Trigger rule mismatch (e.g. invalid HMAC secret, missing header, payload branch mismatch). | Run server with `--verbose` to inspect trigger evaluation logs. Verify HMAC secret matches provider. |
| **`405 Method Not Allowed`** | Webhook sent with an HTTP method not listed in hook's `http-methods`. | Update hook's `http-methods` array or send the request using `POST`. |
| **`413 Payload Too Large`** | Request payload exceeds `server.max_body_bytes`. | Increase `server.max_body_bytes` in `config.toml` (default is 10 MB). |
| **`429 Queue capacity full`** | In-flight queued jobs have reached `queue.max_pending_jobs`. | Increase worker concurrency (`worker.concurrency`), add dedicated worker nodes, or increase `queue.max_pending_jobs`. |
| **`429 Rate limit exceeded`** | Exceeded 60 requests/minute on administrative endpoints. | Wait for the duration in `Retry-After: 60` or throttle management scripts. |
| **`500 Internal Server Error`** | Synchronous command returned non-zero exit code or rule evaluation threw an unhandled error. | Inspect stdout/stderr in response (if `include-command-output-in-response-on-error: true`) or inspect server logs. |
| **`503 Service Unavailable` (On Hook Post)** | Storage backend is unreachable when accepting an asynchronous webhook. | Check SQLite filesystem permissions or verify Valkey/Redis network connectivity. Webhook was **not** acknowledged. |
| **`503 Admin token not configured`** | Calling `/api/v1/*` without configuring `admin_token_env` or `admin_token_file`. | Set `WEBHOOK_ADMIN_TOKEN` in the environment or set `admin_token_file` in `config.toml`. |

---

## Worker & Execution Issues

### Job Transitions Immediately to `dead_letter`
- **Cause**: Executable path or working directory does not exist or lacks execute permissions.
- **Why**: `webhook-async` classifies missing binaries as non-transient `Configuration` errors and dead-letters immediately to prevent infinite retry loops.
- **Solution**: Use absolute executable paths (e.g. `/usr/local/bin/deploy.sh`) and ensure the `webhook` user has read and execute permissions.

### Worker Heartbeat / Lease Expiry
- **Symptom**: Job stays in `running` and is reclaimed by another worker after 60s.
- **Cause**: Task process completely locked up the CPU or network, preventing the asynchronous heartbeat loop from renewing the lease.
- **Solution**: Ensure system resources are sufficient, or increase `worker.lease_duration_seconds` (e.g. to 120s).

### `database is locked` on SQLite
- **Cause**: High concurrency without Write-Ahead Logging (WAL) or missing busy timeout.
- **Solution**: Verify that `wal_mode = true` and `busy_timeout_ms = 5000` are configured in `[storage.sqlite]`.

---

## Diagnostic Commands

### Validate Configuration Files
```bash
webhook-async config validate --config config.toml --hooks hooks.json
```

### Check Storage Health
```bash
webhook-async storage status --config config.toml
```

### Test Webhook Triggers Locally
```bash
curl -v -X POST http://localhost:9000/hooks/test-hook \
  -H "Content-Type: application/json" \
  -d '{"test": true}'
```
