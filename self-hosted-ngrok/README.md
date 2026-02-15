# frp + Dokploy: Self-Hosted ngrok Alternative

> Expose local dev servers to the internet via your own custom tunnel subdomain
> Fully isolated from Dokploy's existing domains and certificates.

## What is this?

This guide helps you create your own **ngrok-like tunnel service** using:
- **frp** (fast reverse proxy) - open-source alternative to ngrok
- **Dokploy** - your deployment platform
- **Cloudflare** - for DNS and SSL certificates

**Why use this?**
- ✅ **Free and unlimited** - no rate limits, no session timeouts
- ✅ **Custom domains** - `myapp.tunnel.yourdomain.com` instead of random URLs
- ✅ **Full control** - you own the infrastructure
- ✅ **Persistent** - same URLs every time (perfect for webhooks, OAuth callbacks)
- ✅ **Team-friendly** - share your server with teammates
- ✅ **Private** - your traffic stays on your server

**Use cases:**
- Share localhost with clients/teammates
- Test webhooks (Stripe, GitHub, etc.)
- Demo apps before deployment
- Mobile device testing
- OAuth callback development

---

## Prerequisites
- A server with Dokploy installed
- A domain name with DNS managed by Cloudflare
- Root/SSH access to your server

> 💡 **This guide prioritizes using Dokploy's UI** whenever possible. SSH alternatives are provided for advanced users, but most tasks can be done through the web interface!

---

## Configuration Variables

Before starting, gather these values (you'll need them throughout the guide):

| Variable | Example | Where to use |
|----------|---------|--------------|
| `YOUR_DOMAIN` | `example.com` | DNS, Traefik config, frps.toml, vite.config |
| `YOUR_SERVER_IP` | `1.2.3.4` | Cloudflare DNS |
| `YOUR_EMAIL` | `admin@example.com` | Traefik certificate config |
| `YOUR_SERVER_HOSTNAME` | `server.example.com` or use IP | frpc.toml (client config) |
| `AUTH_TOKEN` | Generate strong random string | frps.toml and frpc.toml (must match) |
| `DASHBOARD_PASSWORD` | Generate strong random string | frps.toml web server |
| `CONTAINER_NAME` | Found after deploy (step 4.7) | Traefik routing file |

**Generate secure tokens:**
```bash
# Linux/macOS - generate random tokens
openssl rand -base64 32
```

---

## Table of Contents

1. [DNS Configuration](#1-dns-cloudflare)
2. [Cloudflare API Token](#2-cloudflare-api-token)
3. [Configure Traefik](#3-configure-traefik-in-dokploy-ui)
4. [Deploy frps Server](#4-deploy-frps-via-dokploy-compose)
5. [Traefik Routing](#5-traefik-route-via-file-provider)
6. [Firewall Setup](#6-firewall)
7. [Client Setup](#7-client-setup-your-local-machine)
8. [Dev Server Config](#8-dev-server-configuration)
9. [Troubleshooting](#9-troubleshooting)

---

## How it works

```
Internet → myapp.tunnel.yourdomain.com
    ↓
Traefik (:443, wildcard SSL)
    ↓
frps container (:8080)
    ↓ (tunnel via port 7000)
frpc client (your machine)
    ↓
localhost:5173
```

---

## 1. DNS (Cloudflare)

Add one wildcard record pointing to your server:

```
*.tunnel.yourdomain.com  →  A  →  YOUR_SERVER_IP  (DNS only, gray cloud)
```

**Example:** `*.tunnel.example.com → A → 1.2.3.4`

> The subdomain `tunnel` is just a suggestion. Use any name, but keep it consistent throughout.

---

## 2. Cloudflare API Token

Needed for the wildcard SSL certificate (DNS challenge).

1. Cloudflare dashboard → **My Profile** → **API Tokens**
2. **Create Token** → template **"Edit zone DNS"**
3. Zone Resources → Include → Specific zone → `yourdomain.com`
4. Create Token → copy it

---

## 3. Configure Traefik in Dokploy UI

All done through Dokploy's dashboard — survives reboots and updates.

### 3.1 Add Cloudflare token

1. Go to **Web Server → Traefik → Environment** tab
2. Add a new line:

```
CF_DNS_API_TOKEN=your_cloudflare_token_here
```

3. Your environment should look like:

```
1  PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
2  CF_DNS_API_TOKEN=your_cloudflare_token_here
```

4. Click **Save**

### 3.2 Add DNS cert resolver to traefik.yml

**Option A: Via Dokploy UI (recommended)**

1. Go to **Web Server → Traefik → Traefik Config** tab
2. Edit the YAML directly in the browser

**Option B: Via SSH**

Edit `/etc/dokploy/traefik/traefik.yml`

---

Keep everything existing, add `letsencrypt-dns` below the existing `letsencrypt`:

```yaml
global:
  sendAnonymousUsage: false
providers:
  swarm:
    exposedByDefault: false
    watch: true
  docker:
    exposedByDefault: false
    watch: true
    network: dokploy-network
  file:
    directory: /etc/dokploy/traefik/dynamic
    watch: true
entryPoints:
  web:
    address: :80
  websecure:
    address: :443
    http3:
      advertisedPort: 443
    http:
      tls:
        certResolver: letsencrypt
api:
  insecure: true
certificatesResolvers:
  letsencrypt:                        # ← EXISTING, DON'T TOUCH
    acme:
      email: your-email@example.com
      storage: /etc/dokploy/traefik/dynamic/acme.json
      httpChallenge:
        entryPoint: web
  letsencrypt-dns:                    # ← NEW, for wildcard
    acme:
      email: your-email@example.com
      storage: /etc/dokploy/traefik/dynamic/acme-dns.json
      dnsChallenge:
        provider: cloudflare
        resolvers:
          - "1.1.1.1:53"
          - "8.8.8.8:53"
```

Save.

### 3.3 Restart Traefik

**Option A: Via Dokploy UI (easiest)**

1. Go to **Web Server → Traefik**
2. Click the **three dots menu (⋮)** or action menu
3. Select **Restart**

**Option B: Via SSH**

```bash
docker restart dokploy-traefik
```

**Verify it started correctly:**

Via Dokploy UI: Check the Traefik logs in the **Monitoring** or **Logs** tab.

Via SSH:
```bash
docker logs dokploy-traefik --tail 10
```

---

## 4. Deploy frps via Dokploy Compose

### 4.1 Create project

Dokploy dashboard → **Create Project** → name it `frp-tunnel`.

### 4.2 Create Compose service

Inside the project → **Create Service** → **Compose** → **Docker Compose** → **Raw**.

### 4.3 Paste docker-compose.yml

```yaml
services:
  frps:
    image: snowdreamtech/frps:latest
    restart: unless-stopped
    ports:
      - "7000:7000"
    volumes:
      - ../files/frps.toml:/etc/frp/frps.toml
    networks:
      - dokploy-network

networks:
  dokploy-network:
    external: true
```

### 4.4 Create frps.toml via Dokploy file mounts

Compose service → **Advanced** → **Volumes / Files** (or **Files** tab).

Create file `frps.toml`:

```toml
bindPort = 7000
vhostHTTPPort = 8080

auth.method = "token"
auth.token = "CHANGE_ME_TO_A_STRONG_SECRET"

subDomainHost = "tunnel.yourdomain.com"

webServer.addr = "0.0.0.0"
webServer.port = 7500
webServer.user = "admin"
webServer.password = "CHANGE_ME_DASHBOARD_PASS"
```

### 4.5 Add domain for frp dashboard (optional)

Compose service → **Domains** tab → **Add Domain**:
- Domain: `frp.tunnel.yourdomain.com`
- Container Port: `7500`
- HTTPS: enabled

### 4.6 Deploy

Hit **Deploy**. Check logs for successful startup.

### 4.7 Get the container name

**Option A: Via Dokploy UI**

1. Go to your **frp-tunnel** project
2. Click on the **frps** service
3. Go to the **General** or **Monitoring** tab
4. Look for the container name in the details (usually shown at the top)

**Option B: Via SSH**

```bash
docker ps | grep frps
```

Note the full name (e.g. `system-frp-zsykua-frps-1`). You'll need it for step 5.1.

> After redeploying, the container name may change - update `frp-tunnel.yml` if needed.

---

## 5. Traefik Route via File Provider

Dokploy Compose labels are unreliable for Traefik routing. The file provider method works perfectly — Traefik watches `/etc/dokploy/traefik/dynamic/` and hot-reloads changes.

### 5.1 Create the routing file

> ⚠️ **No UI option available** - This file must be created via terminal (Dokploy doesn't have a UI for Traefik dynamic configs).

**Via SSH:**

Connect to your server via SSH (or use Dokploy's integrated terminal: **Monitoring** → **Terminal**).

**Replace the placeholders before running:**
- `YOUR_DOMAIN` with your actual domain (e.g., `example.com`)
- `CONTAINER_NAME_HERE` with your actual container name from step 4.7

```bash
cat > /etc/dokploy/traefik/dynamic/frp-tunnel.yml << 'EOF'
http:
  routers:
    frp-tunnel:
      rule: "HostRegexp(`^[a-z0-9-]+\\.tunnel\\.YOUR_DOMAIN$`)"
      entryPoints:
        - websecure
      service: frp-tunnel
      tls:
        certResolver: letsencrypt-dns
        domains:
          - main: "tunnel.YOUR_DOMAIN"
            sans:
              - "*.tunnel.YOUR_DOMAIN"
      priority: 100

    frp-tunnel-http:
      rule: "HostRegexp(`^[a-z0-9-]+\\.tunnel\\.YOUR_DOMAIN$`)"
      entryPoints:
        - web
      service: frp-tunnel
      priority: 100

  services:
    frp-tunnel:
      loadBalancer:
        servers:
          - url: "http://CONTAINER_NAME_HERE:8080"
EOF
```

Then replace the placeholders (example):

```bash
# Replace YOUR_DOMAIN with your actual domain
sed -i 's|YOUR_DOMAIN|example.com|g' /etc/dokploy/traefik/dynamic/frp-tunnel.yml

# Replace CONTAINER_NAME_HERE with your actual container name
sed -i 's|CONTAINER_NAME_HERE|system-frp-zsykua-frps-1|' /etc/dokploy/traefik/dynamic/frp-tunnel.yml
```

**Traefik picks it up within seconds — no restart needed!**

Traefik watches the `/etc/dokploy/traefik/dynamic/` directory and automatically reloads when you add or modify files.

### 5.2 Verify network

Should be automatic if you used the docker-compose.yml from step 4.3.

If needed:
```bash
docker network connect dokploy-network $(docker ps -qf "ancestor=snowdreamtech/frps")
```

---

### 5.3 Managing frps

**Dokploy UI:** Go to frp-tunnel project → frps service → **⋮** menu

**SSH:**
```bash
docker restart $(docker ps -qf "ancestor=snowdreamtech/frps")  # restart
docker logs -f $(docker ps -qf "ancestor=snowdreamtech/frps")  # logs
```

---

## 6. Firewall

Open TCP port 7000 on your server (cloud provider firewall + ufw).

```bash
sudo ufw allow 7000/tcp
```

---

## 7. Client Setup (Your Local Machine)

### Install frpc

**macOS:**
```bash
brew install frp
```

**Linux:**
```bash
wget https://github.com/fatedier/frp/releases/download/v0.61.1/frp_0.61.1_linux_amd64.tar.gz
tar xzf frp_0.61.1_linux_amd64.tar.gz
cd frp_0.61.1_linux_amd64
```

### Create frpc.toml

```toml
serverAddr = "your-server-hostname-or-ip"
serverPort = 7000

auth.method = "token"
auth.token = "CHANGE_ME_TO_A_STRONG_SECRET"

[[proxies]]
name = "myapp"
type = "http"
localPort = 5173
subdomain = "myapp"
# → https://myapp.tunnel.yourdomain.com

[[proxies]]
name = "api"
type = "http"
localPort = 3000
subdomain = "api"
# → https://api.tunnel.yourdomain.com
```

> Use the same `auth.token` from `frps.toml` (step 4.4).

### Run

```bash
frpc -c frpc.toml
```

Wait for `start proxy success`, then open `https://myapp.tunnel.yourdomain.com`

---

## 8. Dev Server Configuration (SvelteKit + Bun)

**vite.config.ts:**
```ts
import { sveltekit } from '@sveltejs/kit/vite';
import { defineConfig } from 'vite';

export default defineConfig({
  plugins: [sveltekit()],
  server: {
    host: '0.0.0.0',
    port: 5173,
    allowedHosts: ['.yourdomain.com']
  }
});
```

**Run:**
```bash
bun run dev -- --host 0.0.0.0 --port 5173
```

> Both `vite.config.ts` settings AND the `--host` flag are required.

---

## 9. Troubleshooting

**Common issues:**

- **"powered by frp" page** → frpc not connected, restart `frpc -c frpc.toml`
- **"connection refused"** → Dev server not on `0.0.0.0`, use `--host` flag
- **"host not allowed" (Vite)** → Add `allowedHosts: ['.yourdomain.com']` to vite.config
- **502 Bad Gateway** → frps not on dokploy-network, reconnect it
- **TRAEFIK DEFAULT CERT** → Check CF token in Traefik Environment
- **Port 7000 blocked** → Check firewall (ufw, cloud provider)
- **After redeploy** → Update container name in `frp-tunnel.yml`

> Check logs: Dokploy UI → frps service → **Monitoring** → **Logs**
