# Cloudways MCP — setup & connection

How to run the Cloudways MCP server and connect a client (Claude Code, Claude
Desktop, or any MCP-capable client) to it.

Official guide:
[How to Use Cloudways MCP Server](https://support.cloudways.com/en/articles/14654372-how-to-use-cloudways-mcp-server-for-ai-based-server-management)
· Upstream project: [`aphraz/cw-mcp`](https://github.com/aphraz/cw-mcp)

> The exact flags, header names, and config keys below follow the upstream
> README. Cloudways may host an official endpoint with a different connection
> flow — when the README or the official article differs from what's written
> here, follow the source.

## 1. Get Cloudways API credentials

1. Log in to the [Cloudways Platform](https://platform.cloudways.com/).
2. Open your account menu → **API Keys** (Account → API).
3. Copy your **account email** and generate/copy your **API Key**.

These two values authenticate every request. Treat the API key like a password.

## 2. Prerequisites for self-hosting the server

- **Python 3.11+**
- **Redis** running and reachable (used for token storage and rate limiting)
- The Cloudways API credentials from step 1

## 3. Run the server

```bash
git clone https://github.com/aphraz/cw-mcp.git
cd cw-mcp
# install dependencies per the repo (e.g. uv sync / pip install -r requirements.txt)
# configure environment variables (see below)
# start the server
```

Once running, the server listens at:

```
http://127.0.0.1:7000/mcp
```

(Streamable HTTP transport.)

### Environment variables

| Variable | Purpose |
| --- | --- |
| `ENCRYPTION_KEY` | Key used to encrypt stored tokens. Auto-generated if unset. |
| `REDIS_URL` | Redis connection string (e.g. `redis://localhost:6379`). |
| `RATE_LIMIT_REQUESTS` | Requests-per-minute allowed per customer. |
| `LOG_LEVEL` | Logging verbosity (optional, e.g. `INFO`, `DEBUG`). |

Cloudways credentials (email + API key) are supplied **per request by the
client** (see step 4), not as server env vars — this lets one server serve
multiple customers securely.

## 4. Connect a client

The server is an HTTP MCP endpoint. Point your client at it and pass the
Cloudways email + API key as authentication headers. Confirm the exact header
names against the upstream README before relying on them.

### Claude Code (`.mcp.json` in the project root)

```json
{
  "mcpServers": {
    "cloudways": {
      "type": "http",
      "url": "http://127.0.0.1:7000/mcp",
      "headers": {
        "X-Cloudways-Email": "you@example.com",
        "X-Cloudways-API-Key": "YOUR_CLOUDWAYS_API_KEY"
      }
    }
  }
}
```

You can also add it from the CLI:

```bash
claude mcp add --transport http cloudways http://127.0.0.1:7000/mcp \
  --header "X-Cloudways-Email: you@example.com" \
  --header "X-Cloudways-API-Key: YOUR_CLOUDWAYS_API_KEY"
```

### Claude Desktop (`claude_desktop_config.json`)

Use the same HTTP endpoint and headers under `mcpServers` → `cloudways`.

> Keep real credentials out of version control. Prefer per-user / local config
> (e.g. `.mcp.json` ignored by git, or environment substitution) over committing
> the email and API key.

## 5. Verify

After connecting, the client should expose `mcp__cloudways__*` tools. Verify
with:

1. `ping` → server reachable.
2. `customer_info` → credentials valid; shows your account.

If `customer_info` fails, recheck the email/API key headers and that the server
can reach the Cloudways API.
