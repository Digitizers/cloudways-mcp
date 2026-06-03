# Tools Catalog — Cloudways MCP

Tools catalog compiled from an earlier community MCP implementation. Classified by functional category, with flags **R** (read-only) / **W** (write — requires confirmation) / **W!** (write destructive — requires double confirmation).

> ⚠️ **Read before relying on this list:**
> - The tool names here are from the **community** server. The **official** Cloudways MCP may have different names/capabilities — **the live server is the source of truth**. Always check the `mcp__cloudways*__*` tools actually available.
> - **read-only vs write uncertain:** sources conflict on whether the current version includes write operations or only read. The W flags here are a cautious assumption, not a guarantee. If a write tool does not exist on your server — simply do not use it. If it does — it must pass confirmation.

---

## 1. Authentication & Account info

| Tool | Type | When to use |
|------|-----|------------|
| `ping` | R | Liveness check of the MCP endpoint. Useful after a config change. |
| `customer_info` | R | Account details, plan, balance, support tier. A good starting point for any audit. |
| `rate_limit_status` | R | How many calls remain until reset. Useful before a batch operation. |

---

## 2. Discovery (which resources can be allocated)

All these tools are R. Useful mainly before creating a new server/application (because they return the API's capabilities: providers/regions/sizes/apps that are supported).

| Tool | Returns |
|------|--------|
| `get_available_providers` | DigitalOcean, AWS, GCP, Vultr, Linode |
| `get_available_regions` | Available regions per provider |
| `get_available_server_sizes` | RAM/CPU/disk options per provider |
| `get_available_apps` | WordPress, Magento, Laravel, custom PHP, etc. — including versions |
| `get_available_packages` | Software packages (PHP versions, MySQL/MariaDB, Redis, etc.) |
| `get_ssh_keys` | List of SSH keys stored in the account |

---

## 3. Servers — list & inspect (read-only)

| Tool | Returns |
|------|--------|
| `list_servers` | All servers: ID, label, provider, region, size, status, IP, master credentials |
| `get_server_details` | Details of a specific server: app list, ssh user, db credentials, ssl config |
| `get_server_settings` | Configuration (PHP timeout, memory limit, upload_max_filesize, etc.) |
| `get_server_disk_usage` | Space breakdown by directory — useful for locating a log/cache that has ballooned |
| `get_server_services_status` | Status of Apache, Nginx, MySQL, Memcached, Varnish, Redis |
| `get_server_monitoring_detail` | Detailed metrics: CPU, RAM, disk I/O, network |
| `get_server_analytics` | Analytics data for the server (traffic, uptime) |

---

## 4. Servers — power & lifecycle (W — requires confirmation)

⚠️ **All of these affect *every* application on the server.**

| Tool | What it does | Risk |
|------|---------|--------|
| `start_server` | Starts a stopped server | If the server was stopped intentionally — check why |
| `stop_server` | Shuts down (downtime for all applications) | All sites go down. Be careful. |
| `restart_server` | Full reboot | Temporary downtime (1-5 minutes) for all applications |
| `backup_server` | Full snapshot | Takes time + space. Usually billed. |
| `optimize_server_disk` | Cleans up logs and temp files | Can delete important logs. Check policy. |
| `change_service_state` | enable/disable Apache/Nginx/MySQL/etc. | Downtime if disabling a critical service |
| `manage_server_varnish` | enable/disable/purge Varnish at the server level | Can break applications' cache strategy |

---

## 5. Applications — inspect (read-only)

| Tool | Returns |
|------|--------|
| `get_app_details` | URL, FQDN, app folder, SSH user, DB credentials, SSL config |
| `get_app_credentials` | List of additional credentials (SFTP, etc.) |
| `get_app_settings` | PHP-specific overrides, Cron jobs, environment |
| `get_app_monitoring_summary` | bandwidth, requests, response time — summary |
| `get_app_analytics_traffic` | traffic detailed (visitors, page views) |
| `get_app_analytics_php` | PHP metrics (slow scripts, memory) |
| `get_app_analytics_mysql` | MySQL metrics (slow queries, locks) |
| `get_app_varnish_settings` | Varnish config at the application level |

---

## 6. Applications — lifecycle & data (W — requires confirmation)

| Tool | What it does | Risk |
|------|---------|--------|
| `clone_app` | Creates a new copy on the same server or another | Resources + DB copy |
| `backup_app` | Snapshot of files + DB | Time + space |
| `restore_app` | Restores from a backup | **Overwrites the current state**. Back up first. |
| `rollback_app_restore` | Reverts after a restore | Only within a limited window after the restore |
| `clear_app_cache` | Clears page cache, varnish | First traffic will be slow — but usually safe |
| `reset_app_file_permissions` | Resets chmod/chown to default | If manual modifications exist — you'll lose them |
| `enforce_app_https` | Adds an HTTP→HTTPS redirect | Can break if SSL is invalid; check SSL first |
| `manage_app_varnish` | enable/disable/purge at the app level | Can break the caching strategy |

---

## 7. Domains & CNAME (W — careful)

| Tool | What it does | Risk |
|------|---------|--------|
| `update_app_cname` | Updates/adds a CNAME for the application | If DNS is not ready — you break the site |
| `delete_app_cname` | Removes a CNAME | **Immediate destruction of a production application**. Requires double confirmation. |

> Recommended process: never update a CNAME in Cloudways before verifying:
> 1. The source DNS (Cloudflare, Route53, etc.) points correctly
> 2. A low TTL was already set 24h earlier
> 3. There is no live traffic at the moment (check via analytics)

---

## 8. SSL — Let's Encrypt

| Tool | Type | What it does |
|------|-----|---------|
| `install_letsencrypt` | W | Installs LE for a domain/application |
| `renew_letsencrypt` | W | Manual renewal |
| `set_letsencrypt_auto_renewal` | W | Enables auto-renew (always recommended) |
| `revoke_letsencrypt` | **W!** | **Revokes the certificate — the site may immediately throw an SSL error** |

## SSL — Custom certs

| Tool | Type | What it does |
|------|-----|---------|
| `install_ssl_certificate` | W | Installs a custom cert (cert + private key) |
| `remove_ssl_certificate` | W | Removes a custom cert (reverts to default / blocks SSL) |

---

## 9. Security & Access control

| Tool | Type | What it does |
|------|-----|---------|
| `get_whitelisted_ips_ssh` | R | List of IPs allowed for SSH |
| `get_whitelisted_ips_mysql` | R | List of IPs allowed for external MySQL |
| `update_whitelisted_ips` | W | Updates the list (can lock yourself out if you make a mistake) |
| `check_ip_blacklisted` | R | Checks whether a given IP is blocked by Cloudways |
| `allow_ip_siab` | W | Grants access to SIAB (Server-in-a-Box / DB tools) |
| `allow_ip_adminer` | W | Grants access to Adminer (web DB GUI) |

> **Flag:** `update_whitelisted_ips` can block yourself. Always make sure your IP **remains** in the new list.

---

## 10. Git deployment

| Tool | Type | What it does |
|------|-----|---------|
| `generate_git_ssh_key` | W | Creates a new key pair |
| `get_git_ssh_key` | R | Returns the public key (to add to GitHub/GitLab) |
| `git_clone` | W | Initial clone of a repo to the application |
| `git_pull` | W | Pulls all changes (can break if there's a conflict) |
| `get_git_deployment_history` | R | History — useful for pinpointing when something broke |
| `get_git_branch_names` | R | List of branches |

> Cloudways' Git deployment is not a replacement for full CI/CD. It's good for client sites that don't require a build step. For complex sites — stick with GitHub Actions / Vercel / wpe.

---

## 11. Team & Projects

| Tool | Type | What it does |
|------|-----|---------|
| `list_projects` | R | Projects in the account (organization of servers/apps) |
| `list_team_members` | R | Invited users + permissions |
| `get_alerts` | R | Open alerts (disk full, service down, etc.) |

---

## 12. Organized by usage scenario (quick cross-reference)

### End-to-end account check
`customer_info` → `list_servers` → `get_alerts` → `rate_limit_status`

### Application health check
`get_app_details` → `get_app_monitoring_summary` → `get_server_services_status` → `get_app_analytics_traffic`

### SSL renewal before expiry
`get_app_details` (check expiry) → `renew_letsencrypt` (W) → `get_app_details` (verify)

### New client audit
See `workflows-onboarding.md`

### Disk full warning
`get_server_disk_usage` → identify the culprit application → `clear_app_cache` (W) → if not enough → `optimize_server_disk` (W)

### Cache nuke
`clear_app_cache` (W) + `manage_app_varnish` with action=purge (W)

### IP whitelist (development / debugging)
`get_whitelisted_ips_ssh` → `update_whitelisted_ips` (W — make sure you don't delete yourself)

---

## Tools that don't exist (as of now) — workarounds

| What's missing | Alternative |
|---------|--------|
| `create_server` | Cloudways UI only. The public API does not expose provisioning via MCP as of now. |
| `delete_server` | UI / direct API with curl, not via MCP. Safety. |
| `transfer_app` between servers | UI only. |
| `staging_url` toggle | Part of `get_app_details` (existing feature). |
| Application creation on an existing server | Partial — check whether `add_app` exists in your version. |

If a task requires an action that isn't in the MCP, you can still use the Cloudways REST API directly with curl + the credentials. The MCP is a convenient wrapper, not the only one.
