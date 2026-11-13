# Deployment Guide

[← Back to Home](index.md) • [Configuration Reference](configuration.md) • [API Reference](api.md) • [Operations](operations.md)

---

## Table of Contents
1. [Linux Bare Metal / VM (systemd)](#linux-bare-metal-vm-systemd)
2. [Docker & Docker Compose](#docker-docker-compose)
3. [Kubernetes](#kubernetes)
4. [macOS (launchd)](#macos-launchd)
5. [Windows Server (NSSM)](#windows-server-nssm)
6. [Distributed Scale-Out (Dedicated Workers)](#distributed-scale-out-dedicated-workers)
7. [Reverse Proxy & Ingress (Caddy / Nginx)](#reverse-proxy-ingress-caddy-nginx)
8. [Direct TLS & Unix Sockets](#direct-tls-unix-sockets)

---

## Linux Bare Metal / VM (systemd)

Recommended for production Debian, Ubuntu, RHEL, Rocky, or Alpine Linux servers.

### 1. Create Dedicated Service User
```bash
sudo useradd -r -s /usr/sbin/nologin -d /var/lib/webhook-async webhook
sudo mkdir -p /etc/webhook-async /var/lib/webhook-async /var/log/webhook-async
sudo chown -R webhook:webhook /var/lib/webhook-async /var/log/webhook-async
```

### 2. Install Binary
Download from [GitHub Releases](https://github.com/sunilsankar/webhook-async/releases):
```bash
sudo curl -L -o /usr/local/bin/webhook-async https://github.com/sunilsankar/webhook-async/releases/download/v0.1.0-beta/webhook-async-linux-amd64
sudo chmod +x /usr/local/bin/webhook-async
```

### 3. Create Configuration (`/etc/webhook-async/config.toml`)
```toml
[server]
host = "127.0.0.1"
port = 9000
hook_files = ["/etc/webhook-async/hooks.json"]
hot_reload = true

[storage]
backend = "sqlite"

[storage.sqlite]
database_url = "sqlite:///var/lib/webhook-async/data.db?mode=rwc"
wal_mode = true
busy_timeout_ms = 5000

[worker]
concurrency = 4

[ui]
enabled = true
bind = "127.0.0.1:9001"

[auth]
admin_token_env = "WEBHOOK_ADMIN_TOKEN"
```

### 4. Create systemd Unit (`/etc/systemd/system/webhook-async.service`)
```ini
[Unit]
Description=webhook-async standalone receiver and job engine
After=network.target

[Service]
Type=simple
User=webhook
Group=webhook
Environment="WEBHOOK_ADMIN_TOKEN=your-random-32-char-token-here"
ExecStart=/usr/local/bin/webhook-async server --config /etc/webhook-async/config.toml
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=5s
KillMode=mixed
TimeoutStopSec=35s
LimitNOFILE=65536

# Sandboxing & Hardening
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/lib/webhook-async /var/log/webhook-async
PrivateTmp=true
NoNewPrivileges=true

[Install]
WantedBy=multi-user.target
```

### 5. Enable and Start
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now webhook-async
sudo systemctl status webhook-async
```

---

## Docker & Docker Compose

### 1. `Dockerfile`
```dockerfile
# Build Stage
FROM rust:1.89-alpine AS builder
RUN apk add --no-cache musl-dev
WORKDIR /build
COPY . .
RUN cargo build --release

# Runtime Stage
FROM alpine:3.20
RUN apk add --no-cache bash ca-certificates curl jq git
RUN addgroup -S webhook && adduser -S webhook -G webhook
WORKDIR /app
COPY --from=builder /build/target/release/webhook-async /usr/local/bin/webhook-async
RUN mkdir -p /data && chown -R webhook:webhook /data
USER webhook
EXPOSE 9000 9001
VOLUME ["/data", "/config"]
ENTRYPOINT ["/usr/local/bin/webhook-async"]
CMD ["server", "--config", "/config/config.toml"]
```

### 2. `docker-compose.yml` (With Valkey Distributed Queue)
```yaml
version: "3.8"

services:
  valkey:
    image: valkey/valkey:8-alpine
    restart: unless-stopped
    command: ["valkey-server", "--appendonly", "yes"]
    volumes:
      - valkey-data:/data
    networks:
      - webhook-net

  webhook-api:
    image: ghcr.io/sunilsankar/webhook-async:v0.1.0-beta
    restart: unless-stopped
    command: ["server", "--config", "/config/config.toml", "--async"]
    ports:
      - "9000:9000"
      - "127.0.0.1:9001:9001"
    environment:
      - WEBHOOK_ADMIN_TOKEN=super-secure-admin-token
    volumes:
      - ./config:/config:ro
      - ./scripts:/scripts:ro
    depends_on:
      - valkey
    networks:
      - webhook-net

  webhook-worker:
    image: ghcr.io/sunilsankar/webhook-async:v0.1.0-beta
    restart: unless-stopped
    command: ["worker", "--config", "/config/config.toml"]
    environment:
      - WEBHOOK_ADMIN_TOKEN=super-secure-admin-token
    volumes:
      - ./config:/config:ro
      - ./scripts:/scripts
    depends_on:
      - valkey
    networks:
      - webhook-net

volumes:
  valkey-data:

networks:
  webhook-net:
    driver: bridge
```

---

## Kubernetes

Deploy `webhook-async` with decoupled HTTP ingress pods and dedicated worker pods.

### 1. ConfigMap and Secrets
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: webhook-secrets
type: Opaque
stringData:
  admin-token: "generate-a-strong-secret-token"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: webhook-config
data:
  config.toml: |
    [server]
    host = "0.0.0.0"
    port = 9000
    hook_files = ["/etc/webhook/hooks.json"]

    [storage]
    backend = "valkey"
    [storage.valkey]
    url = "redis://valkey-cluster:6379/0"
    key_prefix = "k8s_webhook:"

    [worker]
    concurrency = 8
    shutdown_grace_seconds = 30

    [auth]
    admin_token_env = "WEBHOOK_ADMIN_TOKEN"

  hooks.json: |
    [
      {
        "id": "ci-deploy",
        "execute-command": "/scripts/deploy.sh",
        "async": true
      }
    ]
```

### 2. Receiver Deployment & Service
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webhook-receiver
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webhook-receiver
  template:
    metadata:
      labels:
        app: webhook-receiver
    spec:
      containers:
        - name: receiver
          image: ghcr.io/sunilsankar/webhook-async:v0.1.0-beta
          args: ["server", "--config", "/etc/webhook/config.toml", "--async"]
          ports:
            - containerPort: 9000
          env:
            - name: WEBHOOK_ADMIN_TOKEN
              valueFrom:
                secretKeyRef:
                  name: webhook-secrets
                  key: admin-token
          livenessProbe:
            httpGet:
              path: /api/v1/health
              port: 9000
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /api/v1/ready
              port: 9000
            initialDelaySeconds: 5
            periodSeconds: 5
          volumeMounts:
            - name: config-vol
              mountPath: /etc/webhook
      volumes:
        - name: config-vol
          configMap:
            name: webhook-config
---
apiVersion: v1
kind: Service
metadata:
  name: webhook-service
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 9000
  selector:
    app: webhook-receiver
```

---

## macOS (launchd)

### 1. Install Binary
```bash
curl -L -o /usr/local/bin/webhook-async https://github.com/sunilsankar/webhook-async/releases/download/v0.1.0-beta/webhook-async-macos-arm64
chmod +x /usr/local/bin/webhook-async
```

### 2. Create launchd Agent (`~/Library/LaunchAgents/com.sunilsankar.webhook-async.plist`)
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.sunilsankar.webhook-async</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/webhook-async</string>
        <string>server</string>
        <string>--config</string>
        <string>/Users/youruser/.config/webhook-async/config.toml</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/Users/youruser/.config/webhook-async/stdout.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/youruser/.config/webhook-async/stderr.log</string>
</dict>
</plist>
```

### 3. Load Agent
```bash
launchctl load ~/Library/LaunchAgents/com.sunilsankar.webhook-async.plist
```

---

## Windows Server (NSSM)

Deploy as a managed Windows Service using [NSSM](https://nssm.cc/).

1. Download `webhook-async-windows-amd64.exe` to `C:\webhook-async\webhook-async.exe`.
2. Create `C:\webhook-async\config.toml` and `C:\webhook-async\hooks.json`.
3. Open Administrator PowerShell and install the service:
```powershell
nssm install WebhookAsync "C:\webhook-async\webhook-async.exe" "server --config C:\webhook-async\config.toml"
nssm set WebhookAsync AppDirectory "C:\webhook-async"
nssm set WebhookAsync AppStdout "C:\webhook-async\logs\stdout.log"
nssm set WebhookAsync AppStderr "C:\webhook-async\logs\stderr.log"
nssm set WebhookAsync Start SERVICE_AUTO_START
nssm start WebhookAsync
```

---

## Distributed Scale-Out (Dedicated Workers)

In high-volume environments, separate the HTTP API reception layer from execution workers:

```
                          [ Ingress Load Balancer ]
                                     │
                 ┌───────────────────┴───────────────────┐
                 ▼                                       ▼
       [ API Node 1 ]                          [ API Node 2 ]
       webhook-async server --async            webhook-async server --async
       (worker.concurrency = 0)                (worker.concurrency = 0)
                 │                                       │
                 └───────────────────┬───────────────────┘
                                     │ (Enqueues Job Snapshot)
                                     ▼
                        [ Valkey / Redis Cluster ]
                                     │
                 ┌───────────────────┴───────────────────┐
                 │ (Atomic Lua Claim & Heartbeat)        │
                 ▼                                       ▼
      [ Dedicated Worker 1 ]                  [ Dedicated Worker 2 ]
      webhook-async worker                    webhook-async worker
      (worker.concurrency = 32)               (worker.concurrency = 32)
```

- **API Nodes**: Run `webhook-async server --async --config config.toml` with `worker.concurrency = 0`.
- **Worker Nodes**: Run `webhook-async worker --config config.toml` with `worker.concurrency = 32`.

---

## Reverse Proxy & Ingress (Caddy / Nginx)

### Caddy (Automatic HTTPS)
```caddyfile
webhook.example.com {
    reverse_proxy 127.0.0.1:9000

    # Isolated admin dashboard accessible only from VPN/internal IPs
    handle_path /admin/* {
        @internal remote_ip 10.0.0.0/8 192.168.0.0/16 127.0.0.1
        reverse_proxy @internal 127.0.0.1:9001
    }
}
```

### Nginx
```nginx
upstream webhook_backend {
    server 127.0.0.1:9000;
    keepalive 32;
}

server {
    listen 443 ssl http2;
    server_name webhook.example.com;

    ssl_certificate /etc/letsencrypt/live/webhook.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/webhook.example.com/privkey.pem;

    client_max_body_size 10M;

    location / {
        proxy_pass http://webhook_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Request-Id $request_id;
    }

    # Real-time SSE streaming support
    location /api/v1/events {
        proxy_pass http://webhook_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_buffering off;
        proxy_cache off;
        proxy_read_timeout 24h;
    }
}
```

---

## Direct TLS & Unix Sockets

```bash
# Direct TLS listening on 9443
webhook-async server \
  --port 9443 \
  --secure \
  --cert /etc/ssl/certs/webhook.crt \
  --key /etc/ssl/private/webhook.key \
  --hooks /etc/webhook/hooks.json

# Direct Unix domain socket
webhook-async server \
  --socket /run/webhook-async.sock \
  --hooks /etc/webhook/hooks.json
```
