# Installation — Cloudways MCP Server

This skill targets the **official Cloudways (Remote) MCP** — a Cloudways-hosted MCP you connect to directly, without self-hosting (listed as "Cloudways Remote MCP", Q2 2026, on the Cloudways roadmap; the connection details are per the official article below, not assumed here).

> **The source of truth for connecting is the official article:**
> https://support.cloudways.com/en/articles/14654372-how-to-use-cloudways-mcp-server-for-ai-based-server-management
>
> Follow the steps there for the exact endpoint and the auth method (OAuth / API key). **Do not invent a URL or headers** — they are not hard-coded here on purpose. Once connected, the tools appear in Claude as `mcp__cloudways*__*`.

---

## Step 1 — Obtain API credentials from Cloudways

1. Log in to the Cloudways Platform.
2. Top right: **Account → API**.
3. Create/copy the `API Key`.
4. Note the account's email.

> The API key allows **all** the operations the account can perform in the UI. Keep it like a password — never commit it or print it in responses.

---

## Step 2 — Connect Claude to the MCP

Follow the official article for the connection (endpoint + auth). It typically results in an entry under `mcpServers` in your Claude config — Claude Code: `.mcp.json` or `claude mcp add`; Claude Desktop: `claude_desktop_config.json` — named e.g. `cloudways`.

> Keep real credentials out of version control: put them in a local, git-ignored config (e.g. `.mcp.json`, which is in `.gitignore`) rather than committing them. See `.mcp.json.example` in the repo root for the per-account shape.

---

## Multi-account configuration — multiple Cloudways accounts

Each Cloudways account is connected as its **own** MCP connection, with its own credentials, so it appears in Claude under its own prefix (`mcp__cloudways-clientA__*`). Give each connection a descriptive, client-based name — that name becomes the tool prefix.

```json
{
  "mcpServers": {
    "cloudways-clientA": { "__": "connection per the official article, using clientA's credentials" },
    "cloudways-clientB": { "__": "connection per the official article, using clientB's credentials" }
  }
}
```

(See `.mcp.json.example` in the repo root for the full per-account skeleton.)

### Safety rules for multi-account (mandatory)

- **Consistent names:** Use a uniform `cloudways-<client>` prefix so Claude (and you) immediately recognize which account each tool belongs to.
- **Separate secrets:** Don't keep all the keys in one place. Prefer a secrets manager (e.g. a vault project per client) over plain config files.
- **Don't mix:** Never use the same API key for two accounts, and don't take a server/app ID from one account against another's connection.
- **Runtime:** Claude's runtime behavior (account identification, cross-account search, per-account write-confirmations) is documented in `SKILL.md` section **Multi-account**.

---

## Step 3 — Verify the connection

After connecting, confirm in Claude:

1. `ping` → the MCP endpoint is reachable.
2. `customer_info` → credentials are valid; shows whose account you're on.
3. `list_servers` → returns your servers.

| Error | Meaning | Fix |
|-------|---------|-----|
| no `mcp__cloudways*__*` tools | not connected | re-check the connection steps in the official article |
| `401 / authentication` | wrong credentials | re-check the account email + API key (Account → API) |
| `429 / rate limited` | hit the rate cap | wait and retry; check `rate_limit_status` |

To test credentials directly against the public Cloudways API (independent of MCP):

```bash
curl -X POST "https://api.cloudways.com/api/v1/oauth/access_token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "email=YOUR_EMAIL&api_key=YOUR_API_KEY"
```

API reference: https://developers.cloudways.com/

---

## Notes

- This skill does not cover self-hosting an MCP server — the official hosted MCP is the supported path. Follow the official article for any connection specifics.
- Tool names referenced throughout this skill are **illustrative** (compiled from an earlier community MCP) — the live `mcp__cloudways*__*` tools are the source of truth.
