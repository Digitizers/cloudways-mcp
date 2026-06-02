# Workflows — Onboarding & Audit (agency client takeover)

Core scenario: a new client arrives with a site on Cloudways, and you need to build a complete picture — no crashes, no surprises at 3 AM.

> **Guiding principle:** On the first audit, **don't touch anything.** Just read. Full documentation of the current state first. Changes come after you understand the picture and have a structured plan.

---

## Stage 0 — Pre-flight (before calling MCP)

Confirm with the client:
- [ ] API access approved on their side
- [ ] New API key (don't share with their old team)
- [ ] If there are old team members who don't need access — add this at this point (not via MCP — UI only)

---

## Stage 1 — Account mapping

```
1. customer_info               → plan status, support level, billing status
2. list_servers                → all servers: count, providers, regions, sizes
3. list_projects               → how they're organized
4. list_team_members           → who has access and what level
5. get_alerts                  → everything currently open
6. get_ssh_keys                → who can log in via SSH
```

**Deliverables you record:**

| Field | Value |
|------|-----|
| Cloudways plan | (Starter/Growth/Enterprise?) |
| Number of servers | |
| Providers in use | (DO/AWS/GCP/Vultr) |
| Regions | |
| Total monthly $ Cloudways (estimate) | |
| Team members + permissions | |
| SSH keys (don't publish — only a count) | |
| Open alerts | |

---

## Stage 2 — Server mapping

For each server in the list:

```
1. get_server_details          → label, size, IP, master credentials, app list
2. get_server_settings         → PHP timeout, memory, upload limit, custom PHP
3. get_server_services_status  → what's running (Apache/Nginx/MySQL/Memcached/Varnish/Redis)
4. get_server_disk_usage       → current space + breakdown
5. get_server_monitoring_detail → CPU/RAM trends last 24h
```

**Red flags to watch for:**
- [ ] Disk > 80% → can't add content without risk
- [ ] Sustained CPU spike → performance problem
- [ ] PHP timeout < 60s → may break long operations
- [ ] memory_limit < 256M → WordPress will get stuck
- [ ] Memcached/Redis off → no object caching
- [ ] Varnish off → no page caching
- [ ] Server hosting 5+ apps from different sources → high blast radius

---

## Stage 3 — Application mapping

For each application (per the app list from the previous stage):

```
1. get_app_details             → URL, FQDN, app folder, DB credentials, SSL status
2. get_app_settings            → app-level overrides
3. get_app_credentials         → SFTP/additional access
4. get_app_monitoring_summary  → bandwidth, requests (to get a sense of scale)
5. get_app_analytics_traffic   → visitors at least last 7 days
6. get_app_analytics_php       → slow scripts? memory issues?
7. get_app_analytics_mysql     → slow queries?
8. get_app_varnish_settings    → cache configured?
```

**Deliverables table for each app:**

| Field | Value | red flag? |
|------|-----|-----------|
| App name | | |
| Primary domain | | |
| Additional domains/CNAMEs | | |
| SSL provider + expiry | | check auto-renew? |
| App type (WP/Magento/PHP/Laravel) | | |
| PHP version | | < 8.1 = upgrade needed |
| WP version (if relevant) | | < 6.0 = security risk |
| Active plugins (if WP) | manual | abandoned plugins? |
| DB size (rough) | | |
| Daily traffic (avg) | | |
| Daily bandwidth | | |
| Avg response time | | > 1.5s = problem |
| Backups schedule | | none = critical risk |
| Varnish enabled | | false = perf gap |
| Object cache | | none = perf gap |
| HTTPS enforced | | false = SEO+security gap |

---

## Stage 4 — Security audit

```
1. get_whitelisted_ips_ssh     → who can SSH?
2. get_whitelisted_ips_mysql   → who can access the DB directly?
3. get_ssh_keys                → stored keys
4. For each suspicious IP: check_ip_blacklisted
```

**Red flags:**
- [ ] SSH whitelist empty = open to the world (critical)
- [ ] MySQL whitelist empty = open to the world (security disaster)
- [ ] Old SSH keys whose owners are unclear
- [ ] The old team member still has access
- [ ] Access not only for client employees but also for employees who have left

---

## Stage 5 — Backups audit

Unfortunately: the Cloudways MCP does not expose `list_backups` directly. You need to go to the UI (or a direct API call).

**Questions to check manually:**
- Are automatic backups enabled? (Cloudways → Server → Backups)
- Frequency? (daily / every two days / weekly)
- Retention? (how many days back?)
- Is there an off-platform backup? (Cloudways backups are available only from within Cloudways — if the account is closed, they're lost)

**Standard recommendation:**
- Daily Cloudways backups, 7-day retention
- Weekly off-platform backup (UpdraftPlus to S3 / Wasabi / Cloudways → Drive)

---

## Stage 6 — Reporting to the client

The onboarding document must include (Hebrew):

### 1. Executive summary
- How many servers, how many apps, rough monthly spend
- The 3-5 most severe red flags you found
- General recommendation (priority order)

### 2. Detailed current state
- Tables for each server + each app
- Links / IDs in Cloudways

### 3. Recommended task list
By priority (P0/P1/P2):

**P0 — must be done within the coming week:**
- (security criticalities: expired SSL, open whitelist, etc.)

**P1 — must be done within the month:**
- (PHP/WP upgrades, backup strategy, performance)

**P2 — long-term improvements:**
- (CDN integration, Varnish tuning, monitoring setup)

### 4. Quote
Estimated hours per P (₪300/h). Everything documented.

### 5. SLA / operational routine
- Weekly monitoring (which queries will be run)
- Response to an alert (target time)
- SSL renewals — who is responsible

---

## Quick report template

```markdown
# Cloudways Audit — [Client Name]
Date: [YYYY-MM-DD]
Auditor: [your name]

## Summary
- Servers: X | Apps: Y | Monthly spend (gross): $Z
- Critical findings: [N items]
- Recommendation summary: [one paragraph]

## Red flags
| Severity | Issue | Impact | Effort to fix |
|----------|-------|--------|---------------|
| P0 | ... | ... | ... |

## Inventory
(tables from stages 2-3)

## Recommended actions
### Phase 1 (week 1) — P0 items
### Phase 2 (month 1) — P1 items
### Phase 3 (ongoing) — P2 items

## Quote
| Phase | Hours | ₪ |
|-------|-------|---|
| 1 | ... | ... |
| 2 | ... | ... |
| **Total** | | |
```

---

## Quick reference — Audit checklist (printable)

- [ ] **Account:** customer_info / list_servers / list_projects / list_team_members / get_alerts
- [ ] **Per server:** get_server_details / get_server_settings / get_server_services_status / get_server_disk_usage / get_server_monitoring_detail
- [ ] **Per app:** get_app_details / get_app_settings / get_app_monitoring_summary / get_app_analytics_traffic / get_app_analytics_php / get_app_analytics_mysql / get_app_varnish_settings
- [ ] **Security:** get_whitelisted_ips_ssh / get_whitelisted_ips_mysql / get_ssh_keys
- [ ] **Manual (UI):** Backup schedule + retention / Cloudflare integration (if any) / WP version (if WP) / Active plugins (if WP)
- [ ] **Document:** Red flags / Recommendations / Quote / SLA
