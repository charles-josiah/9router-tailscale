<!--
title: Deploy 9Router AI Gateway Behind Tailscale — Private Docker Setup
description: Step-by-step guide to self-host the 9Router AI gateway on Oracle Cloud with a Tailscale sidecar container. Zero public ports, MagicDNS HTTPS, Docker Compose, env reference, usage, updates and rollback.
keywords: 9router, tailscale, ai gateway, llm router, docker compose, oracle cloud, oci, self-hosted, magicdns, private network, zero public ports, ai proxy, openai compatible
-->

keywords: 9router, tailscale, ai gateway, llm router, docker compose, oracle cloud, oci, self-hosted, magicdns, private network, zero public ports

# Deploy 9Router Behind Tailscale — Private AI Gateway (Docker Compose)

> **What you will have at the end of this guide:** a self-hosted AI gateway that aggregates multiple AI providers (Claude, Gemini, Ollama, OpenRouter, etc.) behind a single OpenAI-compatible endpoint, reachable over HTTPS **exclusively inside your private Tailscale network**. No public ports, no exposed dashboard, no leaked credentials — plus the day-2 procedures to back up, upgrade and roll back the service safely.

---

## Table of contents

1. [Concepts for newcomers](#1-concepts-for-newcomers)
2. [Architecture — why a Tailscale sidecar](#2-architecture--why-a-tailscale-sidecar)
3. [Directory layout — where everything lives](#3-directory-layout--where-everything-lives)
4. [Prerequisites](#4-prerequisites)
5. [Step 1 — Get the source](#5-step-1--get-the-source)
6. [Step 2 — Environment variables (`.env`)](#6-step-2--environment-variables-env)
7. [Step 3 — Tailscale serve config](#7-step-3--tailscale-serve-config)
8. [Step 4 — Docker Compose with Tailscale sidecar](#8-step-4--docker-compose-with-tailscale-sidecar)
9. [Step 5 — Start and verify](#9-step-5--start-and-verify)
10. [How to use the gateway](#10-how-to-use-the-gateway)
11. [Day-2 operations (maintenance)](#11-day-2-operations-maintenance)
12. [Updating 9Router (safe upgrade + rollback)](#12-updating-9router-safe-upgrade--rollback)
13. [Environment variable reference](#13-environment-variable-reference)
14. [Troubleshooting](#14-troubleshooting)
15. [Security notes](#15-security-notes)

---

## 1. Concepts for newcomers

If you are not familiar with these tools, read this section first. It is intentionally brief.

### What is 9Router?

9Router is a self-hosted AI gateway: a web dashboard where you store credentials for multiple AI providers (Claude, Gemini, Ollama, OpenRouter, Kiro, …), and an OpenAI-compatible API endpoint that routes requests to the right provider. Instead of configuring one API key per tool, every tool points to 9Router. It is an open-source project maintained at [github.com/decolua/9router](https://github.com/decolua/9router).

### What is Tailscale?

Tailscale is a private, encrypted network ("tailnet") built on WireGuard. Devices in your tailnet can reach each other by stable hostnames (`<host>.<tailnet>.ts.net`, "MagicDNS") regardless of where they are, without opening any port in your firewall or router. Access is controlled by tailnet membership, not by IP allowlists. This guide uses Tailscale as the **only** way to reach the gateway.

### What is Docker Compose?

Docker runs applications in isolated "containers". Compose is a file format (`docker-compose.yml`) that describes the containers of a service, their volumes (persistent storage) and their relationships. In this project, Compose runs two containers: the 9Router application and a Tailscale "sidecar" (a helper container that provides the network to the app).

---

## 2. Architecture — why a Tailscale sidecar

Instead of binding 9Router to a public port (`-p 20128:20128`), the app container **shares the network namespace** of a Tailscale container (`network_mode: "service:tailscale-9router"`). The Tailscale container is the only one that talks to the network; it terminates HTTPS with a **Let's Encrypt certificate issued automatically by Tailscale** and proxies traffic to `127.0.0.1:20128` inside the sidecar's network namespace.

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
| Dashboard & API exposed to the internet | Yes | **No** |
| OCI security list changes required | Yes (open port + restrict) | **None** |
| TLS certificate | Manual (or none) | **Automatic** (MagicDNS + Let's Encrypt) |
| Access control | IP allowlists, auth, vigilance | **Tailnet membership only** |
| URL | `http://IP:20128` (IP is dynamic) | `https://9router.<tailnet>.ts.net` (stable) |
| Split tunnel / browsing | N/A | Unaffected — only `*.ts.net` traffic goes through the tunnel |

> `tailscale serve` publishes the service on HTTPS at your MagicDNS name, **tailnet-only by default** (no funnel). Exposing it to the public internet would require a deliberate, explicit action (`tailscale funnel`). The default is private.

---

## 3. Directory layout — where everything lives

Understanding the file layout is essential before running any command. There are two trees to keep in mind: **this repository** (portable, version-controlled) and the **server deploy directory** (where the service actually runs).

### 3.1 This repository

```
9router-tailscale/
├── README.md                     ← this guide
├── docker-compose.yml            ← the Compose definition (Tailscale sidecar + 9Router)
├── config/
│   └── 9router-serve.json        ← Tailscale serve rules (HTTPS :443 → app)
├── .env.example                  ← template for your environment file (copy to .env)
└── .gitignore                    ← keeps .env and local artifacts out of git
```

### 3.2 Server deploy directory

On the server, the service lives in a directory that is a git checkout of the upstream 9Router source, with **this repository's files added on top**:

```
/docker/9router-docker/           ← deploy directory (git clone of decolua/9router)
├── docker-compose.yml            ← from this repository (replaces the upstream one)
├── config/
│   └── 9router-serve.json        ← from this repository
├── .env                          ← created by you from .env.example — NEVER commit
├── .env.example                  ← upstream template
├── package.json, app/, …         ← 9Router source code (upstream — do not edit)
```

### 3.3 Where to put and edit each file

| File | Location on the server | How to apply changes |
|---|---|---|
| `docker-compose.yml` | `/docker/9router-docker/docker-compose.yml` | Edit, then `sudo docker compose up -d` to recreate the affected containers |
| `config/9router-serve.json` | `/docker/9router-docker/config/9router-serve.json` | Edit, then `sudo docker compose restart tailscale-9router` |
| `.env` | `/docker/9router-docker/.env` | Edit, then `sudo docker compose up -d` (recreates the 9Router container with the new variables) |

### 3.4 Docker volumes (managed by Docker — do not touch directly)

Persistent data is stored in **named volumes**, not in files. They survive container recreations and upgrades:

| Volume | Contents |
|---|---|
| `<project>_9router-data` | Dashboard database, credentials, logs (`/app/data`) |
| `<project>_n9router-usage` | Usage records (`/root/.n9router`) |
| `<project>_ts-9router-state` | Tailscale node identity — **do not delete**: deleting it resets your MagicDNS name |

`<project>` is the name of the deploy directory (e.g. `9router-docker`). Confirm with `sudo docker volume ls`.

---

## 4. Prerequisites

- A Linux VM with Docker and Docker Compose (this guide was validated on **Oracle Cloud / OCI** — Oracle Linux 8, user `ubuntu` with passwordless `sudo`). Any VM with `/dev/net/tun` works.
- A **Tailscale account** (free tier: up to 100 devices, 3 users).
- A **Tailscale auth key** (`TS_AUTHKEY`) generated in the admin console → **Settings → Keys**. Use a one-time key, or a reusable key if you plan to recreate the node.
- Basic familiarity with a terminal and `sudo`.

---

## 5. Step 1 — Get the source

Clone the upstream 9Router source into your deploy directory (for example `/docker`):

```bash
sudo mkdir -p /docker
cd /docker
sudo git clone https://github.com/decolua/9router.git 9router-docker
cd 9router-docker
```

Then copy the three configuration files from this repository into the deploy directory:

```bash
# From a clone of THIS repo (or from a local copy of the files):
cp docker-compose.yml /docker/9router-docker/
cp config/9router-serve.json /docker/9router-docker/config/
```

**Expected outcome:** `/docker/9router-docker` contains the 9Router source, our `docker-compose.yml` and our `config/9router-serve.json`. The compose file from this repository replaces the upstream one; the 9Router source tree is left untouched.

---

## 6. Step 2 — Environment variables (`.env`)

Create `.env` in the deploy directory. It is read by Docker Compose on every `up` and never committed to git:

```bash
cd /docker/9router-docker
sudo cp .env.example .env     # then edit with your editor of choice
```

Minimum set (values shown are placeholders):

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

Generate a strong `JWT_SECRET`:

```bash
openssl rand -hex 32
```

See the full [environment variable reference](#13-environment-variable-reference) for every variable.

---

## 7. Step 3 — Tailscale serve config

`config/9router-serve.json` tells the Tailscale container which HTTPS endpoint to publish and where to forward traffic. This file is read via the `TS_SERVE_CONFIG` environment variable:

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

`${TS_CERT_DOMAIN}` is expanded by Tailscale at runtime to your MagicDNS name (e.g. `9router.<tailnet>.ts.net`). The proxy target is `127.0.0.1:20128` **inside the sidecar's network namespace** — this is why 9Router must share that namespace (`network_mode`, see Step 4). `AllowFunnel: false` keeps the service private to the tailnet.

---

## 8. Step 4 — Docker Compose with Tailscale sidecar

`docker-compose.yml` is the core of this setup: two services, one shared network namespace.

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

- `network_mode: "service:tailscale-9router"` — 9Router has **no network interface of its own**; it listens on `127.0.0.1:20128` inside the Tailscale container's namespace. Nothing is reachable from outside except what Tailscale explicitly exposes.
- `TS_SERVE_CONFIG` — instructs Tailscale to serve HTTPS on `:443` and proxy to the application.
- `TS_STATE_DIR=/var/lib/tailscale` on a named volume — the node identity persists across container recreations; rebuilding the application does **not** change the MagicDNS name or IP.
- `depends_on` — 9Router only starts after the sidecar is up.
- Named volumes `9router-data` and `n9router-usage` — dashboard database, credentials and usage records survive every rebuild and upgrade.

> **Note:** unlike the upstream compose, there is **no `ports:` section** here, by design. The service is only reachable through the tailnet.

---

## 9. Step 5 — Start and verify

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

Open `https://9router.<tailnet>.ts.net` in your browser **while connected to the tailnet**. Log in with the `INITIAL_PASSWORD` you set, then change it from the dashboard.

**Expected outcome:** the `tailscale serve status` output lists your MagicDNS name with a `proxy http://127.0.0.1:20128` handler, and the `curl` command returns the list of models configured in your 9Router dashboard.

---

## 10. How to use the gateway

### 10.1 Web dashboard

Store provider credentials (Claude, Gemini, Ollama, OpenRouter, …), configure models and inspect usage from the dashboard at the root URL of your MagicDNS endpoint.

### 10.2 OpenAI-compatible API

Any client that speaks the OpenAI protocol can point to your private endpoint:

```
Base URL: https://9router.<tailnet>.ts.net/v1
API key : the key issued by the dashboard (or REQUIRE_API_KEY=false for tailnet-only use)
```

```bash
curl https://9router.<tailnet>.ts.net/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <api-key>" \
  -d '{"model":"modelo-1","messages":[{"role":"user","content":"Hello"}]}'
```

### 10.3 OpenCode / Cursor / OpenWork

The model identifiers below (`modelo-1`, `modelo-2`, `modelo-3`) are placeholders for the models **you configured in the 9Router dashboard** — replace them with the actual model IDs shown in the Models page of your dashboard. The names `IA 1`, `IA 2` and `IA 3` are just display labels that will appear in the model picker.

> Claude Code is not covered here: it does not support custom OpenAI-compatible providers reliably.

Example provider block for OpenCode (`opencode.jsonc`), also usable in Cursor and OpenWork:

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
        "modelo-1": { "name": "IA 1" },
        "modelo-2": { "name": "IA 2" },
        "modelo-3": { "name": "IA 3" }
      }
    }
  }
}
```

> Remember: `*.ts.net` names only resolve while Tailscale is running on the client machine.

### 10.4 CLI

```bash
9r --help                     # CLI shipped with 9Router
9r login https://9router.<tailnet>.ts.net
```

---

## 11. Day-2 operations (maintenance)

```bash
sudo docker compose logs -f 9router      # application logs
sudo docker exec ts-9router tailscale status
sudo docker compose restart 9router
sudo docker compose down && sudo docker compose up -d   # clean recreate (data lives in volumes)
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

Volume names follow the `<project>_<volume>` convention — check with `sudo docker volume ls` and adjust if your deploy directory is named differently.

---

## 12. Updating 9Router (safe upgrade + rollback)

The deploy directory is a git checkout of the upstream repository, so upgrades are `git pull` + rebuild. Your local `docker-compose.yml` and `config/` are local-only changes and are never overwritten by the pull.

```bash
cd /docker/9router-docker

# 1. Backup + rollback tag (see backup snippet in section 11)
sudo git -c safe.directory=/docker/9router-docker tag -f backup-before-update HEAD

# 2. Pull upstream (fast-forward only; your compose/config are not touched)
sudo git -c safe.directory=/docker/9router-docker pull --ff-only origin master

# 3. Rebuild (the old container keeps running during the build)
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

> The `-c safe.directory=...` flag is only required when the deploy directory is owned by root and you operate as a non-root sudo user. It is harmless and local to the server.

---

## 13. Environment variable reference

### 9Router core (required)

| Variable | Purpose |
|---|---|
| `JWT_SECRET` | Signs session and API tokens. Generate with `openssl rand -hex 32` |
| `INITIAL_PASSWORD` | Password for the first dashboard login |
| `DATA_DIR` | Where the SQLite database, certificates and logs live (container: `/app/data`) |

### Recommended

| Variable | Purpose | Default |
|---|---|---|
| `PORT` | HTTP port inside the container | `20128` |
| `NODE_ENV` | `production` for deployments | `production` |
| `HOSTNAME` | Listen address (keep `0.0.0.0` inside the container) | `0.0.0.0` |
| `API_KEY_SECRET` | Secrets material for the endpoint proxy API keys | — |
| `MACHINE_ID_SALT` | Salt for machine-id usage attribution | — |
| `ENABLE_REQUEST_LOGS` | Log every proxied request | `false` |
| `OBSERVABILITY_ENABLED` | Telemetry/metrics | `true` |
| `AUTH_COOKIE_SECURE` | Secure cookie flag (HTTPS, so `true`) | `false` |
| `REQUIRE_API_KEY` | Require an API key on `/v1` endpoints | `false` |
| `BASE_URL` / `CLOUD_URL` | Public URLs for cloud sync jobs; the `NEXT_PUBLIC_*` variants are the backward-compatible aliases | — |

### Tailscale sidecar

| Variable | Purpose |
|---|---|
| `TS_AUTHKEY` | Tailscale auth key (required at first boot) |
| `TS_STATE_DIR` | Where the node identity persists (named volume) |
| `TS_USERSPACE` | `false` → kernel networking with `/dev/net/tun` |
| `TS_SERVE_CONFIG` | Path of the serve JSON inside the container |
| `TS_EXTRA_ARGS` | Optional `tailscale up` flags (tags, exit node, etc.) |

### Optional

| Variable | Purpose |
|---|---|
| `HTTP_PROXY`/`HTTPS_PROXY`/`ALL_PROXY`/`NO_PROXY` | Outbound proxy for upstream provider calls (lowercase variants also supported) |
| `SEARXNG_URL` | Self-hosted SearXNG for the built-in web-search provider |

---

## 14. Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `9router.<tailnet>.ts.net` does not resolve | Tailscale is down on the client or the node is not registered — check `tailscale status` / `tailscale up` |
| `curl: 401` on `/v1/models` | Wrong or missing API key. `REQUIRE_API_KEY=false` skips the check (safe on a tailnet-only service) |
| `421 Misdirected Request` | DNS rebinding protection on the dashboard origin — check that `NEXT_PUBLIC_BASE_URL` matches the MagicDNS name |
| Model answers `429 out_of_credits` | The upstream provider account is out of credits (e.g. Claude) — top up on the provider side, not here |
| Provider returns `403` (e.g. Gemini) | OAuth/credential expired in the dashboard — re-authenticate the provider connection |
| Model returns `410 retired` (e.g. `ollama/glm-5`) | The upstream provider retired the model — switch to a current one |
| `permission denied` on the git/deploy directory | Directory owned by root: use `sudo` and/or `git -c safe.directory=...` |
| Container recreates but the MagicDNS name changes | The `ts-9router-state` volume was deleted — keep it |

---

## 15. Security notes

- **Zero public ports.** No `ports:` mapping, no OCI security-list rule for 20128. SSH is the only public entry point.
- **Tailnet-only HTTPS.** `AllowFunnel: false` — reachable exclusively by devices in your Tailscale network.
- **Never commit `.env`.** It is excluded by `.gitignore`. This repository ships only a `.env.example` template.
- **Rotate secrets.** `JWT_SECRET`, `TS_AUTHKEY` and dashboard API keys should be rotated periodically; a leaked `TS_AUTHKEY` grants node admission to your tailnet.
- **Update regularly.** 9Router ships frequently; the upgrade and rollback procedure in section 12 makes it safe.

---

## About

Maintained by Charles J R Alandt. This repository documents a validated, production-shaped deployment of 9Router behind Tailscale, and is intended as a reusable reference for private AI gateway setups.

:wq!