<!--
title: Deploy 9Router AI Gateway Behind Tailscale — Private Docker Setup
description: Step-by-step guide to self-host the 9Router AI gateway on Oracle Cloud with a Tailscale sidecar container. Zero public ports, MagicDNS HTTPS, Docker Compose, env reference, usage, updates and rollback.
keywords: 9router, tailscale, ai gateway, llm router, docker compose, oracle cloud, oci, self-hosted, magicdns, private network, zero public ports, ai proxy, openai compatible
-->

keywords: 9router, tailscale, ai gateway, llm router, docker compose, oracle cloud, oci, self-hosted, magicdns, private network, zero public ports

# Deploy 9Router Behind Tailscale — Private AI Gateway (Docker Compose)

> **Why this guide?** 9Router is an AI gateway/router that aggregates multiple AI providers (Claude, Gemini, Ollama, OpenRouter, Kiro, etc.) behind a single OpenAI-compatible endpoint, with a web dashboard for credentials, models, usage and routing. The official quick start publishes port `20128` to the public internet. This guide shows the **private-first** deployment: 9Router runs in Docker and is exposed **only inside your Tailscale tailnet** via a sidecar container — no public ports, no exposed dashboard, no leaked credentials.

---

## Table of contents

1. [Architecture — why Tailscale sidecar](#architecture--why-tailscale-sidecar)
2. [Prerequisites](#prerequisites)
3. [Step 1 — Get the source](#step-1--get-the-source)
4. [Step 2 — Environment variables (`.env`)](#step-2--environment-variables-env)
5. [Step 3 — Tailscale serve config](#step-3--tailscale-serve-config)
6. [Step 4 — Docker Compose with Tailscale sidecar](#step-4--docker-compose-with-tailscale-sidecar)
7. [Step 5 — Start and verify](#step-5--start-and-verify)
8. [How to use the gateway](#how-to-use-the-gateway)
9. [Day-2 operations (maintenance)](#day-2-operations-maintenance)
10. [Updating 9Router (safe upgrade + rollback)](#updating-9router-safe-upgrade--rollback)
11. [Environment variable reference](#environment-variable-reference)
12. [Troubleshooting](#troubleshooting)
13. [Security notes](#security-notes)

---

## Architecture — why Tailscale sidecar

Instead of binding 9Router to a public port (`-p 20128:20128`), the app container **shares the network namespace** of a Tailscale container (`network_mode: "service:tailscale-9router"`). The Tailscale container is the only one that talks to the wire; it terminates HTTPS with a **Let's Encrypt certificate issued automatically by Tailscale** and proxies traffic to `127.0.0.1:20128` inside the sidecar network namespace.

```
                          Tailscale tailnet (private, encrypted)
┌─────────────────┐             ┌──────────────────────────────────────────────┐
│  Your laptop    │  WireGuard   │  Oracle Cloud VM (OCI)                        │
│  (tailscale up) │◄────────────►│  ts-9router  (tailscale/tailscale sidecar)    │
│  macOS / Linux  │  100.x.x.x   │    │                                         │
└─────────────────┘              │    │  HTTPS :443 → proxy → http://127.0.0.1:20128
                                 │    ▼                                         │
                                 │  9router (shares network namespace)           │
                                 └──────────────────────────────────────────────┘
```

**Why this approach?**

| Consideration | Public port (`-p 20128`) | Tailscale sidecar |
|---|---|---|
| Dashboard & API exposed to internet | ❌ Yes | ✅ No |
| OCI security list changes needed | Yes (open port + restrict) | **None** |
| TLS certificate | Manual (or none) | **Automatic** (MagicDNS + Let's Encrypt) |
| Access control | IP allowlists, auth, luck | **Tailnet membership only** |
| URL | `http://IP:20128` (IP is dynamic) | `https://9router.<tailnet>.ts.net` (stable) |
| Split tunnel / browsing | N/A | Works normally — only `*.ts.net` traffic goes through the tunnel |

> `tailscale serve` publishes the service on HTTPS at your MagicDNS name, **tailnet-only by default** (no funnel). If you ever want it reachable by the public internet, that is a deliberate, explicit action (`tailscale funnel`) — the default is private.

---

## Prerequisites

- A Linux VM with Docker + Docker Compose (this guide uses **Oracle Cloud / OCI** — Oracle Linux 8, `ubuntu` user with passwordless `sudo`)
- A **Tailscale account** (free tier: up to 100 devices, 3 users)
- A **Tailscale auth key** (`TS_AUTHKEY`) generated in the admin console → **Settings → Keys** (use a one-time key or an ephemeral/reusable key as you prefer)
- `/dev/net/tun` available on the host (standard on OCI VMs)
- 9Router source from GitHub: `https://github.com/decolua/9router`

---

## Step 1 — Get the source

```bash
cd /docker                      # or any deploy dir you own
sudo git clone https://github.com/decolua/9router.git 9router-docker
cd 9router-docker
```

The compose file in this repo replaces the upstream one; keep the rest of the source tree intact.

---

## Step 2 — Environment variables (`.env`)

Create `.env` in the deploy directory (never commit it). Values shown are placeholders:

```bash
# --- 9Router core ---
JWT_SECRET=change-me-to-a-long-random-secret
INITIAL_PASSWORD=change-me                      # first login password for the dashboard
NODE_ENV=production
PORT=20128
HOSTNAME=0.0.0.0
NEXT_PUBLIC_BASE_URL=http://9router:20128      # container-internal URL
NEXT_PUBLIC_CLOUD_URL=https://9router.com
ENABLE_REQUEST_LOGS=true
AUTH_COOKIE_SECURE=true
REQUIRE_API_KEY=false

# --- Tailscale sidecar ---
TS_AUTHKEY=tskey-auth-xxxxxxxx-xxxxxxxxxxxxxxxxx
TS_EXTRA_ARGS=--advertise-tags=tag:server       # optional
```

Generate strong values:

```bash
openssl rand -hex 32      # JWT_SECRET
```

See the full [environment variable reference](#environment-variable-reference) below.

---

## Step 3 — Tailscale serve config

Create `config/9router-serve.json` — this is read by the Tailscale container via `TS_SERVE_CONFIG`:

```json
{
  "TCP": {
    "443": { "HTTPS": true }
  },
  "Web": {
    "${TS_CERT_DOMAIN}:443": {
      "Handlers": {
        "/": { "Proxy": "http://127.0.0.1:20128" }
      }
    }
  },
  "AllowFunnel": {
    "${TS_CERT_DOMAIN}:443": false
  }
}
```

`${TS_CERT_DOMAIN}` is expanded by Tailscale at runtime to your MagicDNS name (e.g. `9router.<tailnet>.ts.net`). The proxy target is `127.0.0.1:20128` **inside the sidecar namespace** — that is why 9Router must share it (`network_mode`).

---

## Step 4 — Docker Compose with Tailscale sidecar

`docker-compose.yml` — this is the key file. Two services, one network namespace:

```yaml
services:
  tailscale-9router:
    image: tailscale/tailscale:latest
    container_name: ts-9router
    hostname: 9router
    environment:
      - TS_AUTHKEY=${TS_AUTHKEY}
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_USERSPACE=false
      - TS_SERVE_CONFIG=/config/9router-serve.json
      - TS_EXTRA_ARGS=${TS_EXTRA_ARGS:-}
    volumes:
      - ts-9router-state:/var/lib/tailscale
      - ./config:/config
    devices:
      - /dev/net/tun:/dev/net/tun
    cap_add:
      - NET_ADMIN
      - NET_RAW
    restart: unless-stopped

  9router:
    build:
      context: .
    container_name: 9router
    network_mode: "service:tailscale-9router"
    depends_on:
      - tailscale-9router
    volumes:
      - 9router-data:/app/data
      - n9router-usage:/root/.n9router
    environment:
      - PORT=20128
      - HOSTNAME=0.0.0.0
      - NODE_ENV=production
      - JWT_SECRET=${JWT_SECRET}
      - INITIAL_PASSWORD=${INITIAL_PASSWORD}
      - NEXT_PUBLIC_BASE_URL=${NEXT_PUBLIC_BASE_URL:-http://9router:20128}
      - NEXT_PUBLIC_CLOUD_URL=https://9router.com
      - DATA_DIR=/app/data
      - ENABLE_REQUEST_LOGS=true
      - AUTH_COOKIE_SECURE=true
      - REQUIRE_API_KEY=false
    restart: unless-stopped

volumes:
  9router-data:
  n9router-usage:
  ts-9router-state:
```

**Why each piece matters:**

- `network_mode: "service:tailscale-9router"` → 9Router has **no interface of its own**; it listens on `127.0.0.1:20128` inside the Tailscale container's namespace. Nothing is reachable from the outside except what Tailscale exposes.
- `TS_SERVE_CONFIG` → tells Tailscale to serve HTTPS on `:443` and proxy to the app.
- `TS_STATE_DIR=/var/lib/tailscale` on a named volume → the node identity persists across container recreations (important: rebuilding the app container does **not** reset the MagicDNS name/IP).
- `depends_on` → 9Router only starts after the sidecar is up.
- Named volumes `9router-data` and `n9router-usage` → dashboard DB, credentials and usage survive every rebuild/update.

> **Note:** unlike the upstream compose, there is **no `ports:` section** on purpose. The service is only reachable through the tailnet.

---

## Step 5 — Start and verify

```bash
sudo docker compose up -d
sudo docker compose ps

# Tailscale node registered?
sudo docker exec ts-9router tailscale status

# HTTPS endpoint published?
sudo docker exec ts-9router tailscale serve status
# https://9router.<tailnet>.ts.net (tailnet only)
# |-- / proxy http://127.0.0.1:20128

# Dashboard + API healthy?
curl -s https://9router.<tailnet>.ts.net/v1/models \
  -H "Authorization: Bearer <your-api-key>"
```

Open `https://9router.<tailnet>.ts.net` in your browser (you must be connected to the tailnet). Log in with `INITIAL_PASSWORD`, then change it.

---

## How to use the gateway

### 1. Web dashboard

Configure provider credentials (Claude, Gemini, Ollama, OpenRouter…), check usage and routing at the dashboard root URL.

### 2. OpenAI-compatible API

Every consumer that speaks the OpenAI protocol can point to your private endpoint:

```
Base URL: https://9router.<tailnet>.ts.net/v1
API key : the key issued by the dashboard (or REQUIRE_API_KEY=false for LAN/tailnet use)
```

```bash
curl https://9router.<tailnet>.ts.net/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <api-key>" \
  -d '{"model":"claude-top","messages":[{"role":"user","content":"Hello"}]}'
```

### 3. OpenCode / Cursor / Claude Code

Example OpenCode provider block (`opencode.jsonc`):

```jsonc
{
  "provider": {
    "9router": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "9router",
      "options": {
        "baseURL": "https://9router.<tailnet>.ts.net/v1",
        "apiKey": "<api-key>"
      },
      "models": {
        "claude-free": { "name": "Claude Free" },
        "claude-top":  { "name": "Claude Top" },
        "claude-kim":  { "name": "Claude Kim" }
      }
    }
  }
}
```

> Remember: `*.ts.net` only resolves while Tailscale is up on the client.

### 4. CLI

```bash
9r --help                     # CLI shipped with 9Router
9r login https://9router.<tailnet>.ts.net
```

---

## Day-2 operations (maintenance)

```bash
sudo docker compose logs -f 9router      # app logs
sudo docker exec ts-9router tailscale status
sudo docker compose restart 9router
sudo docker compose down && sudo docker compose up -d   # clean recreate (data is in volumes)
```

**Backup (before any major change):**

```bash
BK=/docker/data/backups/9router-$(date +%Y%m%d-%H%M)
sudo mkdir -p "$BK"
sudo docker run --rm -v 9router-docker_9router-data:/data:ro -v "$BK":/backup \
  alpine tar czf /backup/9router-data.tar.gz -C /data .
sudo docker run --rm -v 9router-docker_n9router-usage:/data:ro -v "$BK":/backup \
  alpine tar czf /backup/n9router-usage.tar.gz -C /data .
sudo cp -r config "$BK"/config
sudo cp docker-compose.yml .env "$BK"/
```

Volume names follow `<project>_<volume>` — check with `sudo docker volume ls` and adjust if your project dir is named differently.

---

## Updating 9Router (safe upgrade + rollback)

The deploy dir is a git checkout of the upstream repo, so upgrades are `git pull` + rebuild. Your local `docker-compose.yml` and `config/` are **local-only changes** and must be preserved.

```bash
cd /docker/9router-docker

# 1. Backup + rollback tag (see backup snippet above)
sudo git -c safe.directory=/docker/9router-docker tag -f backup-before-update HEAD

# 2. Pull upstream (fast-forward only; your compose/config are not touched)
sudo git -c safe.directory=/docker/9router-docker pull --ff-only origin master

# 3. Rebuild (old container keeps running during the build)
sudo docker compose build 9router

# 4. Recreate (seconds of downtime)
sudo docker compose up -d

# 5. Verify
sudo docker ps | grep 9router
curl -s https://9router.<tailnet>.ts.net/v1/models -H "Authorization: Bearer <api-key>"
```

**Rollback:**

```bash
cd /docker/9router-docker
sudo git -c safe.directory=/docker/9router-docker checkout backup-before-update
sudo docker compose build 9router
sudo docker compose up -d
```

> The `-c safe.directory=...` flag is only needed when the deploy dir is owned by root and you operate as a non-root sudo user — harmless and local-only.

---

## Environment variable reference

### 9Router core (required)

| Variable | Purpose |
|---|---|
| `JWT_SECRET` | Signs session/API tokens. Generate with `openssl rand -hex 32` |
| `INITIAL_PASSWORD` | Password for the first dashboard login |
| `DATA_DIR` | Where SQLite DB + certs/logs live (container: `/app/data`) |

### Recommended

| Variable | Purpose | Default |
|---|---|---|
| `PORT` | HTTP port inside the container | `20128` |
| `NODE_ENV` | `production` for deployment | `production` |
| `HOSTNAME` | Listen address (keep `0.0.0.0` in container) | `0.0.0.0` |
| `API_KEY_SECRET` | Secrets API-key material for endpoint proxy | — |
| `MACHINE_ID_SALT` | Machine-id salt for usage attribution | — |
| `ENABLE_REQUEST_LOGS` | Log every proxied request | `false` |
| `OBSERVABILITY_ENABLED` | Telemetry/metrics | `true` |
| `AUTH_COOKIE_SECURE` | Secure cookie flag (HTTPS, so `true`) | `false` |
| `REQUIRE_API_KEY` | Require API key on `/v1` endpoints | `false` |
| `BASE_URL` / `CLOUD_URL` | Public URLs for cloud sync jobs; `NEXT_PUBLIC_*` variants are the backward-compatible aliases | — |

### Tailscale sidecar

| Variable | Purpose |
|---|---|
| `TS_AUTHKEY` | Tailscale auth key (required at first boot) |
| `TS_STATE_DIR` | Where node identity persists (named volume) |
| `TS_USERSPACE` | `false` → kernel networking with `/dev/net/tun` |
| `TS_SERVE_CONFIG` | Path to the serve JSON inside the container |
| `TS_EXTRA_ARGS` | Optional `tailscale up` flags (tags, exit node, etc.) |

### Optional

| Variable | Purpose |
|---|---|
| `HTTP_PROXY`/`HTTPS_PROXY`/`ALL_PROXY`/`NO_PROXY` | Outbound proxy for upstream provider calls (lowercase variants also supported) |
| `SEARXNG_URL` | Self-hosted SearXNG for the built-in web-search provider |

---

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `9router.<tailnet>.ts.net` does not resolve | Tailscale is down on the client or node not registered — `tailscale status` / `tailscale up` |
| `curl: 401` on `/v1/models` | Wrong/missing API key. `REQUIRE_API_KEY=false` skips the check (tailnet-only is still safe) |
| `421 Misdirected Request` | DNS rebinding protection on the dashboard origin — check `NEXT_PUBLIC_BASE_URL` matches the MagicDNS name |
| Model answers `429 out_of_credits` | Upstream provider account is out of credits (Claude) — top up on the provider side, not here |
| Provider `403` (e.g. Gemini) | OAuth/credential expired in the dashboard — re-authenticate the provider connection |
| Model `410 retired` (e.g. `ollama/glm-5`) | Upstream provider retired the model — switch to a current one |
| `permission denied` on git/deploy dir | Dir owned by root: use `sudo` and/or `git -c safe.directory=...` |
| Container recreates but MagicDNS name changes | `TS_STATE_DIR` volume deleted — keep `ts-9router-state` |

---

## Security notes

- **Zero public ports.** No `ports:` mapping, no OCI security-list rule for 20128. SSH is the only public entry.
- **Tailnet-only HTTPS.** `AllowFunnel: false` — reachable exclusively by devices in your Tailscale network.
- **Never commit `.env`.** Add it to `.gitignore` (already present upstream). This repo ships `.env.example` only.
- **Rotate secrets.** `JWT_SECRET`, `TS_AUTHKEY` and dashboard API keys should be rotated periodically; a leaked `TS_AUTHKEY` grants node admission to your tailnet.
- **Update regularly.** 9Router moves fast (multiple releases/week); the upgrade + rollback routine above makes it safe.

---

*Written as part of the Exímio IT / Charles J R Alandt infrastructure lab. Share responsibly: this setup is a great fit for homelabs, consulting clients and anyone who wants a private AI gateway.*

:wq!