# Changelog

## 1.5.3 - 2026-09-13

From the ClawHub audit of 1.5.2. AIG went from three Highs to **one Medium**, and `static-analysis`
stayed clean. The Medium is fair and this release takes it whole.

- **`app_get` and `server_get` leave every step that only needed metadata** (Medium — "credential-
  bearing API calls exceed the metadata requirements of routine workflows"). Eight maintenance
  and monitoring sequences opened with `app_get — confirm target` or `server_get — current
  state`: a cache purge, a backup, a restore, a custom-cert install, a change baseline, a
  multi-server comparison, a "why is this server not Running". None of them used the database
  or master credentials those calls return beside the label; they were the habit of reaching for
  the richest tool. Each now names the credential-free call that answers the actual question —
  the held roster for identity (an id alone is not a confirmed target — the confirmation block
  needs name + URL, and a mistyped id that belongs to another app is still valid), `service_status`
  and that server's insights for "why" (`operation_status` only for an operation id you already
  hold — it takes no server id), the `server_list`
  row for a baseline, `monitoring_app_summary` and `app_settings_get` for what an app is doing —
  and `workflows-maintenance.md` states the rule once at the top as a ladder, narrowest exposure
  first: the roster you already hold (no call), then an external lookup or one `app_get` for an
  id you already know — narrower than `app_list`, which covers every app on the server — then
  `app_list` for a name, then ask. What `app_get` is never for is habit. The same ladder governs a
  known server: the `server_list` response already held, else the UI, else one `server_get` for
  that server, with its full scope stated (master credentials *and* every hosted application's
  row, per rule 7) — never a fresh account-wide `server_list` to read one row. The three places that read **certificate state** through `app_get` read it from the
  outside instead, as two commands with two jobs: `env -u CURL_CA_BUNDLE -u SSL_CERT_FILE -u SSL_CERT_DIR curl -q --cacert ~/.config/cloudways-mcp/cacert.pem -sS -o /dev/null --noproxy '*' https://<domain>/`
  (`-q` first, so a `~/.curlrc` saying `insecure` cannot weaken it, and the `env -u` prefix so a CA
  override in the environment — which `-q` does not touch — cannot either; both measured, the second
  by pointing `CURL_CA_BUNDLE` at `self-signed.badssl.com`'s own certificate and watching exit 60
  become exit 0; and `--cacert` pinned to a copy of Mozilla's public roots fetched from curl.se's **dated** URL for that revision (the undated name moves with every Mozilla update and would stop matching), downloaded to a temporary name and moved into place only after it verifies against a digest **recorded in the skill** (a same-origin `.sha256` proves only that the transfer was intact, and whatever can replace the bundle can replace it), because a
  corporate CA in the OS store is beyond `env -u`'s reach — an empty bundle exits 77, proof the file
  is the trust source)
  is the **verdict** — exit 0 means chain, hostname and validity passed the OS trust store, exit
  60 means one did not — and `openssl s_client … | openssl x509 -noout -issuer -dates` supplies
  the dates for a report and decides nothing. Measured: the openssl line prints issuer and dates
  and exits 0 for `expired.badssl.com`, `self-signed.badssl.com` and `wrong.host.badssl.com`
  alike, while `curl` exits 60 for each. The "enforce HTTPS" step gates its write on the
  verdict, because redirecting production onto a certificate browsers reject is the failure the
  step exists to prevent — and it is the **origin** it checks, `--resolve`d to the server's IP:
  through public DNS the same command validates a CDN's edge certificate, which says nothing
  about the certificate on the Cloudways app, and enforcing HTTPS at an origin behind a proxy in
  Flexible mode is a redirect loop. Renewals are verified at the origin for the same reason, and the origin
  checks carry `--noproxy '*'`: with `HTTPS_PROXY` in the environment curl hands the request to
  the proxy, which reaches the edge, and the origin check silently becomes an edge check (measured). And before the redirect is enabled,
  the HTTPS answer itself is inspected (`-w '%{http_code} %{redirect_url}'`): an application that
  answers `https://` with a redirect to `http://` — WordPress with `home`/`siteurl` still on HTTP —
  passes every certificate check and loops the moment the server redirect goes on, so the
  WordPress fix moved from a note *after* the write to a gate *before* it, and verification now
  follows the whole chain (`-L --max-redirs 5`; `curl: (47)` is the loop) instead of one hop —
  the preflight too, since a harmless `https://www.` first hop can hide an `http://` second one —
  and both chain checks pass on curl's exit status and an `https://` effective URL, not on a `200`,
  because a `401` behind Basic Auth or a `403` from a WAF is a perfectly good answer over HTTPS — but
  never on a 5xx, since Cloudflare's 525/526 are the edge reporting a failed TLS handshake to the
  origin and arrive as `exit 0` (measured),
  and with `--noproxy '*'` like the origin checks, so an intercepting proxy's own page cannot pass
  them —
  and with `--proto-redir '=https'` so a chain that dips to HTTP and climbs back — invisible to
  `%{url_effective}` — is refused at the hop (`curl: (1) Protocol "http" disabled (in redirect)`,
  measured). Taking the redirect back off after a failed verification is named as the second
  production write it is, with its own **CONFIRM** — the confirmation that enabled it does not carry. An
  app id with no server is stated to be unresolvable — `app_list` and `app_get` both take a
  `server_id`, and so does every app-scoped read — so the answer is to ask, never to walk every
  roster. Whether a CDN or proxy sits in front is decided by DNS — A and AAAA both, address lines only, asked of a **public resolver** (`dig @1.1.1.1`) so split-horizon DNS on a VPN cannot hide a CDN behind a local answer, filtered with `awk` rather than `grep` — and guarded so that an empty answer (resolver blocked or erroring: `dig` exits 9, the pipeline exits 0 and prints nothing, measured) fails the gate instead of vacuously passing it, and each family's lookup is checked on its own exit status so one failed family cannot hide behind the other's good answer, and on its RCODE, since `dig
  +short` is silent and exits 0 on a `SERVFAIL` (measured against `dnssec-failed.org`), so a name with no AAAA record does not
  exit 1 under `set -e`, since `dig +short` prints a CNAME's canonical name on its own line — against the server's own addresses, not by
  comparing certificate issuers (edge and origin can both be Let's Encrypt). The one field
  `app_get` alone returns that a routine job can need — the application's folder name, for
  attributing a large directory in a disk investigation — keeps a targeted call for the one app
  in question, named as the rule-7 case; the first pass ranks by `monitoring_app_summary`
  `type: db`, which is **disk** size per the live tool's own description — not the database — and
  the catalog row now says so, since reading it as the database would misattribute a media-heavy
  site with a small one, and the disk step no longer calls its path credential-free: the roster
  is what costs, it follows the same ladder, and the size calls add nothing on top of it. In the
  enforce-HTTPS sequence, a missing certificate routes to the
  install step and back rather than to a dead stop; the stop is only on the write itself. The
  preflight runs once per **served hostname** — the write covers the whole application, and an
  alias with no matching certificate or behind a Flexible proxy breaks the moment the redirect goes
  on; aliases come from the UI or a filtered direct call, since `app_list` carries the primary
  domain only and the alias tools are all writes. The resolution ladder says the same for a user
  who names a site by a secondary domain: an alias lookup through `app_list` would pay the
  roster's cost and find nothing. The WordPress `home`/`siteurl` repair that the preflight can call
  for reads the current values and changes only the scheme — writing `https://<hostname>` from the
  step's placeholder would have promoted an alias to canonical — and is its own confirmed write with a
  backup first, ahead of the redirect's own confirmation; and the post-write verification runs per
  hostname like the preflight. `curl: (47)` is diagnosed apart from an HTTP downgrade: with
  `--proto-redir '=https'` an HTTP hop is never followed, so 47 is an HTTPS-only loop or an over-long
  chain, and the WordPress scheme repair is reserved for the protocol-disabled error or an observed
  `http://` `Location`. The change baseline makes no roster call for an app-scoped change whose
  target is already known — `app_list` for one known id imports every application's row for nothing. The DNS/proxy gate now comes before the certificate check and
  decides which certificate matters: a site visitors reach directly must pass the OS trust store
  at the origin, while a site behind a proxy in Full / Full (strict) is judged by the proxy's own
  origin policy (a Cloudflare Origin CA certificate is correct there, and the local check would
  have called it invalid) with the edge certificate as the one that must pass — and proxying is judged per DNS answer, not
  per hostname: an origin A record beside a proxied AAAA record puts the hostname on both
  branches, because the direct family's browsers are handed the origin certificate no matter what
  the proxy trusts. And "confirm the target" means the roster you already hold or the id
  you were given — `app_list` only when you have neither, as one call whose payload rule 7
  describes; the first draft of this release called that sequence credential-free, which is
  the claim 1.5.1 removed, and it is gone again. The
  "checking an app's details" pattern in SKILL.md, the one most likely to be copied, no longer
  ends in `app_get` either. After this, three bounded in-agent `app_get` calls remain, each named where it is
  used: one for an app id you already know when no roster is held (narrower than `app_list`,
  which covers every app on the server); one per candidate, bounded by the size ranking, when a
  disk investigation cannot attribute a folder from sizes alone; and one for a single
  certificate the user named. None of them is a sweep, and none of them is a habit.
- **Every public answer is checked, not the first one curl reaches.** The edge handshake, the
  redirect-chain preflight and the post-write verification in "enforce HTTPS" each ran one curl
  request against the hostname, and one request exercises **one** address: curl races the A and
  AAAA answers and keeps the first connection to succeed (measured on a dual-stack hostname —
  three plain requests all connected to the same IPv6 address and never touched IPv4). A proxied
  hostname whose IPv6 edge serves an expired certificate therefore passed on its IPv4 edge, and
  the write sent that family's visitors onto the broken path. All three checks now loop over the
  DNS gate's `$A` with `--resolve <hostname>:443:<answer>` per answer (IPv6 accepted as is, with
  or without brackets — measured), print `<ip> <status> …` per line and `<ip> FAILED (curl exit N)`
  on failure, and pass only when every line passes. The post-write check, which starts from
  `http://`, pins `:80` as well as `:443` — `--resolve` is per host:port, and pinning `:443`
  alone left the `http://` hop, the one the write changed, on whichever address curl reached
  first (measured). The pin covers the hostname under test only: a `www.` hop resolves publicly
  and is checked in its own turn, since every served hostname is in step 0's list.
- **The post-write chain check allows one more hop than the preflight.** The preflight in
  "enforce HTTPS" starts at `https://` with `--max-redirs 5`; the verification after the write
  starts at `http://`, one hop earlier — the `http→https` hop the write adds — and kept the same
  limit, so a healthy five-hop chain that passed the preflight was called a loop after the write
  and routed to rollback (measured: a six-hop chain exits 47 at `--max-redirs 5` and 0 at 6). It
  now uses `--max-redirs 6`, exactly one more than the preflight, and reads 47 as a chain that no
  longer settles or grew by more than that hop.
- **An app-scoped baseline measures the known app.** The change baseline skips the roster when
  the change is scoped to one application, but its app-level steps still said "each application
  from step 2" — read literally, no app summary or traffic was captured at all. They now name
  the known target app for the app-scoped case and the step-2 roster for the server-wide one.
- **An app-scoped baseline never falls through to `server_get`.** Its step 1 wanted the server's
  row and offered the ladder held `server_list` → UI → one `server_get`, so a known
  `(server_id, app_id)` with no held row ended in a `server_get` — the master credentials plus
  the very roster step 2 refuses to fetch for that case — for a row no later step reads: steps
  3–5 take the server id and steps 6 and 8 the pair. The app-scoped case now proceeds from the
  ids, with the descriptive fields taken from a held response or the UI if the record wants
  them; the ladder, `server_get` rung included, is stated for the server-wide case only, where a
  `server_get` also supplies the step-2 roster so no `app_list` follows it.

## 1.5.2 - 2026-09-13

From the ClawHub audit of 1.5.1. `static-analysis` went `suspicious` → **clean** (the
`exposed_secret_literal` false positive is gone with the line it pointed at) and VirusTotal is
clean. AIG returned three Highs, all three real, all three fixed here — and two of them are
mine from 1.5.1.

- **The weekly SSL sweep no longer runs through the agent** (T05, High). `workflows-monitoring.md`
  §5 walked the fleet with `app_get` to read certificate expiry dates — a tool that returns that
  application's **database credentials** in the same payload. 1.5.1 authorized that loop under
  conditions ("on a READ-role token, do not paste the responses anywhere"); that was the wrong
  call, and conditions on a fetch do not unfetch anything. The dates are now collected **outside
  the conversation** — the Cloudways UI, or a direct call through a field filter — and only names
  and dates come back to the agent for triage — which also means the agent calls **no roster
  tool** in that section: it was handed the labels, and `app_list`/`server_list` come from the
  same `/server` payload safety rule 7 is about, so fetching one would give away the section's
  own guarantee for nothing. In the agent, `app_get` is for one certificate
  somebody named. `workflows-automation.md` already ran exactly this as a Sunday cron, so §5 now
  points at it — and that cron makes the **request and the projection in one step**, because on
  n8n or Make every node's output is persisted in the execution record: an HTTP node that emits
  the whole `/app/{id}` payload has already retained every app's DB password, and a filter node
  after it cannot take that back. The three shapes that actually work are named (a plain
  `curl`+`jq` script, one n8n Code node that performs its own requests, or a platform whose
  execution logging is off and verified off), along with when not to run the job at all. The
  headless daily summary in the same file **stopped asking for certificate expiry**: the only
  tool that answers it is `app_get`, so an agent given that line had no way to comply except
  the sweep this release removes — and its prompt no longer says “don’t include credentials in
  the summary”, which was the same too-late instruction in miniature. It now names the three
  tools not to call, and says why leaving them out of the summary would not have helped. The onboarding note that named §5 as
  the one legitimate exception is gone: there is no exception any more.
- **The Access Token leaves both the Desktop config and the command line** (T09, High). The
  bridge config passed `--header X-Access-Token:<token>`, so the token sat in
  `claude_desktop_config.json` as a literal **and** in the process's argument list, where any
  other user on the machine can read it from `ps`. It now lives in
  `~/.config/cloudways-mcp/headers.txt` at mode 600, written with `umask 077` and `read -rs` so
  it never reaches the terminal or the shell history — and written as a **new** file moved into
  place, because `umask` applies only at creation, so rotating a token by redirecting over an
  existing `headers.txt` would have kept whatever mode that file already had. The config carries
  `--header-file`
  pointing at it. The path is **outside** `~/.cloudways-mcp-bridge` deliberately: that directory
  is deleted and recreated by every re-install, so a token kept inside it would disappear on the
  next lockfile bump. Windows gets its own recipe (`Read-Host -AsSecureString` plus an `icacls`
  ACL that is the NTFS equivalent of `chmod 600`) and its own `args`, both documented from
  Microsoft's semantics rather than exercised. `mcp-remote` treats an unreadable header file as **fatal**, so a wrong path
  fails at startup instead of connecting unauthenticated. Verified against `mcp-remote@0.14.0`
  installed from the shipped lockfile: it logs `Loaded 2 header(s)` and the header **names**,
  never the value.
- **The shipped lockfile no longer carries a known-vulnerable `qs`.** GitHub's advisory database
  flagged two moderate issues against `qs < 6.16.0` in `bridge/package-lock.json` — the file this
  release tells people to `npm ci`. `express@4.22.2` is the newest 4.x and requires `~6.15.1`,
  with no release widening it, so `bridge/package.json` now carries
  `"overrides": { "qs": "6.16.0" }`; `body-parser` in the same tree already required `~6.16.0`, so
  the override merges two copies into the patched one. The regenerated lockfile differs by that
  one version and nothing else (82 entries → 81, every package with an integrity hash), `npm
  audit` reports **0 vulnerabilities**, and `npm ci` still yields a working `mcp-remote`.
  A lockfile that is reproducible but knowingly vulnerable is not "vetted".
- **The `npx` fallback is removed** (T08, High). It pinned `mcp-remote` and nothing underneath
  it: ~80 transitive dependencies re-resolved whenever the npx cache is empty, executing in the
  process that holds the Access Token. Anyone who can run `npx` can run the `npm ci` above it,
  against a lockfile that ships with the skill — the fallback bought convenience that was never
  worth its exposure. The troubleshooting row that sent people to check `npx` on PATH now sends
  them to re-run the install.

Not changed: the token is still a secret at rest, now in `headers.txt` rather than in the
Desktop config — a file that can be pasted into an issue or a screen share no longer carries it,
but the header file still must not be. ClawScan's `persistence_privilege` and `purpose_capability`
concerns are the skill's nature — an operations skill for a hosting account reaches production
servers, DNS and billing — and are answered by token role, not by documentation.

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
  credential-free, because “inventory” is not a synonym for “safe to paste”. That is a reduction and not a
  guarantee: this MCP exposes no roster endpoint documented to exclude credentials, so an app-level
  audit costs one such payload per server, and a job that can tolerate none at all is told to build
  the roster outside the conversation (UI, or a direct `GET /server` through a field filter) and
  bring back only ids and labels. The pass takes its detail from
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
