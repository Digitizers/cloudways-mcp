# Changelog

## 1.5.1 - 2026-09-13

From the ClawHub audit of 1.5.0. Context first, because the headline moved the wrong way: the
verdict went from `benign` to `suspicious`, but **AIG returned nothing at all for 1.4.1** (its
export was null) and returned four findings here — this is a scanner reporting for the first
time, not a regression introduced by 1.5.0. What 1.5.0 did change, the static-analysis pass
went from `suspicious` to `clean`. All four findings are real and all four are fixed.

- **`mcp-remote` is pinned** (High). The Claude Desktop bridge config launched
  `npx mcp-remote` with no version, so `npx` resolved and executed whatever the registry
  served at launch — and that config hands the package a live Access Token on its command
  line. Pinned to `0.14.0`, with npm's published digest recorded beside it, and a **committed lockfile** at `.claude/skills/cloudways-mcp/bridge/` — inside the published skill, because the ClawHub workflow and the npm `files` list both ship only that directory, so a lockfile at the repo root would not reach anyone who installed the skill — for pinning the ~80 packages underneath it — a top-level pin and a one-tarball digest do not cover the dependency graph, and generating your own lockfile resolves it at that moment, so a first install during a compromise freezes the bad version in. `npm ci` against the shipped lockfile resolves nothing of its own. **That install is now the default Claude Desktop configuration**, with `npx` demoted to a collapsed fallback that states what it does not cover — a safe path offered as an optional alternative to the copy-paste one is not the path anybody takes. The install removes its target directory first: `cp -R src dst` copies *into* an existing `dst`, so re-running after a lockfile update would nest the new lockfile one level down and `npm ci` would reinstall the stale graph and report success. Windows gets its own `.cmd` path and PowerShell setup (documented from npm's layout, not exercised on Windows). Plus the
  command to check what the registry actually served — computed from the bytes with
  `openssl`, because the line `npm pack` prints elides the middle of the value
  (`sha512-QBYGz02kc2Ahh[...]kpaEJdzlB/png==`), so comparing against it checks only a prefix
  and a suffix. Same class as the `hostinger-api-mcp` pin, in the
  repo next door.
- **Discovery no longer CALLS the credential-returning tools** (High). The onboarding sweep
  ran `app_credentials` for every application, plus `server_get` and `app_get`, which return
  master and database credentials in the same payload. Telling the agent to leave them out of
  the report does not help: by then they are in the transcript, and a transcript is kept,
  scrolled back through and sometimes pasted somewhere. The only way to keep a credential out
  of a conversation is not to fetch it. The pass now builds its inventory from `server_list`
  and `app_list`, which answer the inventory question in one call per account or per server
  instead of one per app — **not** because their payloads are known to be clean: the live
  server describes `app_list` itself as returning “ID, label, application type, version,
  domain, and credentials”, and both list tools are built from the same `GET /server` payload
  that makes `server_get` a credential tool. Safety rule 7 now says so rather than calling them
  credential-free, because “inventory” is not a synonym for “safe to paste” — and takes its detail from
  `server_settings_get`, `service_status`, `app_settings_get`, the monitoring/analytics tools
  and `app_vulnerabilities_list`. All three credential-returning tools are named as
  deliberately absent, with what each is for and when calling it is legitimate — **in every
  place the sweep is written down**: the stage instructions, the printable audit checklist at
  the end of the same file, the fleet-wide SSL sweep in `workflows-monitoring.md`, and the
  weekend health check in SKILL.md. A rule stated once and contradicted by the copy-pasteable
  checklist below it is not a rule. SKILL.md also carries it as a numbered safety rule, so it
  applies to sweeps nobody has written down yet. The restart preflight in
  `workflows-maintenance.md` now takes its "which apps go offline" roster from `app_list`
  rather than `server_get`, which returned the same list plus the server's master
  credentials. And the SSL claim is reconciled across files: there is no dedicated read
  tool, the detail rides inside `app_get` next to that app's database credentials, so the
  audit reads it from the UI while the certificate sweep — the one job that cannot be done
  another way — uses `app_get` under stated conditions. Removing `server_get` from the read
  paths also removed the app roster and the domain fields two sequences had been taking from
  it without saying so, so both are named again at their source: `app_list` for the roster
  (`server_list` returns an app *count*, not IDs) and for the primary `domain`, and the UI or
  direct API for additional domains/CNAMEs — every alias tool (`app_cname_update`,
  `app_cname_delete`, `app_aliases_update`) is a **write**, and a value is never read by
  calling a W tool.
- **The daily-summary example uses `mktemp`, not a fixed `/tmp` path** (Medium), with
  `umask 077` and a `trap` that removes the file even when `curl` fails. The template ends in
  the `X`s: BSD `mktemp` does not substitute them anywhere else, and does not fail either — it
  creates a file called literally `cw-summary.XXXXXX.md`, reinstating the predictable name
  behind a successful exit. Measured on macOS 26.3 (`mktemp ./cw-summary.XXXXXX.md` → exit 0,
  a 0600 file of that literal name), not inferred from the manual page. A predictable name in
  a shared `/tmp` can be pre-created as a symlink by another user.
- **…and builds its JSON with `jq -Rs`** rather than interpolating the file into a string. The
  audit called this output encoding; it is also a plain bug — the first quote, backslash or
  newline in a generated report breaks the payload. `jq` is declared as a dependency and
  checked up front (it is not installed by default on macOS, and `set -e` would otherwise kill
  the job at the encode step after all the work was done), with a `python3` one-liner given as
  the equivalent for anyone without it.
- **The Airtable state store gets an explicit field allowlist** (Medium). Syncing a response
  wholesale carries credentials into a third-party store with its own sharing and retention,
  where they outlive both rotation and the engagement. Same note applied to the Slack and
  email destinations.

No tool, endpoint, auth-header, or safety-rule changes.

## 1.5.0 - 2026-09-13

From the ClawHub audit of 1.4.1. ClawScan rates the skill **benign**; this is the
credential-handling issue its static-analysis pass surfaced (as a false-positive
"exposed secret literal" on an angle-bracket placeholder — the placeholder is not a secret,
but the line teaching users to type a token into a shell is a real problem).

- **The token never goes on a command line.** The setup examples put the Access Token
  literally into `claude mcp add --header "X-Access-Token: <token>"`, which exposes it three
  ways: the shell history file, `ps` / `/proc/<pid>/cmdline` for every other user on the
  machine while the command runs, and the resolved value `claude mcp add -s user` then
  stores in `~/.claude.json` in plaintext until the connection is removed. Every example now
  passes a **single-quoted placeholder** — `'X-Access-Token: ${CLOUDWAYS_ACCESS_TOKEN:-}'` —
  which closes all three: the shell never expands it, and Claude Code expands it from its own
  environment when it opens the connection. Verified rather than assumed: that expansion
  applies to `headers` in local- and user-scoped `~/.claude.json` entries, not only in a
  project `.mcp.json` (an unset reference reports `Missing environment variables` in
  `claude mcp list`).
- **Multi-account uses one variable NAME per account** through the same mechanism, so
  neither token reaches a config file, and `.mcp.json.example` carries the same placeholders
  rather than `CLIENT_A_ACCESS_TOKEN`-style literals — following the example can no longer
  produce the plaintext file the guidance exists to prevent.
- **New "Handling the token safely" section**: where the value should live (`read -rs` for
  one shell, a mode-600 profile or a keychain lookup for a persistent setup, cloud env vars),
  and revoke-and-reissue at platform.cloudways.com as the recovery for an exposed token,
  with the smallest role that works. The check for a forgotten literal **parses**
  `~/.claude.json` and prints connection names, never values — and counts literal material
  inside `:-` defaults, since a token hides just as well in `${VAR:-cw_live}`.
- **The Claude Desktop bridge config is called out as a credential at rest.** The `${VAR}`
  expansion above is Claude Code's; that file is documented as holding the literal, to be
  kept at mode 600 and given the smallest role that works.

No tool, endpoint, auth-header, or safety-rule changes.

## 1.4.1 - 2026-08-25
Re-aligned `references/installation.md` with the current support article (14654372):
- **`X-Mcp-Host: windsurf` → `Devin`.** Windsurf was renamed Devin; the article's own config snippet sends the capitalised `Devin`, and its FAQ states header values are case-sensitive — so the old value was a silent-failure trap.
- **Per-client config table added** for the four non-obvious shapes the article calls out: Devin's `serverUrl`, VS Code's `"servers"` block with a required `"type": "http"`, Gemini CLI's `httpUrl` (plain `url` there opens an SSE connection and fails), and Codex's separate `[mcp_servers.cloudways.http_headers]` TOML table. The article warns that incorrect variants "fail silently", so the trap is worth carrying inline rather than deferring to the link.
- **Cursor documented as native HTTP** — no Node and no `mcp-remote`; the bridge is a proxy-problem fallback only.

No tool-name, endpoint, auth, or safety changes: the endpoint (`https://mcp.cloudways.com/mcp/`), the `X-Access-Token` / `X-Mcp-Host` header pair, the RBAC roles, the 2026-10-15 legacy-API-key EOL, and the whole tool catalog were re-checked against the article and already matched. The article's troubleshooting row saying "Node.js v18+" contradicts its own v24.14.1 prerequisite; this skill keeps v24 in both places deliberately.

## 1.4.0 - 2026-07-21
Zero-config connection for cloud sessions and devices:
- **Committed `.mcp.json`** (secrets as placeholders only) — the connection reads its token from the `CLOUDWAYS_ACCESS_TOKEN` env var, so claude.ai cloud environments (which load the repo's `.mcp.json` from the clone and inject env vars from the environment config) and devices with the var in their shell get the tools with no per-machine setup. The `${CLOUDWAYS_ACCESS_TOKEN:-}` default keeps the config parseable when the var is unset — the connection then just shows as unavailable until the token is provided (a bare unset `${VAR}` would fail the whole config parse, per the Claude Code docs).
- The connection is named **`cloudways-env`**, not `cloudways` — project scope beats user scope on name collisions, so the committed config can never shadow a `claude mcp add -s user cloudways` connection and silently point writes at the wrong account (Codex round-1 P1).
- `.mcp.json` removed from `.gitignore` (the tracked file must never contain a real token); `.mcp.json.example` re-purposed as the multi-account reference shape — real tokens go to user scope (`claude mcp add -s user`) or env vars, never into the tracked file. installation.md's "git-ignored `.mcp.json`" alternative is gone for the same reason (Codex round-1 P1).
- The CI no-leak guard now scans the tracked `.mcp.json` / `.mcp.json.example` (plus an `access[_-]?token` pattern) so a real token pasted into them can't pass CI. Exemptions apply after stripping the grep path prefix (so the `example` filename can't wave a match through), and a JSON-parsing check validates the actual env/header values independent of line layout and charset — closing the split-across-lines and base64-token gaps.
- `.claude/settings.json` sets `enableAllProjectMcpServers` so the committed config is auto-approved once the folder is trusted — untrusted checkouts deliberately ignore the committed key (v2.1.196+) and prompt once.
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
