# Changelog

## 1.2.2 - 2026-07-03
Codex review + live-MCP reconciliation:
- Reframed SKILL.md write-ops list as illustrative; every W/W! tool in the catalog requires confirmation. Added missing live destructive tools (`server_scale`, `app_restore_rollback`, `app_local_backup_delete`, `server_local_backup_delete`, `app_db_password_update`, `app_admin_password_update`, `app_credentials_delete`, `server_master_password_update`, `dns_made_easy_delete_domains`/`_records`, `project_delete`, `ssh_key_delete`, add-on activation); removed catalog entries the live MCP does not expose (`server_packages`).
- Reconciled the restore/rollback guidance with the live tool set: `app_restore_rollback` (W!) IS live, so the maintenance workflow now uses it within the rollback window instead of claiming "no MCP rollback tool."
- SKILL.md now states `execute_tool`/toolset-proxy calls inherit their target's confirmation risk.
- Marked `app_cron_list_update` deprecated (endpoint returns HTTP 500) in the catalog.

## 1.2.1 - 2026-06-04
- Added three tools observed on the live server but missing from the catalog: `varnish_app_status` (R) and the toolset meta-tools `list_available_toolsets` / `get_toolset_tools` / `execute_tool`.
- Publish workflow now sets the ClawHub display name to **Cloudways MCP** (`--name`) and pins the slug (`--slug cloudways-mcp`), so the listing reads "Cloudways MCP" instead of an auto-title-cased "Cloudways Mcp".

## 1.2.0 - 2026-06-03
- Verified the entire skill against the **official Cloudways support article**. Connection is now concrete: endpoint `https://mcp.cloudways.com/mcp/`, header auth (`X-CW-Email` / `X-CW-Api-Key` / `X-Mcp-Host`), API key from platform.cloudways.com → API Integration, Claude Code (native HTTP) + Claude Desktop (`mcp-remote`, Node 18+) setups.
- Replaced the community tool catalog with the **official 134-tool catalog** (real names, 13 categories, R/W/W! flags — 44 R / 74 W / 16 W!). Remapped every tool reference across SKILL.md and all workflows (e.g. `list_servers`→`server_list`, `clear_app_cache`→`app_purge_cache`, `get_alerts`→`copilot_insights_list`).
- Documented gaps honestly: the official MCP has **no** SSL/Let's Encrypt, IP-whitelisting, team-management, `ping`, `customer_info`, or `rate_limit_status` tools — those now point to the Cloudways UI / direct API instead of inventing tools.
- Dropped the "illustrative/unverified" hedging — the catalog is verified; the live `mcp__cloudways*__*` tools remain authoritative if Cloudways changes names.

## 1.1.0 - 2026-06-03
- Removed remnants of the unverified `aphraz/cw-mcp` attribution (the repo now 404s): dropped all dead-repo citations and the entire self-hosted Python+Redis path.
- Re-centered on the official Cloudways (Remote) MCP; `installation.md` now points to the official support article for connection specifics (endpoint/auth not assumed).
- Marked the tool catalog and workflows as illustrative/unverified — the live `mcp__cloudways*__*` tools are the source of truth.

## 1.0.0 - 2026-06-02
- First public-ready cut: genericized (removed studio-specific framing + personal names), kit-standard packaging (CI + ClawHub publish workflow), no-leak CI guard.
- Translated the entire skill (SKILL.md + all references) and README to English for public distribution.
