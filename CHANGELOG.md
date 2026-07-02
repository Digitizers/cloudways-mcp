# Changelog

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
