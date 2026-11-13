# Webhook-Async Documentation

> **A standalone Rust webhook receiver and asynchronous job engine.**

[**Deployment Guide**](deployment.html) • [**Configuration Reference**](configuration.html) • [**API Reference**](api.html) • [**Operations & Backups**](operations.html) • [**Security**](security.html) • [**Troubleshooting**](troubleshooting.html) • [**GitHub Repository**](https://github.com/sunilsankar/webhook-async)

---

## What is `webhook-async`?

`webhook-async` is a modern, memory-safe, standalone executable designed to receive HTTP webhooks and run tasks with complete reliability.

It solves the two biggest challenges of traditional webhook infrastructure:
1. **Preserving Upstream Compatibility**: Full compatibility with `adnanh/webhook` v2.8.3 hook definitions, trigger rules, HMAC signatures, IP whitelists, and template preprocessing.
2. **Eliminating Dropped Webhooks & Timeouts**: Provides a durable, transactional queue engine (SQLite, Valkey, Redis) that commits jobs before acknowledging HTTP `202 Accepted`, running tasks asynchronously with retries and concurrency limits.

---

## Core Features

- **Single Self-Contained Binary**: Written in pure Rust. No Node.js runtime, no Python, no mandatory Docker engine.
- **Dual Execution Engine**:
  - **Synchronous**: Direct process execution returning stdout/stderr immediately; requires zero database or queue storage.
  - **Asynchronous**: Durable queue committing an immutable execution snapshot before returning HTTP 202.
- **Pluggable Storage Backends**:
  - **SQLite** (default): Embedded, WAL-mode, multi-threaded with busy timeouts.
  - **Valkey & Redis**: Distributed scale-out using atomic Lua scripts and clustered hash-tagged keys.
- **Integrated Web UI**: Lightweight Askama server-rendered templates, HTMX reactive components, and Server-Sent Events (SSE).
- **Hardened Process Execution**: Commands run with an explicit argument vector (no shell string interpolation), process group isolation (`-pgid`), execution timeouts, and automatic temporary file cleanup.
- **Dynamic Hot Reloading**: Update hook definitions on the fly via `SIGHUP`, `SIGUSR1`, or `--hotreload` polling with zero downtime.

---

## Architecture at a Glance

```
                      [ Incoming Webhook / Client ]
                                   │
                                   ▼
                       ┌────────────────────────┐
                       │   Axum HTTP Receiver   │
                       │   (Auth, Trigger Rules)│
                       └───────────┬────────────┘
                                   │
                 ┌─────────────────┴─────────────────┐
                 │ Mode Resolution                   │
                 ▼                                   ▼
        [ Synchronous Hook ]               [ Asynchronous Hook ]
                 │                                   │
                 ▼                                   ▼
       ┌──────────────────┐               ┌─────────────────────┐
       │ Direct Executor  │               │ Storage / Queue     │
       │ (Process Group)  │               │ (SQLite / Valkey)   │
       └─────────┬────────┘               └──────────┬──────────┘
                 │                                   │ (HTTP 202)
                 ▼                                   ▼
         Combined Output                    ┌─────────────────┐
          (HTTP 200/500)                    │ Worker Pool     │
                                            │ (Heartbeat, Ack)│
                                            └────────┬────────┘
                                                     ▼
                                            [ Direct Executor ]
```

---

## 60-Second Quickstart

### 1. Download the Binary
Download the precompiled standalone binary for your architecture from [GitHub Releases](https://github.com/sunilsankar/webhook-async/releases):

```bash
# Linux x86_64 example
curl -L -o webhook-async https://github.com/sunilsankar/webhook-async/releases/download/v0.1.0-beta/webhook-async-linux-amd64
chmod +x webhook-async
sudo mv webhook-async /usr/local/bin/
```

### 2. Define a Minimal Hook (`hooks.json`)
```json
[
  {
    "id": "deploy-app",
    "execute-command": "/bin/echo",
    "pass-arguments-to-command": [
      { "source": "payload", "name": "ref" },
      { "source": "payload", "name": "commit" }
    ],
    "response-message": "Deployment initiated.",
    "include-command-output-in-response": true
  }
]
```

### 3. Run the Server
```bash
webhook-async server --port 9000 --hooks hooks.json
```

### 4. Trigger the Webhook
```bash
curl -X POST http://localhost:9000/hooks/deploy-app \
  -H "Content-Type: application/json" \
  -d '{"ref": "refs/heads/main", "commit": "05dc3f3"}'
```
Response:
```text
refs/heads/main 05dc3f3
```

---

## Documentation Navigation

- [**Deployment Guide**](deployment.html): Step-by-step guides for Linux (systemd), Docker Compose, Kubernetes, macOS (launchd), Windows, and Reverse Proxies (Caddy/Nginx).
- [**Configuration Reference**](configuration.html): Complete schema for `config.toml`, hook definitions, trigger rules, parameter extraction, and template preprocessing.
- [**API Reference**](api.html): Complete documentation for receiver endpoints, health checks, administrative REST API (`/api/v1/*`), and SSE real-time stream.
- [**Operations & Backups**](operations.html): Signal handling, zero-downtime hot reload, non-blocking SQLite hot backups (`VACUUM INTO`), logging, and disaster recovery.
- [**Security Hardening**](security.html): Threat model, sandboxing, timing-attack prevention, and secret management.
- [**Troubleshooting Matrix**](troubleshooting.html): Common operational issues and fixes.
