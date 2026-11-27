---
title: "Windows + Debian 13 (VMware) Local Dev & Always-On Live Preview Guide: Mutagen + Docker Compose + Caddy Offline HTTPS + Dockge"
author: "John Yang"
lang: en
pubDatetime: 2026-11-27T13:30:00+08:00
featured: true
tags:
  - DevOps
  - Docker
  - Linux
  - Caddy
  - Mutagen
description: "A generic, reproducible hybrid development environment guide: keep source code on Windows, sync in real time via Mutagen to Debian 13 native ext4, run always-on Go/Node/PostgreSQL 17 dev containers with Docker Compose, and serve everything over offline local HTTPS via Caddy and Dockge."
---

A common cross-platform developer workflow bottleneck is **coding on Windows (IDE, AI assistants, desktop tools) while needing a Linux runtime (Docker, isolated Go/Node versions, PostgreSQL, and native Linux `inotify`)**.

Running databases and multiple dev servers directly on Windows pollutes the host and requires manually starting terminals after every reboot. Conversely, mounting Windows folders into a VMware Linux guest via HGFS shared folders or SMB breaks `inotify` file watchers (killing HMR hot reload) and causes severe `node_modules` I/O degradation.

This guide walks through a **100% offline-capable, auto-starting, live-reloading hybrid dev architecture** where Windows remains the single source of truth for code while a Debian 13 VM hosts the always-on runtime.

---

## 1. Architecture & Core Design Principles

```text
┌─────────────────────────────┐         SSH + Mutagen (one-way-replica)         ┌───────────────────────────────────────┐
│        Windows Host         │ ──────────────────────────────────────────────▶ │        Debian 13 VM (ext4)            │
│                             │  Sync source only; ignore .git/node_modules/dist│                                       │
│ - Source Truth (D:\Projects)│                                                 │ /opt/stacks/ (Unified Dockge Root)    │
│ - IDE / Editors             │                                                 │  ├── caddy/     (Offline HTTPS Proxy) │
│ - Mutagen Daemon (Auto-run) │ ◀────────────────────────────────────────────── │  ├── dockge/    (Web Stack Manager)   │
│ - hosts + Local Root CA     │        https://*.test (Trusted HTTPS + wss://)  │  ├── postgres/  (Shared PG17 + dev-db)│
└─────────────────────────────┘                                                 │  ├── app-fullstack/ (Go+Vite+Demo)    │
                                                                                │  └── blog-site/     (Astro/Node)      │
                                                                                └───────────────────────────────────────┘
```

### Why This Stack?
1. **Source on Windows, Dependencies on Linux**: Windows stores clean source files; `node_modules`, Go module caches, and database volumes live inside Debian 13 native `ext4` Docker volumes, preserving native `inotify` millisecond-level HMR.
2. **Custom `.test` Domains + Caddy Offline HTTPS instead of `IP:Port`**:
   - **Browser Secure Context**: Browsers only treat `localhost` and `https://` as Secure Contexts. Plain LAN `http://192.168.x.x:5173` disables `navigator.clipboard.writeText`, `crypto.subtle`, and `Secure` cookies.
   - **Cookie & Storage Isolation**: Different ports on the same IP share cookies; separate `.test` domains keep auth state isolated per project.
   - **Automatic `wss://` Upgrades**: Caddy `reverse_proxy` transparently upgrades Vite/Astro HMR WebSockets to `wss://`.
3. **Single Unified Root `/opt/stacks/`**: Infrastructure stacks (Caddy, Dockge, PostgreSQL) and application stacks live side-by-side under `/opt/stacks/`, enabling Dockge to discover and manage everything from one place.

---

## 2. Debian 13 Baseline Tuning (One-Time Setup)

Assuming you have a Debian 13 VM in VMware Workstation with a static LAN IP (e.g., `192.168.200.50`) and official **Docker Engine + Docker Compose** installed:

### 1. Enable Docker on Boot & Verify `open-vm-tools`
```bash
systemctl enable --now docker
systemctl is-enabled docker

# Required for VMware graceful guest shutdown coordination
apt update && apt install -y open-vm-tools
systemctl is-active open-vm-tools
```

### 2. Configure Swap & `swappiness`
```bash
fallocate -l 4G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab

echo 'vm.swappiness=10' > /etc/sysctl.d/99-swappiness.conf
sysctl -p /etc/sysctl.d/99-swappiness.conf
```

### 3. Raise `inotify` Limits (Critical for Multiple Watchers)
Running Mutagen alongside Go `air` and multiple Vite/Astro dev servers quickly exhausts Debian's default `fs.inotify.max_user_instances` (`128`), causing silent HMR failures:
```bash
cat <<'EOF' > /etc/sysctl.d/99-inotify.conf
fs.inotify.max_user_watches=524288
fs.inotify.max_user_instances=1024
EOF
sysctl -p /etc/sysctl.d/99-inotify.conf
```

### 4. Create the Unified Stacks Directory
```bash
mkdir -p /opt/stacks
```

---

## 3. Passwordless SSH from Windows to Debian 13

Mutagen requires non-interactive SSH access to maintain background sync sessions. In **Windows PowerShell**:

```powershell
# 1. Generate an ed25519 key if one does not exist
if (-not (Test-Path "$env:USERPROFILE\.ssh\id_ed25519.pub")) {
  ssh-keygen -t ed25519 -N '""' -f "$env:USERPROFILE\.ssh\id_ed25519"
}

# 2. Upload the public key to Debian 13
$pubKey = (Get-Content "$env:USERPROFILE\.ssh\id_ed25519.pub" -Raw).Trim()
ssh root@192.168.200.50 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && touch ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys && (grep -qxF '$pubKey' ~/.ssh/authorized_keys || echo '$pubKey' >> ~/.ssh/authorized_keys)"
```

Add an SSH alias with keepalive to `C:\Users\<Username>\.ssh\config`:
```sshconfig
Host debian-dev
    HostName 192.168.200.50
    User root
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
    ServerAliveInterval 30
    ServerAliveCountMax 3
```
Verify with `ssh -o BatchMode=yes debian-dev "hostname"`.

---

## 4. Deploy Shared Infrastructure (Caddy + Dockge + PostgreSQL 17)

### 1. Caddy Offline HTTPS Reverse Proxy (`/opt/stacks/caddy/`)

Caddy's `tls internal` directive uses an embedded local CA to issue trusted HTTPS certificates **completely offline**.

Create `/opt/stacks/caddy/compose.yaml` (pinning an exact version tag):
```yaml
services:
  caddy:
    image: caddy:2.11.9-alpine
    container_name: dev-caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config
    networks:
      - dev-proxy

volumes:
  caddy_data:
  caddy_config:

networks:
  dev-proxy:
    name: dev-proxy
    driver: bridge
```

Create `/opt/stacks/caddy/Caddyfile`:
```caddyfile
dockge.test {
    tls internal
    reverse_proxy dockge:5001
}

app.test {
    tls internal
    reverse_proxy app-web:5173
}

app-demo.test {
    tls internal
    reverse_proxy app-web-demo:5174
}

blog.test {
    tls internal
    reverse_proxy blog-web:4321
}
```

Start Caddy:
```bash
cd /opt/stacks/caddy && docker compose up -d
```

> **Gotcha**: When editing a single-file bind-mounted `Caddyfile` with tools like `sed -i` (which replace the file inode), run `docker restart dev-caddy` so the container picks up the new inode.

### 2. Trust Caddy's Root CA on Windows & Update `hosts`

1. **Export the local root certificate**:
   ```powershell
   ssh debian-dev "docker exec dev-caddy cat /data/caddy/pki/authorities/local/root.crt" | Out-File "$env:USERPROFILE\Desktop\caddy-local-root.crt" -Encoding ascii
   ```
2. **Install into Windows Trust Store**:
   - Double-click `caddy-local-root.crt` → **Install Certificate** → **Current User** → **Place all certificates in the following store** → **Browse** → **Trusted Root Certification Authorities** → Finish.
3. **Update `hosts`** (in an **Administrator PowerShell**):
   ```powershell
   Add-Content -Path "$env:SystemRoot\System32\drivers\etc\hosts" -Value "`r`n192.168.200.50 dockge.test app.test app-demo.test blog.test" -Encoding ascii
   ipconfig /flushdns
   ```

> **Note**: Windows `curl.exe` (Schannel) lacks an online CRL for private CAs and may report `CRYPT_E_NO_REVOCATION_CHECK`; pass `--ssl-no-revoke` when testing from CLI. Browsers (Chrome/Edge) trust it without warnings.

### 3. Dockge Container Manager (`/opt/stacks/dockge/`)

Create `/opt/stacks/dockge/compose.yaml`:
```yaml
services:
  dockge:
    image: louislam/dockge:1.5.0
    container_name: dockge
    restart: unless-stopped
    ports:
      - "5001:5001"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./data:/app/data
      # Host and container paths MUST match for Dockge relative volume resolution
      - /opt/stacks:/opt/stacks
    environment:
      - DOCKGE_STACKS_DIR=/opt/stacks
      - DOCKGE_ENABLE_CONSOLE=false
    networks:
      - dev-proxy

networks:
  dev-proxy:
    external: true
```
Start it with `cd /opt/stacks/dockge && mkdir -p data && docker compose up -d`, then open `https://dockge.test`.

### 4. Shared PostgreSQL 17 Stack (`/opt/stacks/postgres/`)

Keeping PostgreSQL in its own stack prevents database restarts when cycling application containers and lets multiple projects share one instance via separate databases:

```yaml
services:
  postgres:
    image: postgres:17.12-alpine3.24
    container_name: dev-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: <YOUR_STRONG_DEV_PASSWORD>
    ports:
      - "5432:5432"
    volumes:
      - pg_data:/var/lib/postgresql/data
    networks:
      - dev-db
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s

volumes:
  pg_data:

networks:
  dev-db:
    name: dev-db
    driver: bridge
```
Start and create databases:
```bash
cd /opt/stacks/postgres && docker compose up -d
docker exec dev-postgres createdb -U postgres app_dev
docker exec dev-postgres createdb -U postgres app_test
```

---

## 5. Install Mutagen on Windows & Create Sync Sessions

### 1. Install Mutagen
Download `mutagen_windows_amd64_<version>.tar.gz` (e.g., `v0.18.1`) from [mutagen-io/mutagen Releases](https://github.com/mutagen-io/mutagen/releases), extract to `C:\Users\<Username>\bin\mutagen\`, and add that folder to your user `PATH`.

### 2. Create `one-way-replica` Sessions
Create the target directories on Debian 13 first, then run on Windows:
```powershell
mutagen daemon start

# Full-stack project
mutagen sync create "D:\Projects\app-fullstack" "debian-dev:/opt/stacks/app-fullstack" `
  --name app-fullstack `
  --sync-mode one-way-replica `
  --ignore-vcs `
  --ignore .git `
  --ignore node_modules `
  --ignore /web/dist `
  --ignore .astro `
  --ignore "*.log" `
  --ignore /compose.yaml `
  --ignore /server/.env `
  --ignore /server/uploads `
  --ignore /server/tmp

# Static/Astro site
mutagen sync create "D:\Projects\blog-site" "debian-dev:/opt/stacks/blog-site" `
  --name blog-site `
  --sync-mode one-way-replica `
  --ignore-vcs `
  --ignore .git `
  --ignore node_modules `
  --ignore dist `
  --ignore .astro `
  --ignore "*.log" `
  --ignore /compose.yaml
```

> **Important**:
> - Always ignore Debian-specific files (`/compose.yaml`, `/server/.env`, etc.) in `one-way-replica` mode so Mutagen does not delete them on the VM.
> - If your Go backend uses `//go:embed all:dist`, use a scoped ignore like `--ignore /web/dist` rather than a global `--ignore dist` so placeholder directories in `server/` are preserved.

### 3. Auto-Start Mutagen on Windows Login
Open **Task Scheduler (`taskschd.msc`)** → **Create Basic Task**:
- **Trigger**: When I log on
- **Action**: Start a program
  - **Program/script**: `C:\Users\<Username>\bin\mutagen\mutagen.exe`
  - **Add arguments**: `daemon start`

---

## 6. Project Compose Templates

### Template A: Astro / Vite Frontend (`/opt/stacks/blog-site/compose.yaml`)

Skip reinstalling dependencies when `node_modules/.bin/astro` already exists in the volume so offline VM reboots start in seconds:

```yaml
services:
  blog-web:
    image: node:22.14.0-bookworm-slim
    container_name: blog-web
    restart: unless-stopped
    working_dir: /app
    ports:
      - "4321:4321"
    command: >
      bash -c "
      ([ -x /app/node_modules/.bin/astro ] || npm ci)
      && npm run dev -- --host 0.0.0.0 --port 4321"
    volumes:
      - ./:/app
      - blog_node_modules:/app/node_modules
    networks:
      - dev-proxy

volumes:
  blog_node_modules:

networks:
  dev-proxy:
    external: true
```
Remember to add `'blog.test'` to `server.allowedHosts` in `astro.config.*` / `vite.config.*` (this only affects the dev server and has zero impact on production builds).

### Template B: Go (`air` Live Reload) + Dual Vite Frontends (Real API + Demo Mock)

Run both a **real backend-connected frontend** (`https://app.test`) and a **mocked demo frontend** (`https://app-demo.test`) from the same synced `./web` source:

```yaml
services:
  app-server:
    image: golang:1.27.4-bookworm
    container_name: app-server
    restart: unless-stopped
    working_dir: /app/server
    ports:
      - "8080:8080"
    env_file:
      - ./server/.env
    environment:
      # Must bind 0.0.0.0 inside containers
      HOST: "0.0.0.0"
      PORT: "8080"
    command: >
      bash -c "
      ([ -x /go/bin/air ] || go install github.com/air-verse/air@v1.61.7)
      && air --build.cmd 'go build -o ./tmp/main ./cmd/server'
             --build.include_ext 'go,sql,yaml,tmpl'
             --build.exclude_dir 'tmp,uploads,.git'
      "
    volumes:
      - ./server:/app/server
      - app_gomod:/go/pkg/mod
      - app_gocache:/root/.cache/go-build
      - app_gobin:/go/bin
      - app_uploads:/app/uploads
    networks:
      - dev-db
      - dev-proxy

  app-web:
    image: node:24.13.0-bookworm-slim
    container_name: app-web
    restart: unless-stopped
    working_dir: /app
    ports:
      - "5173:5173"
    environment:
      VITE_USE_MOCK: "false"
      VITE_API_PROXY_TARGET: "http://app-server:8080"
    command: >
      bash -c "
      ([ -x /app/node_modules/.bin/vite ] || npm ci)
      && npm run dev -- --host 0.0.0.0 --port 5173
      "
    volumes:
      - ./web:/app
      - app_node_modules:/app/node_modules
    depends_on:
      - app-server
    networks:
      - dev-proxy

  app-web-demo:
    image: node:24.13.0-bookworm-slim
    container_name: app-web-demo
    restart: unless-stopped
    working_dir: /app
    ports:
      - "5174:5174"
    environment:
      VITE_USE_MOCK: "true"
      VITE_API_PROXY_TARGET: "http://app-server:8080"
    command: >
      bash -c "
      until [ -x /app/node_modules/.bin/vite ]; do sleep 2; done
      && ./node_modules/.bin/vite --mode demo --host 0.0.0.0 --port 5174
      "
    volumes:
      - ./web:/app
      - app_node_modules:/app/node_modules
    depends_on:
      - app-web
    networks:
      - dev-proxy

volumes:
  app_gomod:
  app_gocache:
  app_gobin:
  app_uploads:
  app_node_modules:

networks:
  dev-db:
    external: true
  dev-proxy:
    external: true
```

---

## 7. Daily Operations & Graceful VM Shutdown

1. **Hot Reload Boundaries**:
   - Business code (`.go`, `.tsx`, `.astro`, `.md`, `.css`): saves on Windows sync via Mutagen in milliseconds and trigger instant `air` / Vite / Astro reloads.
   - Framework/env configs (`vite.config.*`, `astro.config.*`, `server/.env`): restart the target container once from `https://dockge.test`.
2. **Graceful VM Shutdown (Preventing DB/File Corruption)**:
   - Use `shutdown -h now` / `poweroff` inside Debian, or VMware **Power → Shut Down Guest**. `systemd` stops `docker.service` gracefully (`SIGTERM` → PostgreSQL WAL flush & container exit) before powering off.
   - For daily breaks, VMware **Suspend Guest** saves RAM in seconds and resumes all containers and Mutagen sessions immediately.
   - **Never** use **Power Off** or force-kill VMware while containers are running.
