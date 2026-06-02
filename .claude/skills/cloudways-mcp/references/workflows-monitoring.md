# Workflows — Monitoring (read-only)

Monitoring scenarios only. None of the actions here require confirmation — they are all read.

> **Basic rule:** Before reporting to the user that something is wrong, gather enough data to be sure. A report with a single data point is noise — a report with 3-4 data points that all point to the same thing is a signal.

---

## 1. Daily account status snapshot

**When:** "Show me what's going on / general overview / what's the situation today"

**Call sequence:**

1. `customer_info` — verify the account is active, plan is valid
2. `list_servers` — list + status for each server
3. `get_alerts` — what's open right now
4. For each server with a status other than Running: `get_server_details` to check why

**How to summarize:**
- How many servers, how many apps, how many active / inactive
- Open alerts — by severity
- If everything is clean: "All systems operational, X servers, Y apps, no open alerts"
- Don't pad with text if everything is fine — be concise

---

## 2. Health check before a significant change

**When:** Before deployment / migration / DNS change / confirming a significant change for the client

**Goal:** baseline before, baseline after. If something goes wrong, you'll have a point of comparison.

**Call sequence:**

1. `get_server_details` (the target server) — current state
2. `get_server_monitoring_detail` — CPU, RAM, disk I/O over the last 5 minutes
3. `get_server_services_status` — verify all the services are running
4. `get_server_disk_usage` — free space
5. `get_app_monitoring_summary` (for each relevant application) — bandwidth, response time
6. `get_alerts` — no active surprises
7. `get_app_analytics_traffic` (last 24h) — to know what the normal traffic is

**Save the output before starting the change.** After the change, repeat the same sequence and compare.

---

## 3. Disk usage investigation

**When:** disk space alert, or "the server is slow"

**Sequence:**

1. `get_server_disk_usage` — where is the space?
2. If application folders are large: for each suspect app `get_app_details` + `get_app_settings`
3. Check logs via manual SSH (Cloudways MCP does not expose direct file system access): the administrator will need to connect via SSH to `/var/log/`, `/home/master/applications/<app>/logs/`
4. Check MySQL slow logs: `get_app_analytics_mysql` — if there are a lot of slow queries, the bin logs can balloon

**Good report:**
```
Server prod-shop-il (1234567): disk 87% full
Breakdown:
  - /home/master/applications/woocommerce-prod/public_html: 18GB
  - /var/log: 4.2GB (30 days of logs — rotation possible)
  - /tmp: 2.1GB
  - other: 6GB
Recommendation: clear_app_cache for all apps + manual log rotation via SSH
Next action requires confirmation: clear_app_cache (W)
```

---

## 4. Performance investigation — "the site is slow"

**When:** a client complains about slowness

**Step 1 — Is it really slow, or just their perception?**

1. `get_app_analytics_traffic` (last hour) — basically, traffic spike?
2. `get_app_monitoring_summary` — response time avg + p95
3. `get_server_monitoring_detail` — CPU/RAM of the server overall

**Step 2 — Where is the problem?**

1. `get_app_analytics_php` — slow scripts? memory exhaustion?
2. `get_app_analytics_mysql` — slow queries? locks?
3. `get_server_services_status` — Varnish/Memcached/Redis running?
4. `get_app_varnish_settings` — cache mode configured?

**Diagnostic matrix:**

| Symptom | Likely cause | Diagnostic tool |
|--------|----------|-----------|
| Sustained CPU 100% | PHP heavy / DB heavy | `get_app_analytics_php` + `get_app_analytics_mysql` |
| RAM 95%+ | memory leak / cache bloat | `get_server_services_status` + restart services |
| High Disk I/O | swap / log writes / DB writes | `get_server_disk_usage` |
| High response time but reasonable CPU/RAM | Varnish not running / slow external API | `get_server_services_status` + `get_app_varnish_settings` |
| Traffic spike | DDoS / viral / bot | `get_app_analytics_traffic` (sources) |

---

## 5. SSL expiry monitoring

**When:** weekly review of clients' SSL expiry dates

**Sequence for each application:**

1. `list_servers`
2. For each server: `get_server_details` → list of apps
3. For each app: `get_app_details` → `ssl` field with expiry date
4. Filter: SSL expiring within the next 30 days → flag for renewal
5. For each flagged app: check whether `letsencrypt_auto_renewal=true`. If not — double flag.

> If Let's Encrypt auto-renewal is active, Cloudways renews 30 days before expiry. If it fails to renew (DNS issue, rate limit) — you'll get an alert. It's still worth reviewing manually once every two weeks.

---

## 6. Traffic anomaly detection

**When:** "there's a jump in traffic" / "sales dropped" / before a campaign

**Sequence:**

1. `get_app_analytics_traffic` — bottom line: visitors, pageviews
2. If a spike: source of the traffic? geographic distribution?
3. Compare to the same day in the previous week / previous month
4. If a drop: `get_app_monitoring_summary` — did the error rate go up?
5. Check `get_alerts` — maybe something is taking the site down

> Cloudways analytics do not replace GA4/Plausible. They complement them with server-level metrics (raw bandwidth, requests). The two angles together give a good picture.

---

## 7. Multi-server comparison

**When:** "which server is client X on?" / "comparison between production and staging"

**Sequence:**

1. `list_servers` — filter by label/project
2. For two or three servers: `get_server_details` + `get_server_monitoring_detail` in parallel
3. Compare: provider, region, size, RAM/CPU usage, applications

**Tip:** Cloudways sometimes groups one client's apps on the same server. This can be a problem in production: a spike in one application affects the others. In an audit for a new client, this is the first thing to check.
