# Changelog

## 1.4.0 - 2026-07-21
Zero-config connection for cloud sessions and devices:
- **Committed `.mcp.json`** (placeholders only) — the `cloudways` connection reads its token from the `CLOUDWAYS_ACCESS_TOKEN` env var, so claude.ai cloud environments (which load the repo's `.mcp.json` from the clone and inject env vars from the environment config) and devices with the var in their shell get the tools with no per-machine setup. Unset var → server skipped silently.
- `.mcp.json` removed from `.gitignore` (the tracked file must only ever contain `${VAR}` placeholders); `.mcp.json.example` re-purposed as the multi-account reference shape — real tokens go to user scope (`claude mcp add -s user`) or env vars, never into the tracked file.
- `.claude/settings.json` sets `enableAllProjectMcpServers` so the committed config is auto-approved.
- installation.md + README document the env-var route and the migration off a local gitignored `.mcp.json`.

## 1.3.1 - 2026-07-20
Live-MCP re-enumeration — closes the follow-up left open by 1.3.0. All 22 toolsets enumerated via `list_available_toolsets` + `get_toolset_tools` on a connected Cloudways account:
- **Alias question resolved: there are no aliases.** Every primary tool name in the catalog matches the live `tool_name` byte-for-byte. The endpoint-style aliases the official tools article lists for the Security and Service categories (`security_dns_create`, `security_whitelisted_ips_update`, `service_state_update`, `service_varnish_manage`, …) **do not exist on the live server**. The "pending live-MCP re-enumeration" banner is removed.
- **Removed 64 phantom identifiers** the article documents but the live MCP does not expose: the entire **Bot Protection** (12), **Client Billing & Reporting** (20), **CloudwaysCDN legacy** (7), and **Lists API** (10) categories, plus `oauth_access_token_generate`, `security_csr_create` / `security_csr_get`, and the alias notes. Follow-on fixes: SKILL.md's write-ops list now cites only `agency_os_*` (not `billing_*`), installation.md drops the `oauth_access_token_generate` pointer, workflows-maintenance §3 no longer promises CSR tooling (generate the CSR with `openssl`; custom-cert install stays UI/direct-API), and `server_package_update` points at the raw `/packages` endpoint instead of a nonexistent `packages_list`.
- **Exact counts confirmed**: **244 tools = 241 across 22 toolsets + 3 meta-tools** — the v1.2 announcement figure is precise, not approximate. Added a per-toolset count table. The 65-visible design (62 direct + 3 meta) is independently verified.
- **New: `safe_update` toolset documented** — declared by the server but empty in the current build (0 tools), so SafeUpdate managed WordPress updates remain UI-only.
- **New caveat: toolset descriptions overstate their contents.** `list_available_toolsets` is reliable for routing but not as a capability contract — `staging_management`, `security`, `cloudways_bot`, `copilot`, and `agency_os` each advertise operations with no backing tool (e.g. `agency_os` claims 7 that don't exist). Verify with `get_toolset_tools` before promising a capability.
- Nothing was missing in the other direction: every live tool was already in the catalog.

## 1.3.0 - 2026-07-19
Synced with **Cloudways MCP v1.2** (verified against the official support articles + the v1.2 announcement, fetched 2026-07-19):
- **Authentication rewritten for role-based Access Tokens**: the MCP now authenticates with `X-Access-Token` + `X-Mcp-Host` (official host values: `claude-code`, `claude-desktop`, `cursor`, `windsurf`, `vs-code`, `gemini-cli`, `codex`, `codex-cli`). Tokens carry RBAC roles — READ / LIMITED / FULL ACCESS — and the skill now recommends least-privilege tokens per connection (READ for monitoring, per-client tokens in multi-account). The legacy `X-CW-Email`/`X-CW-Api-Key` API-key flow is documented as deprecated (API key EOL **2026-10-15**) across SKILL.md, installation.md, README, and `.mcp.json.example`. Removed the now-false "no granular permission at the MCP layer" claims.
- **Catalog expanded to MCP v1.2 (~244 tools)**: added the new categories from the official tools article ([15798823](https://support.cloudways.com/en/articles/15798823-cloudways-mcp-server-tools)) with R/W/W! risk tags — Security (SSL/Let's Encrypt + IP whitelisting), Security Suite (anti-malware/WAF, 26 tools), Bot Protection, CloudwaysBot alerts, CloudwaysCDN (legacy), Staging Management, Team Members, Server Transfer, Supervisord, Copilot subscription, Client Billing & Reporting, AgencyOS, Cloudflare expansion (12 tools), Add-on expansion (Elastic Email, upgrades), Lists API, and `oauth_access_token_generate`. Flagged the v1.2 sections as pending live-MCP re-enumeration, and noted endpoint-style alias names the article lists for security/service tools.
- **Closed the former "honest gaps"**: SSL/Let's Encrypt, SSH/MySQL IP whitelisting, and team management are MCP tools now — workflows-maintenance §2/§3/§7, workflows-monitoring, and workflows-onboarding rewritten to use `security_*` / `team_member_*` tools (with the replace-not-append warning on `security_update_whitelisted_ips` and W! double-confirms on cert revoke / member removal). Remaining real gaps kept honest: SSH-key listing, backup listing, `customer_info`.
- Node requirement for the `mcp-remote` bridge bumped to v24+ per the setup article; documented the intentional 65-visible-tools design (62 direct + 3 meta-tools) so a "missing" tool isn't misread as a caching bug; added a "Last verified" marker to the catalog; automation workflows now mark the `email+api_key` OAuth exchange as legacy.

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
