---
title: Configuration Reference
layout: post
---

# Configuration Reference

[← Back to Home](index.html) • [Deployment Guide](deployment.html) • [API Reference](api.html) • [Operations](operations.html)

---

## 1. Application Configuration (`config.toml`)

`webhook-async` uses a single TOML file for server and system configuration.

```toml
[server]
host = "0.0.0.0"                        # Bind address
port = 9000                             # Bind port
url_prefix = "hooks"                    # Route prefix (e.g. /hooks/{id})
max_body_bytes = 10485760               # Maximum request body size (10 MB)
max_multipart_mem_bytes = 1048576       # Memory buffer limit for multipart parts (1 MB)
hook_files = ["/etc/webhook/hooks.json"]# Paths to hook definition files
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
admin_token_env = "WEBHOOK_ADMIN_TOKEN" # Environment variable containing admin Bearer token
# admin_token_file = "/run/secrets/admin_token" # Or file path containing admin token
```

---

## 2. Hook Definitions (`hooks.json` / `hooks.yaml`)

Hooks can be configured in either JSON or YAML.

### Complete Hook Schema

```json
[
  {
    "id": "github-webhook",
    "execute-command": "/opt/scripts/deploy.sh",
    "command-working-directory": "/opt/repo",
    "response-message": "Deployment initiated.",
    "response-headers": [
      { "name": "X-Custom-Header", "value": "webhook-processed" }
    ],
    "include-command-output-in-response": false,
    "include-command-output-in-response-on-error": true,
    "http-methods": ["POST"],
    "success-http-response-code": 200,
    "trigger-rule-mismatch-http-response-code": 400,
    "trigger-signature-soft-failures": false,
    "incoming-payload-content-type": "application/json",
    "async": true,

    "trigger-rule": {
      "and": [
        {
          "match": {
            "type": "payload-hmac-sha256",
            "secret": "{{ getenv \"GITHUB_SECRET\" }}",
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
    },

    "pass-arguments-to-command": [
      { "source": "payload", "name": "head_commit.id" },
      { "source": "payload", "name": "pusher.name" },
      { "source": "request", "name": "remote-addr" }
    ],

    "pass-environment-to-command": [
      {
        "source": "payload",
        "name": "repository.name",
        "envname": "REPO_NAME"
      },
      {
        "source": "header",
        "name": "X-GitHub-Delivery",
        "envname": "DELIVERY_ID"
      }
    ],

    "pass-file-to-command": [
      {
        "source": "entire-payload",
        "envname": "PAYLOAD_FILE"
      }
    ],

    "parse-parameters-as-json": [
      {
        "source": "payload",
        "name": "custom_data"
      }
    ]
  }
]
```

---

## 3. Parameter Extraction (`source`)

Parameters can be extracted and mapped to command arguments, environment variables, files, or trigger rule checks.

| Source | Description | Example |
| :--- | :--- | :--- |
| `header` | Canonical MIME HTTP request header. | `{"source": "header", "name": "X-GitHub-Event"}` |
| `url` / `query` | URL query parameter. | `{"source": "url", "name": "branch"}` |
| `payload` | Body value from JSON, XML, or Form payload. Supports dot notation. | `{"source": "payload", "name": "commit.author.email"}` |
| `raw-request-body` | The unparsed UTF-8 body string. | `{"source": "raw-request-body"}` |
| `request` | Request metadata (`remote-addr`, `method`). | `{"source": "request", "name": "remote-addr"}` |
| `string` | Static literal string. | `{"source": "string", "name": "production"}` |
| `entire-payload` | Full raw body written to temporary file (`pass-file-to-command`). | `{"source": "entire-payload", "envname": "BODY_FILE"}` |
| `entire-headers` | JSON map of all headers. | `{"source": "entire-headers", "envname": "HEADERS_JSON"}` |
| `entire-query` | JSON map of all query parameters. | `{"source": "entire-query", "envname": "QUERY_JSON"}` |

---

## 4. Trigger Rules (`trigger-rule`)

Trigger rules determine whether the webhook is allowed to run.

### Rule Combinators
- `and`: Array of rules; all must evaluate to `true`.
- `or`: Array of rules; passes if any rule evaluates to `true`.
- `not`: Inverts the result of a child rule.

### Matcher Types (`type`)

#### 1. `value` (String Equality)
Constant-time equality comparison:
```json
{
  "match": {
    "type": "value",
    "value": "refs/heads/main",
    "parameter": { "source": "payload", "name": "ref" }
  }
}
```

#### 2. `regex` (Pattern Match)
Matches value against a regular expression:
```json
{
  "match": {
    "type": "regex",
    "regex": "^v[0-9]+\\.[0-9]+\\.[0-9]+$",
    "parameter": { "source": "payload", "name": "ref" }
  }
}
```

#### 3. `payload-hmac-sha1`, `payload-hmac-sha256`, `payload-hmac-sha512`
Constant-time HMAC hex verification with secret:
```json
{
  "match": {
    "type": "payload-hmac-sha256",
    "secret": "{{ getenv \"WEBHOOK_SECRET\" }}",
    "parameter": { "source": "header", "name": "X-Hub-Signature-256" }
  }
}
```
*Supports optional prefix format (e.g. `sha256=abcdef...`) and comma-separated multiple signatures.*

#### 4. `ip-whitelist`
Verifies client IP against whitespace-delimited IPs or CIDR blocks:
```json
{
  "match": {
    "type": "ip-whitelist",
    "ip-whitelist": "192.168.1.0/24 10.0.0.1 2001:db8::/32"
  }
}
```

#### 5. `scalr-signature`
Validates Scalr-compliant HMAC-SHA1 timestamp and signature:
```json
{
  "match": {
    "type": "scalr-signature",
    "secret": "{{ getenv \"SCALR_KEY\" }}"
  }
}
```

---

## 5. Execution Mode Precedence

Execution mode is resolved using this strict precedence hierarchy:

```
                  ┌───────────────────────────────┐
                  │ Is hook "async" specified?    │
                  └───────────────┬───────────────┘
                                  │
                 ┌────────────────┴────────────────┐
                 ▼ YES                             ▼ NO
     ┌───────────────────────┐         ┌───────────────────────────────┐
     │ Use hook.async value  │         │ CLI flag --async / --sync?    │
     │ (true/false)          │         └───────────────┬───────────────┘
     └───────────────────────┘                         │
                                      ┌────────────────┴────────────────┐
                                      ▼ Set                             ▼ None
                           ┌─────────────────────┐          ┌───────────────────────┐
                           │ Use CLI flag mode   │          │ Default: SYNCHRONOUS  │
                           └─────────────────────┘          └───────────────────────┘
```

---

## 6. Template Preprocessing (`--template`)

Start the server or validation with `--template` to enable dynamic configuration preprocessing:

- `{{ getenv "VAR_NAME" "fallback" }}`: Injects environment variable with optional default.
- `{{ cat "/path/to/file" }}`: Reads file content and trims trailing newline.
- `{{ credential "secret_name" }}`: Reads systemd `$CREDENTIALS_DIRECTORY` file with fallback to env var.
- `{{ ... | js }}`: Encodes value as a safe JSON string literal.
