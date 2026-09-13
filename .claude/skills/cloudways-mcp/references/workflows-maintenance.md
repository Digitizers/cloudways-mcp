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

> **Confirming a target: the narrowest fetch that gives you name + URL, and often no fetch at
> all.** Every sequence below starts by making sure the right application is in hand, and the
> confirmation block above requires its name + URL — **an id alone is never a confirmation**: a
> mistyped id that happens to belong to another application is still a valid id, so a write
> confirmed against nothing but a number can land on the wrong site (`app_restore` is the one
> that cannot be undone). Resolve in this order, and stop at the first rung you can use:
>
> 1. **The roster you already hold** from this conversation — zero calls. This is the usual case.
> 2. **You know the server and the app id, hold no roster** — read the name + URL **outside the
>    conversation** (the application's page in the Cloudways UI, or a direct API call through a
>    field filter): zero secrets in the transcript. If it has to be the API from here, **one
>    `app_get` for that one app** is the narrowest call there is — it returns that app's
>    database credentials, and nothing else's. Do **not** reach for `app_list` to confirm one
>    known id: rule 7 describes its payload, and it covers **every** application on the server,
>    so it exposes strictly more than the `app_get` it would be standing in for.
> 3. **You know the server and only a name or URL** — `app_list` on that server is the one API
>    route (a single call; take the one row you came for and paste none of it), or the same
>    external filtered roster as rung 2.
> 4. **You do not know the server** — stop and ask which server, or for the name/URL. There is
>    no lookup from an app id to its server: `app_list` and `app_get` both take a `server_id`,
>    and the only API route from a bare id is reading every server's roster, which is the sweep
>    this skill refuses.
>
> What `app_get` is never for is **habit**: reaching for it as the opening step of every job,
> for a label a held roster already gives you, was the finding this section exists to close.
>
> **Certificate state is read from the outside, and the verdict and the dates are two different
> commands.** The verdict is `curl -q -sS -o /dev/null --max-time 15 https://<domain>/`: exit **0** means the chain, the hostname and the validity
> period all passed the OS trust store — what a browser checks — and exit **60** means one of
> them did not. `-q` is not optional and must come **first**: it stops curl reading `~/.curlrc`,
> and a machine whose curlrc says `insecure` would otherwise pass an expired, self-signed or
> wrong-host certificate with exit 0 — measured: against `expired.badssl.com` with such a file,
> exit 0 without `-q`, exit 60 with it.
>
> **And there may be two certificates.** Through public DNS that command validates whatever
> answers for the name — behind Cloudflare or any reverse proxy, that is the **edge**
> certificate, not the one installed on the Cloudways application. The **origin** is checked
> by pinning the name to the server's IP (from `server_list`):
> `curl -q -sS -o /dev/null --max-time 15 --noproxy '*' --resolve <domain>:443:<server-ip> https://<domain>/` — `--resolve` keeps the hostname for SNI and verification and only changes where the
> connection goes. `--noproxy '*'` is part of the command: with `HTTPS_PROXY` / `https_proxy` /
> `ALL_PROXY` set in the environment, curl hands the request to that proxy, which resolves
> `<domain>` through its own DNS and reaches the CDN edge — so the "origin" check would be
> validating the edge certificate after all (measured: with a proxy variable set, the
> `--resolve` form connects to the proxy address, not the server, until `--noproxy '*'` is
> added). **Whether something is in front is a DNS question, not a certificate one**:
> `dig +short <domain> A | grep -E '^[0-9.]+$'; dig +short <domain> AAAA | grep ':'` against the server's addresses from `server_list` — **every** routable answer, both
> record types, must be the server. The `grep`s keep only addresses: for a CNAME — an ordinary
> `www` alias — `dig +short` prints the canonical name on its own line before the address
> (`github.com.` then `20.217.135.5`, measured), and comparing that line against a server IP
> would call every alias a proxy. Any other address is a CDN or proxy, whatever certificate
> it shows; and a proxy reachable only over IPv6 (an A record at the origin, an AAAA at the
> edge) is still a proxy for every IPv6 client, so an A-only check is not a check. Comparing
> issuers proves nothing,
> because the edge and the origin can both hold Let's Encrypt certificates and still be two
> different machines with a Flexible-mode HTTP hop between them. Let's Encrypt renewals happen
> at the origin, so a renewal is verified there; and enforcing HTTPS at the origin behind a
> proxy in **Flexible** mode (proxy speaks HTTPS to the browser, HTTP to the origin) makes the
> origin redirect every proxied request back to HTTPS — a loop. Both checks are measured below
> where they matter. The dates for a report come from `openssl s_client -servername <domain> -connect <domain>:443 </dev/null 2>/dev/null | openssl x509 -noout -issuer -dates`, and that line is **informational
> only**: measured against `expired.badssl.com`, `self-signed.badssl.com` and
> `wrong.host.badssl.com`, it prints issuer and dates and exits 0 for all three, while `curl`
> exits 60 for each. Nothing below decides anything on the openssl line.

## 1. Cache clear — basic

**When:** "The site isn't updating after a change" / "Admin screen shows an old version"

**Sequence:**

1. Confirm the target — name + URL, from the first rung of the ladder at the top you can use (held roster → external lookup or one `app_get` for a known id → `app_list` for a name → ask); an id alone is not a confirmation
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

1. Domain from the roster you hold, server IP from `server_list`. The certificate being renewed
   lives on the **origin**, so check that one: `curl -q -sS -o /dev/null --max-time 15 --noproxy '*' --resolve <domain>:443:<server-ip> https://<domain>/` for the verdict (exit 0 / 60), and
   `openssl s_client -servername <domain> -connect <server-ip>:443 </dev/null 2>/dev/null | openssl x509 -noout -issuer -dates`
   for the issuer and `notAfter` on record. If `dig +short <domain> A | grep -E '^[0-9.]+$'; dig +short <domain> AAAA | grep ':'` answers with anything that is not one
   of the server's own addresses, a proxy is in front — the renewal still happens here, at the origin, and
   the browser will keep showing you the proxy's certificate afterwards.
   Nothing in this step needs the database credentials `app_get` would add.
2. Check that the DNS still points to the server (critical for LE validation)
3. **CONFIRM:** `security_lets_encrypt_renew` (W) — or `security_lets_encrypt_install` (W) if no cert was issued yet. For a wildcard domain: **CONFIRM** `security_create_dns` (W), publish the returned TXT record at the DNS host, then **CONFIRM** `security_verify_dns` (W).
4. **CONFIRM:** `security_lets_encrypt_auto_renewal` (W) — turn auto-renewal on if it wasn't active.
5. Verify at the origin — the `--resolve` form of the check must exit **0** now, and the
   openssl line against `<server-ip>:443` should show the new `notAfter` — then load the site
   in a browser (which, behind a proxy, shows you the edge certificate, not this one). A re-read of `app_get` would confirm
   nothing the verified handshake does not, at the price of a second credential payload.

**If renewal fails:**
- Most common problem: DNS doesn't point correctly, or wildcard domains aren't configured
- Second: Let's Encrypt rate limit (5 attempts per week per domain)
- Third: HTTP-01 validation fails because the site is behind a Cloudflare proxy → resolve with DNS-01 or disable the proxy temporarily

---

## 3. SSL — installing a custom cert

**When:** The client purchased a cert from another CA (DigiCert, Sectigo, etc.), not Let's Encrypt

> **Barely covered by the MCP** (live-verified 2026-07-20): the only custom-SSL tool is `security_remove_own_ssl` (W!), which *removes* an installed cert. There is **no install tool and no CSR tool** — the article's `security_csr_create` / `security_csr_get` do not exist on the live server, and the `security` toolset description claiming a custom-cert install is wrong. Generate the CSR on the CA's side (or via `openssl`), then paste cert + key in the **Cloudways Platform UI** (Application → SSL Certificate → Custom SSL) or use the [direct API](https://developers.cloudways.com/).

**Sequence:**

1. Collect from the client: certificate, private key, ca bundle. If the CA still needs a CSR, generate it outside Cloudways (the MCP exposes no CSR tool) — and **name the key explicitly**, because the cert is only installable with the exact key that signed the CSR:

   ```bash
   # Signing with the key you already have:
   openssl req -new -key privkey.pem -out request.csr

   # Or generating a new key + CSR together (-nodes leaves the key
   # unencrypted, which the Cloudways UI requires):
   openssl req -new -newkey rsa:2048 -nodes -keyout privkey.pem -out request.csr
   ```

   Keep `privkey.pem` — a bare `openssl req -new` writes an encrypted key to whatever path the local OpenSSL config picks, and losing it makes the issued certificate unusable.
2. Confirm the target — name + URL, by the ladder at the top (held roster first; an id alone is not a confirmation)
3. **Install the custom cert in the Cloudways UI** (paste cert + key) — manual by necessity; no MCP tool covers this step.
4. Check SSL from the browser (SSL Labs grade A+ preferred)
5. If Let's Encrypt was active — decide: keep as backup or revoke (`security_lets_encrypt_revoke`, W! — double-confirm)

> Warning: Installing a custom cert **cancels** the Let's Encrypt cert if one was active. Make sure you have the custom cert in hand **before** you start.

---

## 4. Backup before a change

**When:** Before a migration, restore, significant plugin update, or any "I'm not sure what this will do"

**Sequence:**

1. Confirm the target — name + URL, by the ladder at the top (held roster first; an id alone is not a confirmation)
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
2. The app's identity — name + URL — by the ladder at the top (a bare id is not an identity,
   and step 5 below has to be checked against something), then `monitoring_app_summary` for
   what it is doing right now — the current state a restore decision needs
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

1. **Mandatory:** `app_list` → the applications about to go offline (`server_get` returns the same roster plus the server's master credentials; the roster is all this preflight needs)
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

1. Check that the **origin** serves a valid certificate: `curl -q -sS -o /dev/null --max-time 15 --noproxy '*' --resolve <domain>:443:<server-ip> https://<domain>/` must exit **0** — chain +
   hostname + dates against the OS trust store, at the Cloudways server itself. Exit 60
   (expired, self-signed, wrong host) means **stop**: enforcing HTTPS now redirects production
   traffic onto a certificate browsers reject. Then ask whether anything sits in front:
   `dig +short <domain> A | grep -E '^[0-9.]+$'; dig +short <domain> AAAA | grep ':'` — every answer, A and AAAA both, must be one of the server's own addresses from
   `server_list`. Any other address is a CDN or reverse proxy — regardless of what certificate
   it presents, even if its issuer matches the origin's, and even if only the AAAA record
   points at it (IPv6 clients would take that path into the loop) — and its origin mode has to
   be **Full (strict)**, or at least Full, confirmed in that proxy's own settings before this
   write. In Flexible mode the
   proxy reaches the origin over HTTP, the origin's new redirect sends it back to HTTPS, and
   the site loops. Do not read any of this off `openssl x509 -dates`, which prints dates for a
   broken certificate just as happily.
2. **The HTTPS answer must not send anyone back to HTTP — at any hop.** Step 1 validated the
   handshake and nothing after it: a valid certificate in front of an application that answers
   `https://` with `301 Location: http://…` still exits 0, and so does one whose first hop is a
   harmless `https://www.` canonical redirect while the **second** hop goes back to `http://`.
   So follow the whole chain, the way a browser will, **before** the write — and refuse
   **any** hop to HTTP, not just an HTTP ending, because `%{url_effective}` reports only the
   final URL and a chain that dips to `http://` and climbs back to `https://` would otherwise
   pass:
   `curl -q -sS -o /dev/null --max-time 15 -L --max-redirs 5 --proto-redir '=https' -w '%{http_code} %{url_effective} %{num_redirects}\n' https://<domain>/`.
   (Quote `'=https'` — in zsh, macOS's default shell, a bare `=https` is expanded as a
   command lookup and the line fails with `https not found`.) Pass is `200`, an `https://`
   effective URL, and a small hop count. `curl: (1) Protocol "http" disabled (in redirect)`
   (measured: exit 1, before curl ever connects to the HTTP target), an effective URL
   beginning `http://`, or `curl: (47) Maximum (5) redirects followed` — any of these means
   the app itself is pushing HTTPS visitors back to HTTP somewhere in its chain; on WordPress
   that is `WP_HOME` / `WP_SITEURL` still set to `http://`, the usual cause on a site that has
   never had HTTPS enforced. Enforcing now produces the loop the audit warned about: the server
   redirects `http→https`, the app redirects `https→http`, and every browser bounces between
   them until it gives up. **Fix the application first**, then re-run this step until it
   passes: `wp option update home https://<domain> && wp option update siteurl https://<domain>`
   (or the two constants in `wp-config.php`). Through public DNS on purpose — hops to other
   hostnames cannot be pinned with `--resolve`, and this is the path visitors take; the origin
   itself was already checked in step 1.
3. If there's no SSL: install one first — `security_lets_encrypt_install` (W, see sections 2 and 3). Enforcing HTTPS without a valid cert will break the site.
4. **CONFIRM:** `app_enforce_https_update` (W) — toggles the HTTP→HTTPS redirect (this is separate from installing the cert)
5. Verify the **whole chain** the way a browser walks it, not the first hop:
   `curl -q -sS -o /dev/null --max-time 15 -L --max-redirs 5 --proto-redir '=https' -w '%{http_code} %{url_effective} %{num_redirects}\n' http://<domain>/`.
   The start is `http://` on purpose — that is what the new redirect acts on — and
   `--proto-redir` governs only the hops after it, so every one of those must be HTTPS. Expect
   `200 https://<domain>/ 1` (or a small count if the app adds a `www` or trailing-slash hop).
   `curl: (47) Maximum (5) redirects followed` **is the loop**, and `curl: (1) Protocol "http"
   disabled (in redirect)` is a downgrade somewhere past the first hop. Either way the fix is
   to take the server redirect back off and return to step 2 — and that is a **second
   production write**: **CONFIRM:** `app_enforce_https_update` (W) with the standard block, the
   `target` and `expected impact` now describing the rollback. The confirmation given in step 4
   authorised enabling the redirect; safety rule 2 does not let it carry to disabling it, and a
   transient failure that looks like a loop must not roll production back to HTTP on nobody's
   say-so. Through public DNS on purpose: this is the path visitors take, proxy included.

> **WordPress, after:** with `home`/`siteurl` already on `https://` (step 2), what can remain is
> mixed content from hard-coded `http://` URLs inside posts and options — a search-replace job
> (`wp search-replace 'http://<domain>' 'https://<domain>' --dry-run` first), not a Cloudways
> setting.

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
