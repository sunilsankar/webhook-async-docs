# REST API & Endpoint Architecture Reference

[← Back to Home](index.md) • [Deployment Guide](deployment.md) • [Configuration Reference](configuration.md) • [Operations](operations.md)

---

## 1. How Webhook Processing Works: The Complete Pipeline

When an HTTP request hits `webhook-async`, it passes through a deterministic 10-stage execution pipeline before any command is executed or enqueued:

```
                  ┌─────────────────────────────────────────┐
                  │ 1. Match Route & Method                 │
                  │    - Path: /{url_prefix}/{hook_id}      │
                  │    - Check hook.http-methods (405)      │
                  └────────────────────┬────────────────────┘
                                       │
                  ┌────────────────────▼────────────────────┐
                  │ 2. Enforce Size & Content-Type Limits   │
                  │    - Body <= max_body_bytes (413)       │
                  │    - Parse Multipart parts (500)        │
                  └────────────────────┬────────────────────┘
                                       │
                  ┌────────────────────▼────────────────────┐
                  │ 3. Extract & Parse Payload Data         │
                  │    - Preserve exact raw body bytes      │
                  │    - Parse JSON, XML, or Form data      │
                  │    - Execute parse-parameters-as-json   │
                  └────────────────────┬────────────────────┘
                                       │
                  ┌────────────────────▼────────────────────┐
                  │ 4. Evaluate Security & Trigger Rules    │
                  │    - Check HMAC signatures (constant-   │
                  │      time equality)                     │
                  │    - Check tokens, IP whitelists, regex │
                  │    - Mismatch -> HTTP 400 (or custom)   │
                  └────────────────────┬────────────────────┘
                                       │
                  ┌────────────────────▼────────────────────┐
                  │ 5. Build Immutable Execution Snapshot   │
                  │    - Resolve command argv & environment │
                  │    - Materialize temporary files        │
                  │    - Bind timeout and output limits     │
                  └────────────────────┬────────────────────┘
                                       │
                   ┌───────────────────┴───────────────────┐
                   │                                       │
         [Sync Hook Branch]                       [Async Hook Branch]
                   │                                       │
      ┌────────────▼─────────────┐             ┌───────────▼────────────┐
      │ 6a. Direct Execution     │             │ 6b. Durable Commit     │
      │   - Spawn tokio::process │             │   - Commit snapshot to │
      │   - Bound stdout/stderr  │             │     SQLite / Redis     │
      │   - Clean up temp files  │             │   - Dedup Idempotency  │
      │   - Return HTTP 200/500  │             │   - Return HTTP 202    │
      └──────────────────────────┘             └────────────────────────┘
```

---

## 2. Webhook Receiver Endpoints

### 1. Root Liveness Ping
```http
GET /
```
Used by network load balancers, Kubernetes liveness probes, and reverse proxies.
- **Authentication**: None required.
- **Response**: `200 OK` with plain text `"OK"`.
- **Headers**: Includes global `response_headers` configured in `config.toml`.

---

### 2. Trigger a Webhook
```http
POST /{url_prefix}/{hook_id}
```
Executes the hook identified by `{hook_id}`.
- If `url_prefix` is configured as `"hooks"` (the default), the path is `/hooks/my-hook`.
- If `url_prefix` is empty (`""`), the path is `/my-hook`.
- Subpaths are fully supported if your hook ID contains forward slashes (e.g. `/hooks/deploy/prod/backend`).

#### Supported Request Headers
| Header | Description |
| :--- | :--- |
| `Content-Type` | `application/json`, `application/x-www-form-urlencoded`, `text/xml`, or `multipart/form-data`. |
| `Idempotency-Key` / `X-Idempotency-Key` | *(Async hooks only)* Unique request identifier. Prevents duplicate execution if a client retries due to network failure. |
| `X-Request-Id` | Distributed tracing request ID. Propagated through execution logs and responses. |

#### Exhaustive Response Status Codes
| HTTP Status | Condition | Response Body |
| :--- | :--- | :--- |
| **`200 OK`** | Synchronous hook executed successfully. | Output from stdout/stderr (if `include-command-output-in-response: true`), or the configured `response-message`. |
| **`202 Accepted`** | Asynchronous hook validated and committed to durable storage queue. | `{"job_id": "...", "status": "queued", "message": "..."}` |
| **`400 Bad Request`** | Trigger rules were not satisfied (HMAC mismatch, missing parameter, regex failure). | `"Hook rules were not satisfied."` *(or custom status code configured via `trigger-rule-mismatch-http-response-code`)* |
| **`405 Method Not Allowed`** | Incoming HTTP verb (e.g. `GET`) is not in the hook's `http-methods` list. | Empty body. |
| **`413 Payload Too Large`** | Request body size exceeds `server.max_body_bytes`. | `"Request body too large."` |
| **`429 Too Many Requests`** | Async queue capacity full (`queue.max_pending_jobs` reached). | `{"error": "Queue capacity full."}` |
| **`500 Internal Server Error`** | Trigger rule evaluation failure, multipart parsing failure, or command exited with non-zero status. | `"Error occurred while evaluating hook rules."` or command stderr output. |
| **`503 Service Unavailable`** | In async mode, storage backend is unreachable. The hook was **not** acknowledged and must be retried by the caller. | `{"error": "Queue backend unavailable."}` |

---

## 3. System Diagnostic & OpenAPI Endpoints

These endpoints provide health monitoring and schema inspection without requiring authentication.

### 1. Basic Health Check
```http
GET /api/v1/health
```
Reports service availability and compiled application version.

```bash
curl -s http://localhost:9000/api/v1/health
```
```json
{
  "status": "ok",
  "version": "0.1.0"
}
```

---

### 2. Storage Readiness Probe
```http
GET /api/v1/ready
```
Performs an active ping against the configured storage backend (e.g. `SELECT 1` on SQLite or `PING` on Valkey/Redis).

```bash
curl -s http://localhost:9000/api/v1/ready
```
```json
{
  "status": "ready",
  "storage": "connected"
}
```
*If storage is disconnected, returns `503 Service Unavailable` with `{"status": "not_ready", "error": "..."}`.*

---

### 3. OpenAPI 3.1 Specification
```http
GET /api/v1/openapi.json
```
Returns the complete machine-readable OpenAPI 3.1 specification describing all REST endpoints, request schemas, response models, and authentication requirements.

```bash
curl -s http://localhost:9000/api/v1/openapi.json | jq .
```

---

## 4. Administrative REST API (`/api/v1/*`)

All administrative endpoints require Bearer token authentication configured via `[auth].admin_token`, `[auth].admin_token_env`, or `[auth].admin_token_file`.

```http
Authorization: Bearer <YOUR_ADMIN_TOKEN>
```

> **Security Guardrails**:
> - If no admin token is configured in `config.toml`, all administrative endpoints return **`503 Service Unavailable`** ("Admin token not configured"). Unauthenticated administrative access is strictly disallowed by default.
> - Rate limited to 60 requests/minute per token. Exceeding limits returns **`429 Too Many Requests`** with `Retry-After: 60`.

---

### 1. Get Queue Statistics
```http
GET /api/v1/stats
```
Returns real-time job counts partitioned across all 8 canonical job states.

```bash
curl -s -H "Authorization: Bearer $WEBHOOK_ADMIN_TOKEN" \
  http://localhost:9000/api/v1/stats
```

```json
{
  "counts_by_state": {
    "queued": 0,
    "running": 1,
    "retry_wait": 0,
    "succeeded": 412,
    "failed": 2,
    "cancel_requested": 0,
    "cancelled": 1,
    "dead_letter": 0
  },
  "pending_count": 1
}
```

---

### 2. List & Filter Jobs
```http
GET /api/v1/jobs?state={state}&hook_id={id}&limit={n}&offset={n}
```

#### Query Parameters
| Parameter | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `state` | string | *(optional)* | Filter by state: `queued`, `running`, `retry_wait`, `succeeded`, `failed`, `cancel_requested`, `cancelled`, `dead_letter`. |
| `hook_id` | string | *(optional)* | Filter by hook identifier. |
| `limit` | integer | `50` | Number of records to return (1 to 500). |
| `offset` | integer | `0` | Pagination offset. |

```bash
curl -s -H "Authorization: Bearer $WEBHOOK_ADMIN_TOKEN" \
  "http://localhost:9000/api/v1/jobs?state=failed&limit=10"
```

```json
{
  "items": [
    {
      "id": "018d9f10-1a2b-7c3d-8e4f-5a6b7c8d9e0f",
      "hook_id": "deploy-app",
      "state": "failed",
      "priority": 0,
      "attempt": 5,
      "max_attempts": 5,
      "created_at": "2026-03-29T10:15:30.123456789Z",
      "started_at": "2026-03-29T10:15:31.000000000Z",
      "completed_at": "2026-03-29T10:16:05.000000000Z",
      "worker_id": "worker-prod-1:4192",
      "exit_code": 1,
      "error_class": "transient",
      "error_summary": "Process exited with status: exit status: 1"
    }
  ],
  "total": 1,
  "limit": 10,
  "offset": 0
}
```

---

### 3. Get Job Details & Execution History
```http
GET /api/v1/jobs/{job_id}
```
Returns complete metadata, immutable execution snapshot parameters, and the full audit trail of all execution attempts (including captured stdout and stderr previews).

```bash
curl -s -H "Authorization: Bearer $WEBHOOK_ADMIN_TOKEN" \
  http://localhost:9000/api/v1/jobs/018d9f10-1a2b-7c3d-8e4f-5a6b7c8d9e0f
```

```json
{
  "job": {
    "id": "018d9f10-1a2b-7c3d-8e4f-5a6b7c8d9e0f",
    "hook_id": "deploy-app",
    "state": "succeeded",
    "priority": 0,
    "attempt": 1,
    "max_attempts": 5,
    "created_at": "2026-03-29T10:15:30.123456789Z",
    "started_at": "2026-03-29T10:15:31.000000000Z",
    "completed_at": "2026-03-29T10:15:45.000000000Z",
    "worker_id": "worker-prod-1:4192",
    "exit_code": 0
  },
  "attempts": [
    {
      "job_id": "018d9f10-1a2b-7c3d-8e4f-5a6b7c8d9e0f",
      "attempt": 1,
      "worker_id": "worker-prod-1:4192",
      "started_at": "2026-03-29T10:15:31.000000000Z",
      "finished_at": "2026-03-29T10:15:45.000000000Z",
      "exit_code": 0,
      "stdout_preview": "Deploying release v1.4.2...\nMigration complete.\nService restarted.",
      "stderr_preview": ""
    }
  ]
}
```

---

### 4. Retry a Job
```http
POST /api/v1/jobs/{job_id}/retry
```
Manually re-enqueues a job that ended in `failed` or `dead_letter` state.
- Resets the job to `queued` state.
- Automatically increments `max_attempts` so the job is eligible for immediate execution.
- Returns `409 Conflict` if the job is already active (`running` or `queued`).

```bash
curl -X POST -s -H "Authorization: Bearer $WEBHOOK_ADMIN_TOKEN" \
  http://localhost:9000/api/v1/jobs/018d9f10-1a2b-7c3d-8e4f-5a6b7c8d9e0f/retry
```

---

### 5. Cancel a Job
```http
POST /api/v1/jobs/{job_id}/cancel
```
Cancels a job:
- If `queued` or `retry_wait`: Transitions immediately to `cancelled`.
- If `running`: Transitions to `cancel_requested`. The active worker's heartbeat loop detects this state within milliseconds, sends `SIGKILL` to the child process group (`-pgid`), cleans up temporary files, and marks the job as `cancelled`.

```bash
curl -X POST -s -H "Authorization: Bearer $WEBHOOK_ADMIN_TOKEN" \
  http://localhost:9000/api/v1/jobs/018d9f10-1a2b-7c3d-8e4f-5a6b7c8d9e0f/cancel
```

---

### 6. Inspect Active Hooks (Secrets Redacted)
```http
GET /api/v1/hooks
```
Returns a list of all loaded hook definitions. Sensitive parameters (HMAC keys, bearer tokens) are safely redacted.

```bash
curl -s -H "Authorization: Bearer $WEBHOOK_ADMIN_TOKEN" \
  http://localhost:9000/api/v1/hooks
```

---

### 7. Inspect Active Workers
```http
GET /api/v1/workers
```
Returns all background workers currently heartbeating into the cluster.

```bash
curl -s -H "Authorization: Bearer $WEBHOOK_ADMIN_TOKEN" \
  http://localhost:9000/api/v1/workers
```

```json
[
  {
    "worker_id": "worker-prod-1:4192",
    "host": "ip-10-0-1-15.internal",
    "concurrency": 8,
    "active_jobs": 2,
    "completed_count": 894,
    "failed_count": 3,
    "draining": false,
    "last_heartbeat_at": "2026-03-29T10:20:00.000000000Z"
  }
]
```

---

### 8. Storage Engine Status & Capabilities
```http
GET /api/v1/storage/status
```
Returns backend engine type, multi-host capabilities, and connection status.

```bash
curl -s -H "Authorization: Bearer $WEBHOOK_ADMIN_TOKEN" \
  http://localhost:9000/api/v1/storage/status
```

```json
{
  "backend": "sqlite",
  "connected": true,
  "capabilities": {
    "backend": "sqlite",
    "supports_priority": true,
    "supports_multi_host": false,
    "supports_idempotency": true
  }
}
```

---

### 9. Real-Time Server-Sent Events (SSE) Stream
```http
GET /api/v1/events
```
Streams live JSON snapshots of queue metrics and active jobs every 2 seconds. Used by monitoring tools and the built-in Web management UI.

```bash
curl -N -s -H "Authorization: Bearer $WEBHOOK_ADMIN_TOKEN" \
  http://localhost:9000/api/v1/events
```

```http
event: stats
data: {"counts_by_state":{"queued":0,"running":1,"succeeded":412},"pending_count":1}

event: stats
data: {"counts_by_state":{"queued":0,"running":0,"succeeded":413},"pending_count":0}
```
