# Installation — Cloudways MCP Server

This skill targets the **official Cloudways (Remote) MCP** — a Cloudways-hosted MCP at `https://mcp.cloudways.com/mcp/` that you connect to directly, without self-hosting.

> **Source of truth:** [How to Use Cloudways MCP Server for AI-Based Server Management](https://support.cloudways.com/en/articles/14654372-how-to-use-cloudways-mcp-server-for-ai-based-server-management). The endpoint, headers, and steps below are from that article; if Cloudways changes them, the article wins.

**Prerequisites**

- A valid Cloudways account with API access.
- A Cloudways **Access Token** (see Step 1).
- **Node.js v24.14.1+** (only for the Claude Desktop path, which uses the `mcp-remote` bridge). Claude Code connects over native HTTP and does not need it.

---

## Step 1 — Generate an Access Token

1. Log in to [platform.cloudways.com](https://platform.cloudways.com).
2. Open the **API** section (bottom-left of the platform — the same place the legacy API key lived).
3. Generate an **Access Token** (Access Token Details → Create Access Token). Note: only the **primary account owner** can create/manage API credentials — team-member accounts have no API Integration section. Give the token a name, an expiration period (1 day → never; shortest that works), and a **role**:
   - **READ** — look-ups only (status, config, monitoring). Recommended starting point for every new integration, and for monitoring-only connections.
   - **LIMITED** — only the endpoint groups you select.
   - **FULL ACCESS** — everything the account can do, including destructive actions. Only for connections that genuinely need to make changes.
4. Copy the token.

> Treat the token like a password — never commit it or print it in responses. Prefer one token **per integration** (per MCP connection), each with the minimum role it needs, so tokens can be revoked individually.

> **Legacy API key — deprecated.** The old flow (API key + `X-CW-Email`/`X-CW-Api-Key` headers) still works but the API key **stops working on October 15, 2026**. If you have an existing connection using the old headers, regenerate as an Access Token and update the connection before then. Until migrated, a legacy connection retains **unrestricted full-account access** — the RBAC roles apply only to Access Tokens. And note there is still **no per-tool permission control at the MCP layer** beyond the token's role: a FULL ACCESS token can call every tool.

**Required headers** (every request; header names are case-sensitive):

| Header | Value |
|--------|-------|
| `X-Access-Token` | your Cloudways Access Token |
| `X-Mcp-Host` | the client you connect from — official values: `claude-code`, `claude-desktop`, `cursor`, `windsurf`, `vs-code`, `gemini-cli`, `codex`, `codex-cli` |

---

## Step 2 — Connect Claude to the MCP

### Claude Code (native HTTP — recommended)

```bash
claude mcp add --transport http \
  --header "X-Access-Token: <your-cloudways-access-token>" \
  --header "X-Mcp-Host: claude-code" \
  -s user \
  cloudways https://mcp.cloudways.com/mcp/
```

`-s user` stores it at the user level so it persists across projects. Verify with `claude mcp list`; remove later with `claude mcp remove cloudways`.

### Claude Desktop (via `mcp-remote` bridge — needs Node v24+)

Claude Desktop does not natively support remote HTTP MCP servers, so it uses the `mcp-remote` Node bridge. Config file:

- **macOS:** `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Windows:** `%APPDATA%\Claude\claude_desktop_config.json`

```json
{
  "mcpServers": {
    "cloudways": {
      "command": "npx",
      "args": [
        "mcp-remote",
        "https://mcp.cloudways.com/mcp/",
        "--header", "X-Access-Token:<your-cloudways-access-token>",
        "--header", "X-Mcp-Host:claude-desktop"
      ]
    }
  }
}
```

> **No spaces around the colon** in `--header` values for the bridge: use `X-Access-Token:abc123`, not `X-Access-Token: abc123`. After saving, **fully quit** Claude Desktop (Cmd-Q / tray → Quit — closing the window is not enough) and reopen.

> Keep real credentials out of version control. For Claude Code, prefer the `claude mcp add` command above (stored in your user config) or a git-ignored `.mcp.json`. See `.mcp.json.example` in the repo root for the per-account shape. Header names are case-sensitive.

### Other clients (Cursor, Windsurf, VS Code Copilot, Gemini CLI, Codex)

Same endpoint and headers everywhere; only the config-file shape and the `X-Mcp-Host` value differ per client. The [official article](https://support.cloudways.com/en/articles/14654372-how-to-use-cloudways-mcp-server-for-ai-based-server-management) has the exact snippet for each client.

---

## Multi-account configuration — multiple Cloudways accounts

Each Cloudways account is a **separate** MCP connection with its own Access Token, so it appears under its own prefix (`mcp__cloudways-clientA__*`). Give each a descriptive, client-based name — that name becomes the tool prefix. Same endpoint for all; only the `X-Access-Token` differs.

```bash
# one `claude mcp add` per account, with that account's token:
claude mcp add --transport http \
  --header "X-Access-Token: <clientA-access-token>" \
  --header "X-Mcp-Host: claude-code" \
  -s user cloudways-clientA https://mcp.cloudways.com/mcp/
```

(See `.mcp.json.example` for the JSON form across multiple accounts.)

### Safety rules for multi-account (mandatory)

- **Consistent names:** uniform `cloudways-<client>` prefix so Claude (and you) immediately recognize which account each tool belongs to.
- **Separate secrets:** don't keep all the tokens in one place. Prefer a secrets manager (a vault project per client) over plain config files.
- **Don't mix:** never reuse one token across accounts, and never take a server/app ID from one account against another's connection.
- **Scope by role:** generate each connection's token with the **minimum role** it needs — READ for monitoring/audit connections, LIMITED for specific workflows, FULL ACCESS only where changes are genuinely required. Set an expiration and rotate; revoke a client's token the moment the engagement ends.
- **Runtime:** account identification, cross-account search, and per-account write-confirmations are documented in `SKILL.md` → **Multi-account**.

---

## Step 3 — Verify the connection

In Claude, ask: **"Show me all my Cloudways servers"** → calls `server_list` and returns your servers (name, status, provider, region, IP). That round-trip confirms the endpoint + credentials.

(There is no `ping` / `customer_info` tool on the official MCP — `server_list` is the liveness + auth check. Identify which account you're on by the connection prefix.)

| Symptom | Meaning | Fix |
|---------|---------|-----|
| Connection failed / red indicator | wrong URL | endpoint must be exactly `https://mcp.cloudways.com/mcp/` (trailing slash) |
| `401 Unauthorized` | bad credentials | re-check the Access Token (case-sensitive header); it may be expired or revoked — regenerate in the platform |
| Write tool fails but reads work | token role too narrow | the connection uses a READ (or too-narrow LIMITED) token; use a token whose role covers the operation |
| No `mcp__cloudways*__*` tools | not connected / stale cache | restart the client (see "Tools not appearing" below) |
| Timeout | transient network | retry after a moment |
| `mcp-remote not found` (Desktop) | Node missing | install Node.js v24+, ensure `npx` is on PATH |

To test credentials directly against the public Cloudways API, independent of the MCP layer (useful to isolate "bad credentials" from "MCP connection problem"):

```bash
# Access Token (current):
curl -H "Authorization: Bearer YOUR_ACCESS_TOKEN" "https://api.cloudways.com/api/v2/server"

# Legacy API key (works until the EOL, 2026-10-15):
curl -X POST "https://api.cloudways.com/api/v1/oauth/access_token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "email=YOUR_EMAIL&api_key=YOUR_API_KEY"
```

API reference: <https://developers.cloudways.com/> — note the tools article's `oauth_access_token_generate` tool does **not** exist on the live MCP (verified 2026-07-20); mint direct-API tokens through the Cloudways platform instead.

---

## Tools not appearing / after an update

MCP clients **cache the tool list** on first connect. If new tools don't show up, or the agent says a tool doesn't exist:

- **Quickest:** in the client's MCP server settings, toggle `cloudways` off then on.
- **If no toggle:** **fully quit** the client (Cmd-Q on macOS / File → Exit / tray → Quit — closing the window is not enough) and reopen.

Then re-test with "Show me all my Cloudways projects" (`project_list`).

> Note the intentional design: even when correctly connected, the client sees only **65 tools** (62 direct + 3 meta-tools). The 62 direct tools are **members of the toolsets**, not a separate tier — so of the 244 total (241 toolset members + 3 meta-tools, live-verified 2026-07-20) the hidden remainder is **179**, discovered on demand via `list_available_toolsets` / `get_toolset_tools` / `execute_tool`. Their absence from the visible list is **not** a caching problem.

---

## Notes

- This skill does **not** cover self-hosting an MCP server — the official hosted MCP is the supported path.
- Tool names throughout this skill match the official articles' catalog (see `tools-catalog.md`). The **live** `mcp__cloudways*__*` tools remain the source of truth if Cloudways adds or renames any.
