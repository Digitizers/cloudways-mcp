# Workflows — Maintenance (write operations)

Maintenance scenarios requiring write operations. **Every operation here requires explicit confirmation from the user before execution**, per the pattern in `SKILL.md`.

> **Basic rule:** Read before Write. Always read the current state before you change it. Both to verify the operation is needed, and so you have a baseline to roll back to.

---

## Standard confirmation pattern

Before **every** call to a W tool, display a block like this to the user and wait for an explicit "yes":

```
🔒 Confirm execution?
   tool: <tool_name>
   target: <server name + ID or app name + URL>
   parameters: <key params>
   expected impact: <what happens>
   risks: <what could go wrong>
   Continue? (yes / no / pause)
```

If the user wrote "yes" — execute. If anything else — ask for clarification.

For especially dangerous operations (W!): add a **second step**: "Type the server/application name to confirm". This ensures they are reading and not approving automatically.

---

## 1. Cache clear — basic

**When:** "The site isn't updating after a change" / "Admin screen shows an old version"

**Sequence:**

1. `app_get` — confirm the target app
2. `app_varnish_settings_get` — see if Varnish is active
3. **CONFIRM:** `app_purge_cache` (W)
4. If Varnish is active: **CONFIRM:** `varnish_app_manage` with action=purge (W)
5. Check the site in a browser (curl or manually)

**If the problem persists:**
- Check plugin caches (W3 Total Cache, WP Rocket, LiteSpeed) — these are not in Cloudways, they must be cleared from WP-Admin
- Check Cloudflare cache if in use — `purge everything` in the Cloudflare UI
- CDN caches

---

## 2. SSL — Let's Encrypt renewal

**When:** SSL is approaching expiry and there is no auto-renewal / auto-renewal failed

> **Covered by the MCP as of v1.2** via the security toolset: `security_lets_encrypt_install` (W), `security_lets_encrypt_renew` (W), `security_lets_encrypt_auto_renewal` (W), `security_lets_encrypt_revoke` (W!). For wildcard certs: `security_create_dns` + `security_verify_dns` handle the DNS-01 challenge.

**Sequence:**

1. `app_get` — check the domain and read what you can about the app's current state
2. Check that the DNS still points to the server (critical for LE validation)
3. **CONFIRM:** `security_lets_encrypt_renew` (W) — or `security_lets_encrypt_install` (W) if no cert was issued yet. For a wildcard domain: **CONFIRM** `security_create_dns` (W), publish the returned TXT record at the DNS host, then **CONFIRM** `security_verify_dns` (W).
4. **CONFIRM:** `security_lets_encrypt_auto_renewal` (W) — turn auto-renewal on if it wasn't active.
5. `app_get` — re-read and verify the app looks healthy; confirm the cert in the browser

**If renewal fails:**
- Most common problem: DNS doesn't point correctly, or wildcard domains aren't configured
- Second: Let's Encrypt rate limit (5 attempts per week per domain)
- Third: HTTP-01 validation fails because the site is behind a Cloudflare proxy → resolve with DNS-01 or disable the proxy temporarily

---

## 3. SSL — installing a custom cert

**When:** The client purchased a cert from another CA (DigiCert, Sectigo, etc.), not Let's Encrypt

> **Partially covered by the MCP as of v1.2:** `security_csr_create` (W) / `security_csr_get` (R) generate and retrieve the CSR, and `security_remove_own_ssl` (W!) removes an installed custom cert. The **install/paste step itself** is still done in the **Cloudways Platform UI** (Application → SSL Certificate → Custom SSL) or via the [direct API](https://developers.cloudways.com/) — verify against the live MCP, and prefer the UI if no install tool is exposed.

**Sequence:**

1. Collect from the client: certificate, private key, ca bundle. (Only if the CA still needs a CSR from the server: **CONFIRM** `security_csr_create` (W), retrieve with `security_csr_get`.)
2. `app_get` — confirm target
3. **Install the custom cert in the Cloudways UI** (paste cert + key) — check the live MCP first; if no install tool is exposed, this step stays manual.
4. Check SSL from the browser (SSL Labs grade A+ preferred)
5. If Let's Encrypt was active — decide: keep as backup or revoke (`security_lets_encrypt_revoke`, W! — double-confirm)

> Warning: Installing a custom cert **cancels** the Let's Encrypt cert if one was active. Make sure you have the custom cert in hand **before** you start.

---

## 4. Backup before a change

**When:** Before a migration, restore, significant plugin update, or any "I'm not sure what this will do"

**Sequence:**

1. `app_get` — confirm target
2. `monitoring_app_summary` — before: snapshot of state
3. **CONFIRM:** `app_backup` (W)
4. Check that the backup is progressing (`app_backup_status_get` for in-progress state, or via the UI). Note: there is no general "list backups" tool — the available restore points are visible in the Cloudways UI.
5. Record the backup timestamp — you'll need it for restore if something goes wrong

**Server-level backup:**
- `server_backup` (W) — slower, includes everything, more expensive
- Useful before a server-wide change (PHP upgrade, OS upgrade, package change)

---

## 5. Restore after an error

**When:** Something went wrong (bad deployment, hack, accidental delete)

**Sequence — critical to follow in order:**

1. **STOP** — don't do anything until you understand the scope of the problem.
2. `app_get` — current state
3. Check the list of available backups (via the Cloudways UI — there is no MCP "list backups" tool; `app_backup_status_get` only reports in-progress backup status)
4. **CONFIRM step 1:** "Is the backup from date X the point you want to roll back to?"
5. **CONFIRM step 2:** Type the app name to confirm restore
6. **CONFIRM:** `app_restore` (W!) — full overwrite of the current state
7. Check that the site works
8. If the restore itself made things worse: **CONFIRM (W!):** `app_restore_rollback` — returns the app to its pre-restore files + database. This is available only within the limited rollback window (see below); after that, fall back to restoring an earlier backup or the Cloudways UI.

> **Limited rollback window.** `app_restore_rollback` only works for a short time after the restore (a few hours / a day) — after that the pre-restore local snapshot is gone and rollback is no longer possible. Make sure the site works **on the same day** as the restore. (Note: `app_local_backup_delete` deletes that pre-restore snapshot immediately, which also forecloses the rollback.)

---

## 6. Restart server / service

**When:** memory leak, services stuck, or troubleshooting

**Priority order — try the quietest one first:**

1. `app_purge_cache` (W) per app — sometimes that's all it takes
2. `service_restart` (W) on a single service (e.g. restart MySQL only)
3. If that doesn't help: `server_restart` (W) — 1-5 minutes downtime for all the apps

**Before server_restart:**

1. **Mandatory:** `server_get` → list of apps
2. **Mandatory:** count active users (if relevant — a store site with open carts?)
3. **Double CONFIRM:** "The server hosts X applications — Y, Z, W. Each of them will be offline for X minutes. Continue?"
4. Execute
5. **VERIFY:** `service_status` after the restart

---

## 7. IP whitelist — SSH/MySQL

**When:** Adding a key for a team member, or removing access for an old IP

> **Covered by the MCP as of v1.2** via the security toolset: `security_get_whitelisted_ips` (R, SSH/SFTP), `security_get_whitelisted_ips_mysql` (R), and `security_update_whitelisted_ips` (W! — **replaces** the whole list, not an append). Web-SSH / Adminer access: `security_whitelist_ip_siab` / `security_whitelist_ip_adminer` (W).

**Sequence:**

1. `security_get_whitelisted_ips` (or `_mysql`) — read what's there **now**
2. Plan the new list — **including your own IP** (the update **replaces** the list; anything you omit is removed)
3. **CONFIRM:** Show the user: "The new list is: [...]. Does your IP X.X.X.X stay on the list? yes/no"
4. If the user is missing from the list — **stop and clarify**
5. **Double CONFIRM (W!):** `security_update_whitelisted_ips` with the complete new list
6. **VERIFY:** re-read the list, then try SSH immediately (if it doesn't work — Cloudways support to restore)

> **Nightmare scenario to avoid:** updating the whitelist + removing your own IP + no alternative SSH. The only way out — the Cloudways UI (panic) or a support ticket (time). Be careful.

---

## 8. Disk cleanup

**When:** disk usage > 80%

**Priority order:**

1. `server_disk_usage_fetch` (init) then `monitoring_server_summary` (read) — where's the space?
2. **CONFIRM:** `app_purge_cache` (W) for all the suspect apps
3. If not enough: **CONFIRM:** `server_disk_cleanup_*` (W) — Cloudways magic cleanup
4. If not enough: upgrade size (UI only) or manual SSH to clean logs
5. **VERIFY:** `server_disk_usage_fetch` + `monitoring_server_summary` again

> `server_disk_cleanup_*` may delete logs. If the client must keep logs (compliance, debugging), export them manually first.

---

## 9. Enforce HTTPS

**When:** An old site still running on HTTP / client wants an SEO/security boost

**Sequence:**

1. `app_get` — check that a valid SSL is installed
2. If there's no SSL: install one first — `security_lets_encrypt_install` (W, see sections 2 and 3). Enforcing HTTPS without a valid cert will break the site.
3. **CONFIRM:** `app_enforce_https_update` (W) — toggles the HTTP→HTTPS redirect (this is separate from installing the cert)
4. Check that the redirect works: `curl -I http://example.com` → 301 to HTTPS

> **WordPress quirk:** After enforcing HTTPS, you may need to update `WP_HOME` and `WP_SITEURL` in `wp-config.php` or via WP-CLI: `wp option update home https://example.com && wp option update siteurl https://example.com`. Otherwise — mixed content errors.

---

## 10. Git pull deployment

**When:** Deploying a branch to an application

**Sequence:**

1. `git_branches_get` — verify the branch exists
2. `git_history_get` — what the current state is
3. **CONFIRM:** `app_backup` (W) — always backup before a deploy
4. **CONFIRM:** `git_pull` (W) with branch + commit hash if relevant
5. **VERIFY:** Check the site
6. If something broke: `app_restore` (W!) to the backup from section 3

> Cloudways Git deploy doesn't run build steps. If the site requires `npm run build` / `composer install` — you'll need to do that manually over SSH after the pull, or keep build artifacts in the repo.

---

## 11. Varnish — configure/purge

**When:** Setting a cache strategy / performance tuning

**Sequence:**

1. `app_varnish_settings_get` — current config
2. Understand the policy: cache TTL, exceptions, purge rules
3. **CONFIRM:** `varnish_app_manage` (W) with a specific action (enable/disable/purge/configure). For changing Varnish settings, `app_varnish_settings_update` (W) is also available.
4. If purge: check that the cache is clean (`curl -I` to the URL → should be `X-Cache: MISS` on the first request)

> Varnish isn't compatible with every application. For WooCommerce or applications with session-heavy data, precise exclusion rules are needed. Always check after a change.

---

## Cleanup after a session

Before ending a conversation that included write operations:

1. Summarize what was done
2. Mention if backups were created and when they expire
3. Mention if there are operations that still require later verification (SSL renewal, cache propagation)
4. If something wasn't finished — document what remains open
