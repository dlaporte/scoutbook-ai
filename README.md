# Scoutbook AI

Self-hosted deployment for Scoutbook AI — an LLM-powered chat interface that gives Scoutmasters natural-language access to their Scoutbook data.

This repo orchestrates all three services with a single `docker compose up`:

| Service | Description | Image |
|---|---|---|
| **app** | Web chat interface | `ghcr.io/dlaporte/scoutbook-ai-app` |
| **mcp** | MCP server — exposes 135+ Scoutbook API tools to the LLM | `ghcr.io/dlaporte/scoutbook-ai-mcp` |
| **sandbox** | Isolated code execution for generating PDFs, charts, and spreadsheets | `ghcr.io/dlaporte/scoutbook-ai-sandbox` |

For details on the individual components, see [scoutbook-ai-app](https://github.com/dlaporte/scoutbook-ai-app) and [scoutbook-ai-mcp](https://github.com/dlaporte/scoutbook-ai-mcp).

---

## Quick Start (Local)

```bash
git clone https://github.com/dlaporte/scoutbook-ai.git
cd scoutbook-ai
cp .env.example .env
```

Edit `.env` with your values (see [Configuration](#configuration)), then:

```bash
docker compose up -d
```

The app is available at `http://localhost:8000`. The MCP server runs on `http://localhost:8001`. Both ports are exposed by the included `docker-compose.override.yml`.

---

## Deploying on a VPS

This stack runs on any VPS with Docker installed. Recommended providers:

- [DigitalOcean](https://www.digitalocean.com/) ($6/mo droplet)
- [Hetzner](https://www.hetzner.com/cloud/)
- [Oracle Cloud](https://www.oracle.com/cloud/free/) (free tier ARM instances)
- [Linode](https://www.linode.com/)

### Option A: Coolify (Recommended)

[Coolify](https://coolify.io) is a self-hosted PaaS that handles builds, deployments, TLS certificates, and auto-deploy from GitHub.

**1. Install Coolify on your VPS:**

```bash
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | bash
```

**2. Create a new project** in the Coolify dashboard.

**3. Add a resource** — choose **Docker Compose** and point it at this repo (`dlaporte/scoutbook-ai`).

**4. Remove or rename `docker-compose.override.yml`** — Coolify handles port exposure and TLS. The override file is for local development only. You can do this by adding it to the "Ignore files" section in Coolify's resource settings.

**5. Set environment variables** in Coolify's UI:

| Variable | Value |
|---|---|
| `APP_BASE_URL` | `https://your-app-domain.example.com` |
| `MCP_SERVER_URL` | `https://your-mcp-domain.example.com` |
| `SESSION_SECRET` | Generate: `python3 -c "import secrets; print(secrets.token_urlsafe(32))"` |
| `ANTHROPIC_API_KEY` | Your Anthropic API key (or set `OPENAI_*` vars) |

**6. Set domains** — In Coolify, click on each service and set the domain:

- **app** service: `https://your-app-domain.example.com` (port `8000`)
- **mcp** service: `https://your-mcp-domain.example.com` (port `8000`)
- **sandbox** service: No domain needed (internal only)

> **Important:** Include the `https://` prefix when entering domains in Coolify. Without it, Traefik routing will not work correctly.

**7. Deploy.** Coolify provisions Let's Encrypt TLS certificates automatically.

### Option B: CapRover

[CapRover](https://caprover.com) is another self-hosted PaaS with Docker Compose support. The setup is similar to Coolify — install CapRover on your VPS, create an app, and deploy this repo.

### Option C: Docker Compose + Caddy

For a manual VPS setup without a PaaS, see [Enabling HTTPS with Caddy](#enabling-https-with-caddy) below.

---

## Enabling HTTPS with Caddy

When running on a bare VPS without a PaaS like Coolify, your services are exposed over plain HTTP. You need a reverse proxy to:

- **Terminate TLS** — serve HTTPS with valid certificates
- **Route by hostname** — direct traffic to the right service based on domain name
- **Expose both services on port 443** — without a proxy, each service needs its own port

[Caddy](https://caddyserver.com) is the simplest option because it automatically provisions Let's Encrypt certificates with zero configuration.

**1. Create a `Caddyfile`:**

```
your-app-domain.example.com {
    reverse_proxy app:8000
}

your-mcp-domain.example.com {
    reverse_proxy mcp:8000
}
```

**2. Create `docker-compose.caddy.yml`:**

```yaml
services:
  caddy:
    image: caddy:2
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy-data:/data

volumes:
  caddy-data:
```

**3. Remove `docker-compose.override.yml`** (the Caddy proxy replaces direct port mapping):

```bash
rm docker-compose.override.yml
```

**4. Update `.env`** to use your HTTPS domains:

```bash
APP_BASE_URL=https://your-app-domain.example.com
MCP_SERVER_URL=https://your-mcp-domain.example.com
```

**5. Start everything:**

```bash
docker compose -f docker-compose.yml -f docker-compose.caddy.yml up -d
```

Caddy will automatically obtain and renew Let's Encrypt certificates for both domains. Make sure ports 80 and 443 are open on your firewall and both domains have DNS A records pointing to your VPS IP.

> **Local HTTPS:** For local development with HTTPS (e.g., testing OAuth flows), replace the domain names in the Caddyfile with `localhost`. Caddy will generate a self-signed certificate automatically.

---

## Configuration

Copy `.env.example` to `.env` and configure:

### Required

| Variable | Description |
|---|---|
| `APP_BASE_URL` | Public URL of the app (e.g., `https://chat.example.com` or `http://localhost:8000`) |
| `MCP_SERVER_URL` | Public URL of the MCP server (e.g., `https://mcp.example.com` or `http://localhost:8001`) |
| `SESSION_SECRET` | Random string for signing session cookies |

### LLM Provider

Choose one provider. If `LLM_PROVIDER` is not set, it auto-detects based on which keys are present.

**Anthropic (default):**

| Variable | Description |
|---|---|
| `ANTHROPIC_API_KEY` | Your Anthropic API key |
| `ANTHROPIC_MODEL` | Model to use (default: `claude-sonnet-4-20250514`) |

**OpenAI-compatible** (Ollama, vLLM, LM Studio, etc.):

| Variable | Description |
|---|---|
| `OPENAI_API_KEY` | API key (or any string for local servers) |
| `OPENAI_BASE_URL` | Endpoint URL (e.g., `http://192.168.8.80:8000/v1`) |
| `OPENAI_MODEL` | Model name |
| `LLM_PROVIDER` | Set to `openai` to force this provider |

### Optional

| Variable | Default | Description |
|---|---|---|
| `ADMIN_USERS` | *(none)* | Comma-separated BSA usernames for global document management |
| `RAG_TOP_K` | `8` | Number of document chunks to retrieve per query |
| `DATA_DIR` | `/data` | Directory for persistent data |
| `SANDBOX_ENABLED` | `true` | Set to `false` to disable the code execution sandbox |
| `SANDBOX_URL` | `http://sandbox:8080` | Sandbox endpoint (auto-configured) |
| `SANDBOX_TIMEOUT` | `60` | Max execution time in seconds |
| `APP_PORT` | `8000` | Host port for the app (local override only) |
| `MCP_PORT` | `8001` | Host port for the MCP server (local override only) |

---

## Persistent Data

Both the app and MCP server store data in Docker named volumes:

| Volume | Path | Contents |
|---|---|---|
| `app-data` | `/data` | Uploaded documents, ChromaDB vector store, generated exports, OAuth registration |
| `mcp-data` | `/data` | MCP server state |

These persist across container restarts and redeploys. They are only removed if you explicitly run `docker compose down -v`.

---

## Security

The code execution sandbox runs with strict security controls:

- **Read-only filesystem** — no writes except to `/tmp`
- **No network access** — runs on an isolated internal Docker network
- **No Linux capabilities** — `cap_drop: ALL`
- **No privilege escalation** — `no-new-privileges` security option
- **Resource limits** — 512 MB RAM, 1 CPU, 50 PIDs max, 60s timeout
- **Non-root user** — runs as UID 1000

These controls are enforced at the Docker level and require a host with full Docker access. Managed container platforms (Railway, Render, Fly.io) do not support these security options — set `SANDBOX_ENABLED=false` on those platforms.
