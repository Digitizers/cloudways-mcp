# Installation — Cloudways MCP Server

There are **two ways** to connect. Pick yours:

| | Official — Cloudways (Remote) MCP | Self-hosted — community `cw-mcp` |
|---|---|---|
| Who runs it | **Cloudways** (hosted) | **You** (Python+Redis local/VPS) |
| Maintenance | None | You are responsible for uptime, security, credentials |
| Source | The official article (below) | `aphraz/cw-mcp` — ⚠️ **currently 404** |
| When | Recommended default (shipped Q2 2026) | Only if you have a copy/fork available |

---

## Option 1 — Official Cloudways (Remote) MCP  ✅ Recommended

Cloudways launched their own **hosted** MCP (Cloudways Remote MCP, Q2 2026). You connect to it directly — **without** Python/Redis/self-hosting.

> **The source of truth for connecting is the official article:**
> https://support.cloudways.com/en/articles/14654372-how-to-use-cloudways-mcp-server-for-ai-based-server-management
>
> Follow the steps there for the exact endpoint and the auth method (OAuth / API key). **Do not invent a URL or headers** — they may differ from the community version, which is why they are not hard-coded here. The credentials rules (Account → API) are the same — see below.
>
> After connecting, the tools will appear in Claude as `mcp__cloudways*__*` without you running anything locally. That is the sign you are on the official path.

To work with **multiple accounts** via the official path — the same connection-per-account principle; see the "Multi-account configuration" section below (applies to both ways).

---

## Option 2 — Self-hosted (community `cw-mcp`)

> **Important:** In this approach you run the server yourself. ⚠️ The repo `github.com/aphraz/cw-mcp` **currently returns 404** — before you start, make sure you have a copy/fork of the code available. If you don't — go to Option 1.

> Cloudways does not run this server for you. It runs **on your side** and calls the Cloudways API on your behalf — you are responsible for the security, the credentials, and the uptime.

---

## Prerequisites (Option 2)

- **Python 3.11+** (not 3.10 — the `asyncio.TaskGroup` that the MCP uses requires 3.11)
- **Redis** running (locally or remote). Uses: encrypted storage of tokens, rate limiting, session isolation.
- **Cloudways account** with API access enabled
- **Node.js 18+** for `npx mcp-remote` (if connecting from Claude Desktop)

---

## Step 1 — Obtaining API credentials from Cloudways

1. Log in to the Cloudways Platform
2. Top right: **Account → API**
3. Create/copy the `API Key`
4. Note the account's email

> The API key allows **all** the operations the account can perform in the UI. Keep it like a password.

---

## Step 2 — Clone and install the MCP server

```bash
# Clone
git clone https://github.com/aphraz/cw-mcp.git
cd cw-mcp

# Virtual env
python3 -m venv venv
source venv/bin/activate    # macOS/Linux
# or: venv\Scripts\activate  # Windows

# Dependencies
pip install -r requirements.txt
```

---

## Step 3 — Redis

### Local (macOS):
```bash
brew install redis
brew services start redis
# Check:
redis-cli ping   # Expected: PONG
```

### Local (Linux):
```bash
sudo apt install redis-server
sudo systemctl enable --now redis-server
redis-cli ping
```

### Docker (any platform):
```bash
docker run -d --name cw-mcp-redis -p 6379:6379 redis:7-alpine
```

---

## Step 4 — Environment variables

Create a `.env` file in the root of the repo (make sure it's in `.gitignore`):

```bash
# Encryption key — required. Generate once and save:
ENCRYPTION_KEY=$(python -c 'from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())')

# Redis URL
REDIS_URL=redis://localhost:6379/0

# Performance
HTTP_POOL_SIZE=500
REDIS_POOL_SIZE=500

# Rate limiting (requests per minute, per customer)
RATE_LIMIT_REQUESTS=90
RATE_LIMIT_WINDOW=60

# Logging
LOG_LEVEL=INFO
LOG_FORMAT=console
```

**Keep `ENCRYPTION_KEY` in a safe place.** If you lose it, all the tokens stored in Redis will not be decryptable. If you replace it, all users will have to re-auth.

> **Security tip:** `ENCRYPTION_KEY` is exactly the kind of secret you should keep in a vault (Infisical / OpenBao / 1Password), not in `.env` plain text.

---

## Step 5 — Starting the server

```bash
source venv/bin/activate
python cw-mcp.py
```

Expected to see:
```
==================================================
🚀 Cloudways MCP Server
==================================================
INFO: Started server process [XXXX]
INFO: Uvicorn running on http://127.0.0.1:7000
```

The endpoint is `http://127.0.0.1:7000/mcp`.

### Quick liveness check:
```bash
curl -v "http://127.0.0.1:7000/mcp/"
# Should return 200 or 405 (not 404/connection refused)
```

---

## Step 6 — Connecting to Claude Desktop / Claude Code

The live MCP server is HTTP. To connect it to Claude (which speaks stdio), you use `mcp-remote` as a proxy.

### Claude Desktop config

File location:
- **macOS:** `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Windows:** `%APPDATA%\Claude\claude_desktop_config.json`
- **Linux:** `~/.config/Claude/claude_desktop_config.json`

Edit and add to `mcpServers`:

```json
{
  "mcpServers": {
    "cloudways": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "http://127.0.0.1:7000/mcp",
        "--header",
        "x-cloudways-email: ${CLOUDWAYS_EMAIL}",
        "--header",
        "x-cloudways-api-key: ${CLOUDWAYS_API_KEY}"
      ],
      "env": {
        "CLOUDWAYS_EMAIL": "your-account@example.com",
        "CLOUDWAYS_API_KEY": "your-cloudways-api-key"
      }
    }
  }
}
```

After saving — **Quit & Reopen** Claude Desktop (not just close the window).

### Claude Code config

For Claude Code, add the same blob to `~/.claude.json` or via `claude mcp add` (depending on the version).

Newer Claude Code versions also support a direct `http` transport (without `mcp-remote` as a proxy), with headers:

```bash
claude mcp add --transport http cloudways http://127.0.0.1:7000/mcp \
  --header "x-cloudways-email: your-account@example.com" \
  --header "x-cloudways-api-key: your-cloudways-api-key"
```

---

## Multi-account configuration — multiple Cloudways accounts

Suppose there are **multiple Cloudways accounts**. The `aphraz/cw-mcp` MCP server is **multi-tenant** — it reads the credentials from the headers of **every request**, and isolates sessions in Redis by customer. This means: usually you **don't need multiple server instances** — it's enough to set up in Claude **one connection per account**, all pointing to the same URL, just with different headers.

### Approach A — connection per account (recommended)

One server instance (`http://127.0.0.1:7000/mcp`), and several entries in `mcpServers`, one per account. Descriptive names by client — they become the prefix of the tools (`mcp__cloudways-clientA__*`):

```json
{
  "mcpServers": {
    "cloudways-clientA": {
      "command": "npx",
      "args": [
        "-y", "mcp-remote", "http://127.0.0.1:7000/mcp",
        "--header", "x-cloudways-email: ${CW_A_EMAIL}",
        "--header", "x-cloudways-api-key: ${CW_A_API_KEY}"
      ],
      "env": { "CW_A_EMAIL": "clientA@example.com", "CW_A_API_KEY": "..." }
    },
    "cloudways-clientB": {
      "command": "npx",
      "args": [
        "-y", "mcp-remote", "http://127.0.0.1:7000/mcp",
        "--header", "x-cloudways-email: ${CW_B_EMAIL}",
        "--header", "x-cloudways-api-key: ${CW_B_API_KEY}"
      ],
      "env": { "CW_B_EMAIL": "clientB@example.com", "CW_B_API_KEY": "..." }
    }
  }
}
```

(In Claude Code with the direct `http` transport — same idea, an entry per account with its own headers.)

### Approach B — instance per account (stronger isolation)

If you want full process separation (credentials from separate Infisical projects, or separate rate-limit/logs per client), run several server instances on different ports — each with its own `.env`:

```bash
PORT=7000 ENV_FILE=.env.clientA  python cw-mcp.py   # clientA
PORT=7001 ENV_FILE=.env.clientB  python cw-mcp.py   # clientB
```

Then in the config each connection points to its own port (`:7000`, `:7001`...). Consider a systemd unit / Docker Compose service per instance.

### Safety rules for multi-account (mandatory)

- **Consistent names:** Use a uniform prefix `cloudways-<client>` so that Claude (and you too) immediately recognize which account each tool belongs to.
- **Separate secrets:** Don't put all the keys in the same `.env`. Prefer an Infisical/OpenBao project per client (see the security checklist below).
- **Don't mix:** Never use the same API key for two accounts, and don't take a server/app ID from one account against another's connection.
- **runtime:** Claude's runtime behavior (account identification, cross-account search, per-account write-confirmations) is documented in `SKILL.md` section **Multi-account**.

---

## Step 7 — Testing in Claude

Open a conversation in Claude Desktop. Ask:
```
list my Cloudways servers
```

Expected: a list of your servers. If you get an error:

| Error | Meaning | Solution |
|--------|--------|--------|
| `connection refused` | The server is not running | `python cw-mcp.py` |
| `401 / authentication` | Incorrect credentials | Check email + API key |
| `429 / rate limited` | You hit the cap | Wait a minute / increase `RATE_LIMIT_REQUESTS` |
| `redis connection error` | Redis is not running | `redis-cli ping` → start it |
| `cryptography invalid token` | `ENCRYPTION_KEY` differs from what was used for encoding | Restore the old key or clear Redis |

To directly test credentials against the API (without going through MCP):

```bash
curl -X POST "https://api.cloudways.com/api/v1/oauth/access_token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "email=YOUR_EMAIL&api_key=YOUR_API_KEY"
```

---

## Running as a service (production-grade)

### With systemd (Linux):

Create `/etc/systemd/system/cw-mcp.service`:

```ini
[Unit]
Description=Cloudways MCP Server
After=network.target redis.service

[Service]
Type=simple
User=cw-mcp
WorkingDirectory=/opt/cw-mcp
EnvironmentFile=/opt/cw-mcp/.env
ExecStart=/opt/cw-mcp/venv/bin/python cw-mcp.py
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable --now cw-mcp
sudo systemctl status cw-mcp
```

### With Docker Compose (recommended if your VPS already runs n8n on Docker):

```yaml
services:
  cw-mcp-redis:
    image: redis:7-alpine
    restart: unless-stopped
    volumes:
      - cw-mcp-redis:/data

  cw-mcp:
    build: .
    restart: unless-stopped
    depends_on: [cw-mcp-redis]
    env_file: .env
    environment:
      REDIS_URL: redis://cw-mcp-redis:6379/0
    ports:
      - "127.0.0.1:7000:7000"

volumes:
  cw-mcp-redis:
```

> **Safety:** `ports: 127.0.0.1:7000:7000` — bind only to loopback, not to the internet. If you want remote access, put it behind Nginx + Cloudflare Access or similar, and don't expose the port directly.

---

## Self-hosting on Cloudways itself

Possible, and even elegant. The VPS you're already paying for can also host its own MCP server. Considerations:
- Don't put this on a production server that hosts client applications — put it on an operational/internal server.
- Connect Cloudflare Access (Zero Trust) so that only you can access the endpoint remotely.
- Don't run this on a shared server of different projects — credentials will be exposed between the environments.

---

## Security — checklist before production use

- [ ] `ENCRYPTION_KEY` stored in Infisical / OpenBao / 1Password (not in plain `.env`)
- [ ] `.env` in `.gitignore`
- [ ] Redis not exposed to the internet (`bind 127.0.0.1` or password protected)
- [ ] Port 7000 not exposed to the internet (loopback only or behind an auth proxy)
- [ ] Cloudways API key — separate per environment if relevant, rotatable/revocable
- [ ] Logs are not printed with credentials (`LOG_LEVEL=INFO` and not `DEBUG` in production)
- [ ] Rate limit configured — don't leave the default if you serve multiple users

---

## Troubleshooting

```bash
# Is the process running?
ps aux | grep cw-mcp

# Is the port taken?
lsof -i :7000

# Is Redis available?
redis-cli ping

# Logs (if running under systemd):
journalctl -u cw-mcp -f

# Logs (if running idiomatically):
# stdout — redirect it to a file if you run under nohup
```

---

## Issues / contribute

The repo: `https://github.com/aphraz/cw-mcp`

Before you report a bug — try to reproduce the problem with `LOG_LEVEL=DEBUG` and save the log. The MCP server is still young and relatively unstable; you may run into edge cases.
