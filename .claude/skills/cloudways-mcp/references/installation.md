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
| `X-Mcp-Host` | the client you connect from — official values: `claude-code`, `claude-desktop`, `cursor`, `Devin` (the client formerly called Windsurf — the article's own snippet sends the capitalised `Devin`, and header values are case-sensitive), `vs-code`, `gemini-cli`, `codex`, `codex-cli` |

---

## Step 2 — Connect Claude to the MCP

### Env var (zero-config — devices and cloud sessions)

The repo commits a `.mcp.json` whose `X-Access-Token` header reads
`${CLOUDWAYS_ACCESS_TOKEN:-}` — a placeholder, never a real token. Set that variable — in
your shell profile on a device, or in the claude.ai cloud environment's environment
variables for web/phone sessions — and the connection (named `cloudways-env`) authenticates
automatically. While the variable is unset the config still parses (the `:-` default), but
the connection can't authenticate and shows as unavailable in `/mcp` — expected until you
provide the token. The `cloudways-env` name is deliberate: project scope beats user scope on
name collisions, so it never shadows a `cloudways` or `cloudways-<client>` connection you
add with `claude mcp add -s user`. Never put a real token in `.mcp.json` itself; it is
tracked in git. The committed `.claude/settings.json` sets `enableAllProjectMcpServers`,
which is why the project-scope server activates without a prompt once the folder is trusted —
to opt out locally, set `"enableAllProjectMcpServers": false` in `.claude/settings.local.json`
(not committed). Cloud environments with a restricted network policy must allow
`mcp.cloudways.com`.

### Claude Code (native HTTP — recommended)

```bash
claude mcp add --transport http \
  --header 'X-Access-Token: ${CLOUDWAYS_ACCESS_TOKEN:-}' \
  --header "X-Mcp-Host: claude-code" \
  -s user \
  cloudways https://mcp.cloudways.com/mcp/
```

**Keep the single quotes, and do not put the token's value here.** See "Handling the
token safely" below: quoted this way, the shell never expands it, so neither this
command's arguments nor the stored config holds the secret — Claude Code expands it from
its own environment when it opens the connection.

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
        "mcp-remote@0.14.0",
        "https://mcp.cloudways.com/mcp/",
        "--header", "X-Access-Token:<your-cloudways-access-token>",
        "--header", "X-Mcp-Host:claude-desktop"
      ]
    }
  }
}
```

> **Pin `mcp-remote`.** Unpinned, `npx` resolves whatever the registry serves at launch and
> executes it — and this config hands that package a live Access Token on its command line, so
> a compromised release, maintainer account or transitive dependency would receive it. The
> version above is pinned deliberately; bump it after reading the upstream release notes, and
> record the new digest here in the same commit.
>
> ```
> mcp-remote@0.14.0
> sha512-QBYGz02kc2AhhM6RNDzNyoA/FlzwJCNYPFF+o3opSvwfo5lnp8mcVIj/Zqw92dPuoafyLBKQZkpaEJdzlB/png==
> ```
>
> To check what the registry served you, compute the digest from the bytes — **do not compare
> against the line `npm pack` prints**, which elides the middle
> (`sha512-QBYGz02kc2Ahh[...]kpaEJdzlB/png==`), so a comparison against it only ever checks a
> prefix and a suffix:
>
> ```bash
> npm pack mcp-remote@0.14.0
> printf 'sha512-%s\n' "$(openssl dgst -sha512 -binary mcp-remote-0.14.0.tgz | openssl base64 -A)"
> ```
>
> (`npm pack --json` also emits the full value, if you would rather read npm's own figure than
> compute one.) A digest **recorded here, out of band** is what makes this worth running: npm
> forbids republishing a version with different content, so it catches a registry that later
> serves different bytes for 0.14.0.

> **This config file holds the literal token.** The `${VAR}` expansion used above is
> Claude Code's; do not assume the Desktop bridge performs it — treat that file as holding
> the real value, keep it at mode 600 (`chmod 600 ~/Library/Application\ Support/Claude/claude_desktop_config.json`),
> and give it the **smallest role that works** (READ for monitoring-only), since it is a
> credential at rest rather than one held in an environment.

> **No spaces around the colon** in `--header` values for the bridge: use `X-Access-Token:abc123`, not `X-Access-Token: abc123`. After saving, **fully quit** Claude Desktop (Cmd-Q / tray → Quit — closing the window is not enough) and reopen.

> Keep real credentials out of version control. For Claude Code, put the token in the `CLOUDWAYS_ACCESS_TOKEN` env var (the committed `.mcp.json` reads it, and so does the user-scope form above) — never edit a real token into `.mcp.json`, which is a **tracked** file. See `.mcp.json.example` in the repo root for the per-account shape. Header names are case-sensitive.

### Other clients (Cursor, Devin, VS Code Copilot, Gemini CLI, Codex)

Same endpoint and headers everywhere; only the config-file shape and the `X-Mcp-Host` value differ per client. The [official article](https://support.cloudways.com/en/articles/14654372-how-to-use-cloudways-mcp-server-for-ai-based-server-management) has the full snippet for each — but three of them use a **non-obvious key**, and the article warns that a wrong variant **fails silently**:

| Client | Config file | URL key | Headers key | `X-Mcp-Host` |
|--------|-------------|---------|-------------|--------------|
| Cursor | `~/.cursor/mcp.json` | `url` | `headers` | `cursor` |
| Devin (ex-Windsurf) | `~/.codeium/devin/mcp_config.json` | **`serverUrl`** | `headers` | `Devin` |
| VS Code (Copilot) | `~/Library/Application Support/Code/User/mcp.json` (macOS) / `%APPDATA%\Code\User\mcp.json` | `url`, and the file uses **`"servers"` (not `"mcpServers"`)** plus a required **`"type": "http"`** | `headers` | `vs-code` |
| Gemini CLI | `~/.gemini/settings.json` | **`httpUrl`** — `url` there attempts an SSE connection instead and fails | `headers` | `gemini-cli` |
| Codex / Codex CLI | `~/.codex/config.toml` | `url` | **`[mcp_servers.cloudways.http_headers]`** (a separate TOML table) | `codex` / `codex-cli` |

**Cursor connects over native HTTP** — no Node and no `mcp-remote`; the bridge is only a fallback for proxy/connection problems (and then it needs Node v24+, as Claude Desktop does). The article also offers a one-click "Install in Cursor" button that prompts for the credentials.

---

## Handling the token safely

An Access Token carries whatever role it was issued with — up to FULL ACCESS, which is
everything the Cloudways account can do, including destructive server and app actions and
billing. Keep the value in **one** place, the environment Claude Code starts with, and put a
*placeholder* everywhere else.

**Never pass the value to `claude mcp add`.** Written as `"X-Access-Token: $TOKEN"`, the
shell expands it before launching anything, so the live token is in that process's argument
list — readable through `ps` or `/proc/<pid>/cmdline` by any other user on the machine while
the command runs — and `claude mcp add` then stores the **resolved value** in
`~/.claude.json` in plaintext, where it stays until you remove the connection. Single-quoted
as `'X-Access-Token: ${CLOUDWAYS_ACCESS_TOKEN:-}'`, neither the argument list nor the stored
config carries the secret. Claude Code expands `${VAR}` in headers for local- and
user-scoped entries in `~/.claude.json`, not only in a project `.mcp.json`; an unset
reference produces a `Missing environment variables` warning in `claude mcp list`.

**Where the value lives.** It must be in the environment Claude Code itself starts with:

```bash
printf 'Cloudways Access Token: '
read -rs CLOUDWAYS_ACCESS_TOKEN; echo
export CLOUDWAYS_ACCESS_TOKEN
```

`read -rs` does not echo the token and never writes it to `~/.zsh_history` /
`~/.bash_history`, but it lasts only for that shell — start Claude Code **from it**. For a
persistent setup, put the export in your shell profile at mode 600, or better, read it from
a keychain rather than storing it inline:

```bash
export CLOUDWAYS_ACCESS_TOKEN="$(security find-generic-password -s cloudways-api -w)"   # macOS
```

In claude.ai cloud sessions the equivalent is the environment's own environment variables.

**If a token may have been exposed** — pasted into a chat, committed, left in a history file
or an old `~/.claude.json` entry — **revoke it at platform.cloudways.com → API** and issue a
new one with the smallest role that works. To check whether an old connection left a literal
behind, **parse** the config rather than grepping it (JSON may put a value on the line after
its key), and report names rather than values:

```bash
node -e '
const fs = require("fs"), p = require("os").homedir() + "/.claude.json";
let c; try { c = JSON.parse(fs.readFileSync(p, "utf8")); }
catch (e) { console.error("could not read " + p); process.exit(1); }
const all = [...Object.entries(c.mcpServers || {}),
             ...Object.values(c.projects || {}).flatMap(x => Object.entries(x.mcpServers || {}))];
const literal = v => { const re = /\$\{[A-Za-z_][A-Za-z0-9_]*(?::-([^}]*))?\}/g;
                       let n = v.replace(re, "").length;
                       for (const m of v.matchAll(re)) n += (m[1] || "").length;
                       return n; };
const hits = all.filter(([, s]) => { const v = (s.headers || {})["X-Access-Token"];
                                     return typeof v === "string" && literal(v) > 0; })
                .map(([n]) => n);
console.log(hits.length
  ? "Literal token stored in: " + hits.join(", ") + " — revoke it and re-add with the placeholder form."
  : "No literal token in ~/.claude.json.");
'
```

It counts literal material inside `:-` defaults as well as outside the placeholders — a
token hides just as well in `${CLOUDWAYS_ACCESS_TOKEN:-cw_live}`, which is still plaintext
in the file and is still what gets sent whenever the variable is unset.

---

## Multi-account configuration — multiple Cloudways accounts

Each Cloudways account is a **separate** MCP connection with its own Access Token, so it appears under its own prefix (`mcp__cloudways-clientA__*`). Give each a descriptive, client-based name — that name becomes the tool prefix. Same endpoint for all; only the `X-Access-Token` differs.

Give each account its **own variable name** and reference it as a placeholder, so no
token reaches a command line or a config file:

```bash
# one `claude mcp add` per account, each reading its own variable:
claude mcp add --transport http \
  --header 'X-Access-Token: ${CLOUDWAYS_TOKEN_CLIENTA:-}' \
  --header "X-Mcp-Host: claude-code" \
  -s user cloudways-clientA https://mcp.cloudways.com/mcp/

claude mcp add --transport http \
  --header 'X-Access-Token: ${CLOUDWAYS_TOKEN_CLIENTB:-}' \
  --header "X-Mcp-Host: claude-code" \
  -s user cloudways-clientB https://mcp.cloudways.com/mcp/
```

Export `CLOUDWAYS_TOKEN_CLIENTA` / `CLOUDWAYS_TOKEN_CLIENTB` in the environment Claude
Code starts with. The header sent is always `X-Access-Token`; only the source differs per
connection, which is what keeps the accounts separated.

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
