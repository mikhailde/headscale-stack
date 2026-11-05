# Headscale Stack

[![Docker Compose](https://img.shields.io/badge/docker-compose-blue.svg)](https://docs.docker.com/compose/)
[![Headscale](https://img.shields.io/badge/headscale-v0.27.1-blue)](https://github.com/juanfont/headscale)
[![Headplane](https://img.shields.io/badge/headplane-latest-orange)](https://github.com/tale/headplane)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

Production-ready, declarative **Headscale** + **Headplane** stack with optional **Caddy** reverse proxy and automatic HTTPS.

---

## ⚡ Features

- **100% Declarative & GitOps-Ready**: Zero host-level bash scripts or runtime string replacements. The entire stack is configured via a single `.env` file.
- **Modern Web Interface**: Integrated with [Headplane](https://github.com/tale/headplane) for visual node management, DNS configuration, and ACL rules.
- **Flexible Reverse Proxy**:
  - **Built-in Caddy**: Automatic Let's Encrypt TLS and HTTP Basic Auth out of the box (`COMPOSE_PROFILES=caddy`).
  - **External Proxy Ready**: Seamlessly plug into your existing Traefik, Nginx, or Nginx Proxy Manager.
- **Secure Persistence**: Databases and encryption keys are stored inside isolated Docker named volumes.
- **Official Compliance**: Implements official Headscale recommendations (WebSocket upgrade support, real client IP passing, and `/generate_204` captive portal handling).

---

## 📁 Repository Structure

```text
headscale-stack/
├── config/
│   ├── Caddyfile             # Proxy configuration with automatic env interpolation
│   ├── headplane.yaml        # Headplane web UI settings and Docker integration
│   └── headscale.yaml        # Modern Headscale server configuration
├── .env.example              # Single source of truth for all environment variables
├── .gitignore                # Protects against leaking .env, keys, and databases
├── compose.yaml              # Core multi-container Docker Compose manifest
├── LICENSE                   # MIT License
└── README.md                 # Complete documentation and client recipes
```

---

## 🚀 Quick Start

### 1. Clone the repository
```bash
git clone https://github.com/mikhailde/headscale-stack.git
cd headscale-stack
```

### 2. Configure environment
```bash
cp .env.example .env
```
Edit `.env` and set:
- `DOMAIN`: Your public domain (e.g. `hs.example.com`).
- `HEADPLANE_COOKIE_SECRET`: Generate with `openssl rand -hex 16`.
- `ADMIN_PASSWORD_HASH`: Generate with:
  ```bash
  docker run --rm caddy:latest caddy hash-password --plaintext "your_secure_password"
  ```

> ⚠️ **Cloudflare Notice**: If using Cloudflare DNS, set the proxy status to **DNS-only (grey cloud)**. Cloudflare Proxy blocks Tailscale WebSocket POST requests.

### 3. Launch the stack
```bash
docker compose up -d
```

### 4. Create an API Key and Access the Web UI
Generate an API key for Headplane:
```bash
docker compose exec headscale headscale apikeys create --expiration 90d
```
Open **`https://<your-domain>/admin`** in your browser, enter your Basic Auth credentials, and log into Headplane with the generated API key.

---

## 🌐 Reverse Proxy Scenarios

### Option A: Built-in Caddy (Default)
In `.env`, keep `COMPOSE_PROFILES=caddy`.
Caddy handles automatic HTTPS and Basic Auth on ports `80` and `443`.

### Option B: External Reverse Proxy on the Host (Nginx / Caddy on host)
1. In `.env`, set `COMPOSE_PROFILES=` (empty).
2. Headscale listens on `127.0.0.1:8080`, and Headplane listens on `127.0.0.1:3000`.
3. Proxy rules:
   - Route `/admin*` -> `http://127.0.0.1:3000`
   - Route `/` -> `http://127.0.0.1:8080` (Pass `True-Client-IP` and enable WebSockets).

### Option C: External Reverse Proxy in Docker (NPM / Traefik)
Connect your proxy container to the `headscale-net` Docker network:
```bash
docker network connect headscale-net <your-proxy-container>
```
Route requests:
- `/admin*` -> `http://headplane:3000`
- `/*` -> `http://headscale:8080`

---

## 💻 Client Configurations (Docker Compose)

Deploy clients on other machines using standard Docker Compose.

### 1. Tailscale Exit Node (Full VPN Gateway `0.0.0.0/0`)
Run this on a remote server/VPS to route all traffic through it:

```yaml
services:
  tailscale-exit:
    image: tailscale/tailscale:stable
    container_name: tailscale-exit
    restart: unless-stopped
    devices:
      - /dev/net/tun:/dev/net/tun
    cap_add:
      - NET_ADMIN
      - NET_RAW
    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv6.conf.all.forwarding=1
    volumes:
      - tailscale-state:/var/lib/tailscale
    environment:
      - TS_AUTHKEY=replace_with_preauth_key
      - TS_HOSTNAME=exit-node
      - TS_USERSPACE=false
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_ROUTES=0.0.0.0/0,::/0
      - TS_EXTRA_ARGS=--login-server=https://<your-domain> --advertise-exit-node
volumes:
  tailscale-state:
```

### 2. Tailscale Server Node (Host Network Client)
Run this on any server to securely connect it to your Tailnet:

```yaml
services:
  tailscale:
    image: tailscale/tailscale:stable
    container_name: tailscale-server
    restart: unless-stopped
    network_mode: host
    devices:
      - /dev/net/tun:/dev/net/tun
    cap_add:
      - NET_ADMIN
      - NET_RAW
    volumes:
      - tailscale-state:/var/lib/tailscale
    environment:
      - TS_AUTHKEY=replace_with_preauth_key
      - TS_HOSTNAME=server-main
      - TS_USERSPACE=false
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_EXTRA_ARGS=--login-server=https://<your-domain> --reset
volumes:
  tailscale-state:
```

---

## 🛠️ Headscale CLI Shortcuts

Run Headscale management commands directly through Docker:

```bash
# User Management
docker compose exec headscale headscale users create <username>
docker compose exec headscale headscale users list

# Pre-Auth Keys (for joining nodes automatically)
docker compose exec headscale headscale preauthkeys create --user <username> --reusable --expiration 24h

# Node (Machine) Management
docker compose exec headscale headscale nodes list
docker compose exec headscale headscale nodes expire --identifier <node-id>
docker compose exec headscale headscale nodes delete --identifier <node-id>

# Generate new Headplane API key
docker compose exec headscale headscale apikeys create --expiration 90d
```

---

## 💾 Backup & Restore

All critical state is stored in named volumes:
- `headscale-data` (SQLite DB and private encryption keys)
- `headplane-data` (Web sessions and UI persistence)

**Backup to archive:**
```bash
docker run --rm -v headscale-data:/data -v $(pwd):/backup alpine tar czf /backup/headscale-backup.tar.gz -C /data .
```

---

## 📄 License

This project is open-source software licensed under the [MIT License](LICENSE).
