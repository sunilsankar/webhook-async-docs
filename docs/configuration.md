# Configuration Reference & Step-by-Step Hook Guide

[← Back to Home](index.md) • [Deployment Guide](deployment.md) • [API Reference](api.md) • [Operations](operations.md)

---

## 1. Application Configuration (`config.toml`)

`webhook-async` uses a single TOML file for server, worker, queue, storage, and security configuration.

```toml
[server]
host = "0.0.0.0"                        # Network interface to bind
port = 9000                             # Listening port
url_prefix = "hooks"                    # Route prefix (e.g. /hooks/{id})
max_body_bytes = 10485760               # Maximum request body size (10 MB)
max_multipart_mem_bytes = 1048576       # Memory buffer limit for multipart parts (1 MB)
hook_files = ["/etc/webhook/hooks.json"]# Paths to hook definition files (JSON or YAML)
hot_reload = true                       # Automatically reload hook files on change
default_http_methods = ["POST"]         # Fallback methods when hook omits http-methods
response_headers = [                    # Global response headers added to all replies
  { name = "X-Service", value = "webhook-async" },
  { name = "Access-Control-Allow-Origin", value = "*" }
]
use_x_request_id = true                 # Propagate or generate X-Request-Id header
x_request_id_limit = 64                 # Maximum character length for X-Request-Id
trust_proxy_headers = true              # Honor X-Forwarded-For and X-Real-IP for remote-addr

[storage]
backend = "sqlite"                      # Storage engine: "sqlite", "valkey", or "redis"

[storage.sqlite]
database_url = "sqlite:///var/lib/webhook-async/data.db?mode=rwc"
wal_mode = true                         # Enable Write-Ahead Logging (WAL) for concurrency
busy_timeout_ms = 5000                  # Maximum wait time on busy database locks

# Valkey / Redis configuration (uncomment when backend is "valkey" or "redis"):
# [storage.valkey]
# url = "redis://10.0.1.50:6379/0"      # Accepts redis://, rediss://, valkey://, valkeys://
# key_prefix = "webhook_async:"         # Namespaced key prefix (uses hash-tags for cluster)

[worker]
concurrency = 8                         # Maximum concurrent background tasks
poll_interval_ms = 250                  # Delay between queue poll attempts when idle
lease_duration_seconds = 60             # Lease window granted to worker for each claim
heartbeat_interval_seconds = 15         # Frequency at which active worker extends lease
shutdown_grace_seconds = 30             # Maximum drain duration on shutdown signal

[queue]
max_pending_jobs = 10000                # Maximum queued jobs (returns HTTP 429 when full)
default_max_attempts = 5                # Maximum execution attempts before dead-lettering
retry_base_delay_seconds = 5            # Exponential backoff base delay
retry_max_delay_seconds = 300           # Exponential backoff ceiling delay
default_timeout_seconds = 300           # Maximum execution time per command attempt
max_output_bytes = 65536                # Maximum stdout/stderr output captured (64 KB)
idempotency_retention_seconds = 86400   # Idempotency deduplication TTL (24 hours)

[ui]
enabled = true                          # Enable built-in Web management UI
bind = "127.0.0.1:9001"                 # Dedicated bind for the Web UI
path_prefix = ""                        # Optional URL subpath prefix (e.g. "/admin")

[auth]
admin_token = "password@123"                # Direct literal password in TOML
# Or load from an environment variable (or file path):
# admin_token_env = "WEBHOOK_ADMIN_TOKEN"
# Or load from a dedicated secret file:
# admin_token_file = "/run/secrets/admin_token"
```

---

## 2. Step-by-Step Guide: How to Configure `hooks.json`

A `hooks.json` file is a JSON array of hook definition objects. Below is a progressive, step-by-step tutorial on building production-ready hooks.

### Step 1: The Absolute Minimum Hook (5 Lines)
At its core, a hook only requires an `id` and an `execute-command`.

Create a file named `hooks.json`:
```json
[
  {
    "id": "hello",
    "execute-command": "/opt/scripts/hello.sh"
  }
]
```

When you start the server (`webhook-async server --hooks hooks.json`), calling:
```bash
curl -X POST http://localhost:9000/hooks/hello
```
will execute `/opt/scripts/hello.sh` and return `200 OK`.

---

### Step 2: Set Working Directory and Restrict HTTP Methods
In production, scripts should execute from an explicit directory and only allow intended HTTP verbs (typically `POST`).

```json
[
  {
    "id": "deploy-app",
    "execute-command": "/opt/app/scripts/deploy.sh",
    "command-working-directory": "/opt/app",
    "http-methods": ["POST"]
  }
]
```
If an unauthorized method (like `GET`) is sent, the server immediately returns `405 Method Not Allowed` with an empty body and does not execute the command.

---

### Step 3: Customize HTTP Responses and Output Capture
By default, the server returns an empty body on success. You can customize the response message or return stdout/stderr from the command directly to the caller.

```json
[
  {
    "id": "ping-hook",
    "execute-command": "/opt/scripts/check-health.sh",
    "command-working-directory": "/opt/scripts",
    "http-methods": ["POST"],
    "response-message": "Health check started successfully.",
    "include-command-output-in-response": true,
    "include-command-output-in-response-on-error": true,
    "response-headers": [
      { "name": "Content-Type", "value": "text/plain" },
      { "name": "X-Executed-By", "value": "webhook-async" }
    ]
  }
]
```

- `include-command-output-in-response: true`: The server waits for the command to finish and returns the combined stdout and stderr as the HTTP response body.
- `include-command-output-in-response-on-error: true`: If the command exits with a non-zero exit code, the HTTP response status becomes `500 Internal Server Error` and includes the error output.

---

### Step 4: Pass Dynamic Arguments to Your Command
Webhooks frequently receive payload data (commit IDs, branches, usernames) that must be passed as CLI arguments to your script.

Use `pass-arguments-to-command`:
```json
[
  {
    "id": "deploy-branch",
    "execute-command": "/opt/scripts/deploy.sh",
    "command-working-directory": "/opt/scripts",
    "http-methods": ["POST"],
    "pass-arguments-to-command": [
      {
        "source": "payload",
        "name": "ref"
      },
      {
        "source": "payload",
        "name": "head_commit.id"
      },
      {
        "source": "url",
        "name": "environment"
      },
      {
        "source": "request",
        "name": "remote-addr"
      }
    ]
  }
]
```

When a request arrives:
```bash
curl -X POST "http://localhost:9000/hooks/deploy-branch?environment=production" \
  -H "Content-Type: application/json" \
  -d '{"ref": "refs/heads/main", "head_commit": {"id": "c759720"}}'
```
`webhook-async` invokes:
```bash
/opt/scripts/deploy.sh "refs/heads/main" "c759720" "production" "127.0.0.1"
```

> **Security Note**: `webhook-async` executes commands via direct process invocation (`execve`). Arguments are passed directly in the argv array and are **never** interpolated into a shell string, eliminating shell injection vulnerabilities.

---

### Step 5: Pass Values via Environment Variables
Passing sensitive information or large payloads through environment variables is often cleaner than positional CLI arguments.

Use `pass-environment-to-command`:
```json
[
  {
    "id": "notify-slack",
    "execute-command": "/opt/scripts/notify.sh",
    "pass-environment-to-command": [
      {
        "source": "payload",
        "name": "pusher.name",
        "envname": "DEPLOY_AUTHOR"
      },
      {
        "source": "header",
        "name": "X-GitHub-Delivery",
        "envname": "DELIVERY_ID"
      },
      {
        "source": "payload",
        "name": "repository.name"
      }
    ]
  }
]
```

Inside `/opt/scripts/notify.sh`, you can read:
```bash
echo "Author: $DEPLOY_AUTHOR"
echo "Delivery ID: $DELIVERY_ID"
echo "Repository: $HOOK_repository_name"   # Defaults to HOOK_<name> when envname is omitted
```

---

### Step 6: Write Payloads to Temporary Files
When tools (like `kubectl`, `ansible`, or `terraform`) require a file input, use `pass-file-to-command`. `webhook-async` creates a temporary file before execution, passes its absolute path in an environment variable, and automatically deletes it when execution finishes.

```json
[
  {
    "id": "import-config",
    "execute-command": "/opt/scripts/apply-config.sh",
    "command-working-directory": "/opt/scripts",
    "pass-file-to-command": [
      {
        "source": "entire-payload",
        "envname": "CONFIG_FILE"
      },
      {
        "source": "payload",
        "name": "certificates.ssl_crt",
        "envname": "CERT_PATH",
        "base64decode": true
      }
    ]
  }
]
```
Inside `/opt/scripts/apply-config.sh`:
```bash
echo "Applying config from: $CONFIG_FILE"
kubectl apply -f "$CONFIG_FILE"
```

---

### Step 7: Secure the Hook with Trigger Rules (`trigger-rule`)
Never expose unauthenticated webhooks to the public internet. Use `trigger-rule` to enforce signature verification, tokens, IP whitelists, or branch filters.

#### Scenario A: GitHub / GitLab HMAC-SHA256 Signature Verification
```json
[
  {
    "id": "github-deploy",
    "execute-command": "/opt/scripts/deploy.sh",
    "http-methods": ["POST"],
    "trigger-rule": {
      "and": [
        {
          "match": {
            "type": "payload-hmac-sha256",
            "secret": "{{ getenv \"GITHUB_WEBHOOK_SECRET\" }}",
            "parameter": {
              "source": "header",
              "name": "X-Hub-Signature-256"
            }
          }
        },
        {
          "match": {
            "type": "value",
            "value": "refs/heads/main",
            "parameter": {
              "source": "payload",
              "name": "ref"
            }
          }
        }
      ]
    }
  }
]
```

#### Scenario B: Static Shared Secret / Bearer Token
```json
[
  {
    "id": "internal-backup",
    "execute-command": "/opt/scripts/run-backup.sh",
    "trigger-rule": {
      "match": {
        "type": "value",
        "value": "{{ getenv \"BACKUP_SECRET_TOKEN\" }}",
        "parameter": {
          "source": "header",
          "name": "X-Webhook-Token"
        }
      }
    }
  }
]
```

#### Scenario C: Restricting Client IPs (CIDR Whitelist)
```json
[
  {
    "id": "office-only-hook",
    "execute-command": "/opt/scripts/restart.sh",
    "trigger-rule": {
      "match": {
        "type": "ip-whitelist",
        "ip-whitelist": "192.168.1.0/24 10.0.0.50 2001:db8::/32"
      }
    }
  }
]
```

---

### Step 8: Choose Synchronous vs. Durable Asynchronous Execution
By default, `webhook-async` operates in legacy synchronous mode (the HTTP request waits until the command finishes). 

For long-running tasks (CI builds, deployments, backups), enable **asynchronous mode**:

```json
[
  {
    "id": "async-ci-build",
    "execute-command": "/opt/scripts/build-and-test.sh",
    "command-working-directory": "/opt/builds",
    "async": true,
    "response-message": "Job enqueued for background processing."
  }
]
```

#### What happens when `async: true`:
1. The server validates the trigger rules and signature.
2. The server serializes the command and snapshot into the durable storage queue (SQLite, Valkey, or Redis).
3. The server immediately returns **`HTTP 202 Accepted`** with a tracking JSON payload:
   ```json
   {
     "job_id": "018d9f10-1a2b-7c3d-8e4f-5a6b7c8d9e0f",
     "status": "queued",
     "message": "Job enqueued for background processing."
   }
   ```
4. Background workers claim the job, execute it, heartbeat lease extensions, and handle automatic exponential backoff retries if transient errors occur.

---

### Step 9: Use Template Expressions (`-template`)
To avoid hardcoding sensitive secrets or environment-dependent paths directly inside `hooks.json`, start `webhook-async` with the `-template` flag.

Supported template functions:
- `{{ getenv "VAR_NAME" }}`: Reads environment variable `VAR_NAME` (returns empty string if unset).
- `{{ getenv "VAR_NAME" "default_value" }}`: Reads environment variable with fallback default.
- `{{ cat "/path/to/file" }}`: Reads file content, trimming trailing newlines.
- `{{ credential "SECRET_NAME" }}`: Reads `/run/credentials/$SECRET_NAME` (systemd credential directory), with fallback to environment variable.
- `{{ value | js }}`: Escapes string safely for JSON string embedding.

Example:
```json
[
  {
    "id": "secure-hook",
    "execute-command": "/opt/scripts/run.sh",
    "trigger-rule": {
      "match": {
        "type": "payload-hmac-sha256",
        "secret": "{{ cat \"/etc/secrets/webhook_hmac.key\" }}",
        "parameter": {
          "source": "header",
          "name": "X-Signature-256"
        }
      }
    }
  }
]
```

---

### Step 10: Validate Your Configuration
Before deploying to production or reloading the service, validate your files with the built-in validator:

```bash
# Validate standard JSON/YAML hooks
webhook-async config validate --hooks /etc/webhook/hooks.json

# Validate templated hooks
webhook-async config validate --template --hooks /etc/webhook/hooks.json
```
If errors are present (invalid rule depth, missing required fields, syntax typos), the command prints line-specific diagnostics with non-zero exit code.

---

## 3. Parameter Extraction Reference (`source`)

Parameters can be extracted from any part of the HTTP request and mapped into command arguments, environment variables, temporary files, or trigger rule checks.

| Source | Description | Example Specification |
| :--- | :--- | :--- |
| `header` | Canonical MIME HTTP request header. Case-insensitive header lookup. | `{"source": "header", "name": "X-GitHub-Event"}` |
| `url` / `query` | URL query parameter string. | `{"source": "url", "name": "environment"}` |
| `payload` | Body value from JSON, XML, or Form payload. Supports dot notation (`a.b.c`) and array indices (`items.0.id`). | `{"source": "payload", "name": "repository.name"}` |
| `raw-request-body` | The unparsed raw UTF-8 request body string. | `{"source": "raw-request-body"}` |
| `request` | Request metadata: `remote-addr` (client IP) or `method` (HTTP verb). | `{"source": "request", "name": "remote-addr"}` |
| `string` | Static literal string value. | `{"source": "string", "name": "production"}` |
| `entire-payload` | The full raw body written to a temporary file (`pass-file-to-command`). | `{"source": "entire-payload", "envname": "BODY_FILE"}` |
| `entire-headers` | JSON map of all HTTP request headers. | `{"source": "entire-headers", "envname": "HEADERS_JSON"}` |
| `entire-query` | JSON map of all URL query parameters. | `{"source": "entire-query", "envname": "QUERY_JSON"}` |

---

## 4. Trigger Rule Combinators and Matchers

Trigger rules determine whether the incoming request is authorized to execute the command.

### Combinators
- `and`: Array of rules; all child rules must evaluate to `true`.
- `or`: Array of rules; passes if **any** child rule evaluates to `true`.
- `not`: Inverts the boolean result of a child rule.

### Matchers (`match`)

| Matcher Type | Description | Required Fields |
| :--- | :--- | :--- |
| `value` | Constant-time string equality comparison. | `value`, `parameter` |
| `regex` | Regular expression match against parameter value. | `regex`, `parameter` |
| `payload-hmac-sha1` | HMAC-SHA1 verification against raw request body. | `secret`, `parameter` |
| `payload-hmac-sha256` | HMAC-SHA256 verification against raw request body. | `secret`, `parameter` |
| `payload-hmac-sha512` | HMAC-SHA512 verification against raw request body. | `secret`, `parameter` |
| `ip-whitelist` | Matches client IP against whitespace-delimited IPs/CIDRs. | `ip-whitelist` |
| `scalr-signature` | Validates Scalr-compliant HMAC-SHA1 timestamp and signature. | `secret` |
