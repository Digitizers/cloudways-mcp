---
name: cloudways-mcp
version: 1.1.0
description: |
  Operational guide for managing Cloudways servers and applications, across one or several Cloudways accounts, via the official Cloudways MCP server (Cloudways' hosted MCP / Remote MCP, per their support docs).
  Use whenever the user mentions Cloudways, a Cloudways server or app, server monitoring, app monitoring, bandwidth, disk usage, PHP/MySQL/traffic analytics, SSH/MySQL IP whitelisting, Let's Encrypt or SSL on Cloudways, Varnish cache, app cloning, backups/restore on Cloudways, Git deployments on Cloudways, or running an audit/onboarding on a Cloudways-hosted client site.
  Any write operation (start/stop/restart server, backup, restore, rollback, install/revoke SSL, update CNAME, change whitelist, clear cache, change service state, git pull) requires explicit confirmation of target server/app and intended action before execution.
---

# Cloudways MCP — Operational Skill

Managing Cloudways infrastructure through the Cloudways MCP server.

> **Connection:** This skill targets the **official Cloudways (Remote) MCP** — an MCP hosted by Cloudways (delivered Q2 2026) that you connect to directly. The source of truth for connecting is the **official article**: `support.cloudways.com/en/articles/14654372`. See `references/installation.md`.
>
> **Tool names are illustrative.** The tool catalog and workflows in this skill were compiled from an earlier community MCP implementation and are **not verified** against the official server. Always treat the live `mcp__cloudways*__*` tools as the source of truth (see "Versioning and source of truth" below).

> **Context:** The skill is built for day-to-day work managing clients/environments on Cloudways — monitoring, routine maintenance, onboarding/audit for new clients, and automations. All monetary values reported by the API are in $ (USD), not ₪.

---

## Quick Route

| Intent | Load |
|--------|------|
| Initial installation/configuration of the MCP server | `references/installation.md` |
| Don't know which tool exists / searching for a tool by name | `references/tools-catalog.md` |
| Monitoring, status check, bandwidth, analytics | `references/workflows-monitoring.md` |
| Cache clear, SSL, backup, restart, IP whitelist | `references/workflows-maintenance.md` |
| New audit / onboarding a new client | `references/workflows-onboarding.md` |
| Building an automated workflow for n8n/Make/Claude Code | `references/workflows-automation.md` |
| Multiple Cloudways accounts / multi-account configuration | `references/installation.md` (Multi-account section) |

**Load only what's needed.** Maximum 2-3 references per task. If the user just asks "show me my servers", don't load the entire catalog — call `list_servers` directly.

---

## Safety rules (read before every operation)

1. **Selecting the correct account — before anything else (multi-account).** There are **multiple Cloudways accounts**, each as a separate MCP connection with its own prefix (e.g. `mcp__cloudways-clientA__*`). Before every call — verify which account it belongs to. If more than one account is connected and it's not clear from context which one is meant — **stop and ask**, don't guess. server/app IDs are **not interchangeable between accounts** — ID 1234567 in account A is an entirely different resource (or nonexistent) in account B. Don't take an ID from one account's response and run it against another account. See the "Multi-account" section below.

2. **Write operations require explicit confirmation.** Before every call to a tool that belongs to the Write category (see list below), present to the user: **the account**, the tool name, the target server/application (ID + name), the parameters, the expected impact. Wait for a confirmation response before executing. Don't assume that confirming one operation grants confirmation for further operations — nor that confirmation on one account applies to another.

3. **Backup before a significant change.** Before `restore_app`, `rollback_app_restore`, `reset_app_file_permissions`, `manage_server_varnish`, or any configuration change — check with the user whether a recent backup exists. If not, offer to run `backup_app` / `backup_server` first.

4. **Multiple services = multiplied risk.** Cloudways usually hosts **several applications on the same server**. `stop_server` or `restart_server` affects **all** the applications. Always make sure the user is aware of the list of applications on the server before a server-level operation.

5. **`revoke_letsencrypt` and `delete_app_cname` = immediate destruction in production.** Requires double confirmation: of both the operation and the specific domain/application.

6. **Credentials.** Each account has its own API key + email, exposed through headers in the MCP. Don't print them in responses. Don't mix credentials between accounts. If the user asks to see them, refer them to Cloudways Platform → Account → API.

7. **Read-only by default.** If the user just asks "show me / check / monitor" — always choose the appropriate read-only tool. Don't suggest a destructive operation unless the user explicitly asked for it.

---

## Write operations — full list by category

For each of the following operations, **explicit confirmation is mandatory before execution**:

**Server level (affects all applications on the server):**
- `start_server`, `stop_server`, `restart_server`
- `backup_server`
- `optimize_server_disk`
- `change_service_state` (Apache/Nginx/Memcached/MySQL/Varnish)
- `manage_server_varnish`

**App level:**
- `clone_app` (creates a new copy — consumes resources)
- `backup_app`, `restore_app`, `rollback_app_restore`
- `reset_app_file_permissions`
- `enforce_app_https`
- `update_app_cname`, `delete_app_cname` ⚠️ (can break production)
- `clear_app_cache`, `manage_app_varnish`

**Security & SSL:**
- `install_ssl_certificate`, `remove_ssl_certificate` ⚠️
- `install_letsencrypt`, `renew_letsencrypt`, `set_letsencrypt_auto_renewal`
- `revoke_letsencrypt` ⚠️⚠️ (immediate destruction)
- `update_whitelisted_ips`, `allow_ip_siab`, `allow_ip_adminer`

**Git deployment:**
- `git_clone`, `git_pull` (can break production if there's a conflict)
- `generate_git_ssh_key`

---

## Confirmation pattern for a destructive operation

Before execution, present a block like this:

```
🔒 Confirm operation execution?
   Account: clientA (mcp__cloudways-clientA)
   Tool: stop_server
   Server: production-shop-il (ID: 1234567)
   Applications affected: woocommerce-prod, staging-clone, admin-tools
   Impact: all 3 applications will be offline until a manual restart
   Proceed? (yes / no / pause and check backup first)
```

Wait for an explicit response. A literal "yes" only = confirmation. Implied consent is not enough. The **account line is mandatory** when more than one account is connected — it prevents executing an operation on the wrong account.

---

## Authentication — quick overview

Each account authenticates with its **Cloudways email + API key**, generated in Cloudways Platform → **Account → API**. The API key grants every action the account can perform in the UI — treat it like a password and never print it in responses.

How the credentials are passed depends on the connection method (follow the official article). For the full connection and multi-account setup, see `references/installation.md`.

---

## Multi-account — working with multiple Cloudways accounts

There are usually **multiple Cloudways accounts** (different clients / different environments). Each account is connected as a **separate** MCP connection with its own credentials, and therefore appears in Claude with **its own prefix**:

```
mcp__cloudways-clientA__list_servers
mcp__cloudways-clientB__list_servers
mcp__cloudways-internal__list_servers
```

> The configuration (how multiple accounts are connected — one connection per account, each with its own credentials) is documented in `references/installation.md` section **Multi-account configuration**. The runtime rules are here.

### The golden rule: identify the account before every operation

1. **A single account connected** → use it, no need to ask.
2. **Multiple accounts connected** → determine which account the request belongs to **before** you call a tool:
   - If the user explicitly specified a client/account ("check clientB's prod") → use the matching connection.
   - If the server/domain name unambiguously identifies a single account → you may infer, but explicitly state which account you're operating on.
   - If **it's unclear** → stop and ask: "Which account? (clientA / clientB / internal)". Don't guess, and don't run on all of them "just to be safe".

### Complete isolation between accounts

- **IDs don't cross accounts.** A server_id / app_id you received from `mcp__cloudways-clientA` is valid **only** against clientA. Never take an ID from one account's response and pass it to a tool of another connection.
- **Per-account confirmation.** A write confirmation on one account does not apply to another. Every write operation on a new account = a new confirmation block (including the account line).
- **Credentials don't mix.** Each connection has its own email + API key. Don't assume the same credentials work on another account.

### Cross-account search (read only)

When the user asks for something broad — "which account does the domain shop.example.co.il live on?", "give me a disk overview for all accounts" — it's permitted and legitimate to **read (read-only)** from all the connections, but:
- Run the same sequence of reads on each connection **separately**, and tag each result with the account name.
- Summarize in a table with a clear "Account" column.
- **Never** perform a broad write operation across multiple accounts without individual confirmation for each one.

```
Example tagging in the response:
| Account  | Server           | disk |
|----------|------------------|------|
| clientA  | prod-shop-il     | 87%  |
| clientB  | prod-blog        | 41%  |
| internal | ops-tools        | 63%  |
```

---

## Common usage patterns (examples)

### Quick snapshot of an account
```
1. list_servers              → list of all servers
2. get_alerts                → active alerts
3. customer_info             → account details + plan status
```

### Health check before a weekend (production client)
```
1. get_server_details        → CPU/RAM/disk
2. get_server_monitoring_detail → metrics
3. get_app_monitoring_summary   → for each application
4. get_alerts                → open alerts
5. get_server_disk_usage     → if disk > 80% — red flag
```

### Renewing SSL before expiry
```
1. get_app_details           → what's the correct FQDN
2. renew_letsencrypt          ← write — requires confirmation
3. get_app_details (again)    → verify the SSL was updated
```

For more detailed patterns, load the relevant workflows.

---

## Versioning and source of truth

- **The live MCP is the source of truth — always.** The tool names and categories in the catalog here were compiled from an earlier community MCP implementation and are **not verified** against Cloudways' official MCP, which may expose different names/capabilities. Before you declare that a tool exists/doesn't exist — **check the live list of tools** connected in Claude (`mcp__cloudways*__*`), and update the catalog accordingly.
- **read-only vs. write — uncertain, so proceed with caution.** Sources conflict: some documented read-only-only (write "planned"), while this catalog assumed write operations exist. **Don't assume** — check against the live server which tools are available. In any case, every tool that makes a change **must** go through the confirmation pattern; this rule is safe even if in practice there's no write tool (in which case it simply isn't triggered).
