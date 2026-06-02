# Tools Catalog — Cloudways MCP

קטלוג מלא של כלים שחושפים `aphraz/cw-mcp` בגרסה הנוכחית (43+ כלים). מסווג לפי קטגוריה תפקודית, עם דגלי **R** (read-only) / **W** (write — דורש confirmation) / **W!** (write destructive — דורש confirmation כפול).

> **חשוב:** הרשימה נכונה לתיעוד מ-2026-Q1. ייתכנו כלים נוספים שנוספו אחרי. אם מחפש כלי ולא מוצא — בדוק את הרשימה החיה שה-MCP מחזיר ועדכן.

---

## 1. Authentication & Account info

| Tool | סוג | מתי להשתמש |
|------|-----|------------|
| `ping` | R | בדיקת חיים של ה-MCP endpoint. שימושי לאחר שינוי config. |
| `customer_info` | R | פרטי החשבון, חבילה, יתרה, רמת תמיכה. נקודת פתיחה טובה לכל audit. |
| `rate_limit_status` | R | כמה calls נשארו עד reset. שימושי לפני batch operation. |

---

## 2. Discovery (אילו משאבים אפשר להקצות)

כל הכלים האלה R. שימושיים בעיקר לפני יצירת שרת/אפליקציה חדשים (כי הם מחזירים את המקצוע של ה-API: providers/regions/sizes/apps שתומכים).

| Tool | מחזיר |
|------|--------|
| `get_available_providers` | DigitalOcean, AWS, GCP, Vultr, Linode |
| `get_available_regions` | אזורים זמינים לכל provider |
| `get_available_server_sizes` | RAM/CPU/disk options לכל provider |
| `get_available_apps` | WordPress, Magento, Laravel, custom PHP, וכו' — כולל גרסאות |
| `get_available_packages` | חבילות תוכנה (PHP versions, MySQL/MariaDB, Redis, etc.) |
| `get_ssh_keys` | רשימת SSH keys שמורות בחשבון |

---

## 3. Servers — list & inspect (read-only)

| Tool | מחזיר |
|------|--------|
| `list_servers` | כל השרתים: ID, label, provider, region, size, status, IP, master credentials |
| `get_server_details` | פרטי שרת ספציפי: app list, ssh user, db credentials, ssl config |
| `get_server_settings` | קונפיגורציה (PHP timeout, memory limit, upload_max_filesize, etc.) |
| `get_server_disk_usage` | פירוט שטח לפי תיקייה — שימושי לאיתור log/cache שתפח |
| `get_server_services_status` | סטטוס Apache, Nginx, MySQL, Memcached, Varnish, Redis |
| `get_server_monitoring_detail` | metrics מפורט: CPU, RAM, disk I/O, network |
| `get_server_analytics` | נתוני analytics על השרת (traffic, uptime) |

---

## 4. Servers — power & lifecycle (W — דורש confirmation)

⚠️ **כל אלה משפיעים על *כל* האפליקציות בשרת.**

| Tool | מה עושה | סיכון |
|------|---------|--------|
| `start_server` | מפעיל שרת כבוי | אם השרת היה כבוי בכוונה — לבדוק למה |
| `stop_server` | מכבה (downtime לכל האפליקציות) | כל האתרים יורדים. הזהר. |
| `restart_server` | reboot מלא | downtime זמני (1-5 דקות) לכל האפליקציות |
| `backup_server` | snapshot מלא | צורך זמן + שטח. לרוב חיוב. |
| `optimize_server_disk` | מנקה logs ו-temp files | יכול למחוק logs חשובים. לבדוק policy. |
| `change_service_state` | enable/disable Apache/Nginx/MySQL/etc. | downtime אם מבטל service קריטי |
| `manage_server_varnish` | enable/disable/purge Varnish ברמת השרת | יכול לשבור cache strategy של אפליקציות |

---

## 5. Applications — inspect (read-only)

| Tool | מחזיר |
|------|--------|
| `get_app_details` | URL, FQDN, app folder, SSH user, DB credentials, SSL config |
| `get_app_credentials` | רשימת additional credentials (SFTP, etc.) |
| `get_app_settings` | PHP-specific overrides, Cron jobs, environment |
| `get_app_monitoring_summary` | bandwidth, requests, response time — תקציר |
| `get_app_analytics_traffic` | traffic detailed (visitors, page views) |
| `get_app_analytics_php` | PHP metrics (slow scripts, memory) |
| `get_app_analytics_mysql` | MySQL metrics (slow queries, locks) |
| `get_app_varnish_settings` | Varnish config ברמת אפליקציה |

---

## 6. Applications — lifecycle & data (W — דורש confirmation)

| Tool | מה עושה | סיכון |
|------|---------|--------|
| `clone_app` | יוצר עותק חדש על אותו שרת או אחר | משאבים + DB עותק |
| `backup_app` | snapshot של files + DB | זמן + שטח |
| `restore_app` | משחזר מ-backup | **דורסת state נוכחי**. גבה תחילה. |
| `rollback_app_restore` | חוזרת אחורה אחרי restore | רק תוך זמן מוגבל מ-restore |
| `clear_app_cache` | מנקה page cache, varnish | תנועה ראשונה תהיה איטית — אבל בדרך כלל בטוח |
| `reset_app_file_permissions` | אופס chmod/chown לברירת מחדל | אם מודיפיקציות ידניות קיימות — תאבד אותן |
| `enforce_app_https` | הוספת redirect HTTP→HTTPS | יכול לשבור אם SSL לא תקין; בדוק SSL קודם |
| `manage_app_varnish` | enable/disable/purge רמת app | יכול לשבור caching strategy |

---

## 7. Domains & CNAME (W — careful)

| Tool | מה עושה | סיכון |
|------|---------|--------|
| `update_app_cname` | מעדכן/מוסיף CNAME לאפליקציה | אם DNS לא מוכן — אתה שובר את האתר |
| `delete_app_cname` | מסיר CNAME | **הרס מיידי לאפליקציה production**. דורש אישור כפול. |

> תהליך מומלץ: לעולם לא לעדכן CNAME ב-Cloudways לפני שווידאת:
> 1. ה-DNS המקור (Cloudflare, Route53, etc.) מצביע נכון
> 2. TTL נמוך הוגדר כבר 24h קודם
> 3. אין traffic חי כרגע (בודק b- analytics)

---

## 8. SSL — Let's Encrypt

| Tool | סוג | מה עושה |
|------|-----|---------|
| `install_letsencrypt` | W | מתקין LE לדומיין/אפליקציה |
| `renew_letsencrypt` | W | חידוש ידני |
| `set_letsencrypt_auto_renewal` | W | מפעיל auto-renew (מומלץ תמיד) |
| `revoke_letsencrypt` | **W!** | **מבטל את התעודה — האתר עלול לזרוק SSL error מיידית** |

## SSL — Custom certs

| Tool | סוג | מה עושה |
|------|-----|---------|
| `install_ssl_certificate` | W | מתקין cert מותאם אישית (cert + private key) |
| `remove_ssl_certificate` | W | מסיר cert מותאם (חוזר לברירת מחדל / חוסם SSL) |

---

## 9. Security & Access control

| Tool | סוג | מה עושה |
|------|-----|---------|
| `get_whitelisted_ips_ssh` | R | רשימת IPs מורשים ל-SSH |
| `get_whitelisted_ips_mysql` | R | רשימת IPs מורשים ל-MySQL חיצוני |
| `update_whitelisted_ips` | W | מעדכן רשימה (יכול לנעול את עצמך החוצה אם טועה) |
| `check_ip_blacklisted` | R | בדיקה אם IP מסוים חסום על ידי Cloudways |
| `allow_ip_siab` | W | מעניק גישה ל-SIAB (Server-in-a-Box / DB tools) |
| `allow_ip_adminer` | W | מעניק גישה ל-Adminer (web DB GUI) |

> **דגל:** `update_whitelisted_ips` יכול לחסום את עצמך. תמיד תוודא שה-IP שלך **נשאר** ברשימה החדשה.

---

## 10. Git deployment

| Tool | סוג | מה עושה |
|------|-----|---------|
| `generate_git_ssh_key` | W | יוצר זוג מפתחות חדש |
| `get_git_ssh_key` | R | מחזיר את ה-public key (להוסיף ל-GitHub/GitLab) |
| `git_clone` | W | clone ראשוני של repo לאפליקציה |
| `git_pull` | W | pull בכל ה-changes (יכול לשבור אם conflict) |
| `get_git_deployment_history` | R | היסטוריה — שימושי לאיתור מתי משהו נשבר |
| `get_git_branch_names` | R | רשימת branches |

> ה-Git deployment של Cloudways לא מחליף CI/CD מלא. הוא טוב לאתרי לקוח שלא דורשים build step. לאתרים מורכבים — תמשיך עם GitHub Actions / Vercel / wpe.

---

## 11. Team & Projects

| Tool | סוג | מה עושה |
|------|-----|---------|
| `list_projects` | R | פרויקטים בחשבון (ארגון של servers/apps) |
| `list_team_members` | R | משתמשים שהוזמנו + הרשאות |
| `get_alerts` | R | התראות פתוחות (disk full, service down, etc.) |

---

## 12. ארגון לפי תסריט שימוש (cross-reference מהיר)

### בודק חשבון מקצה לקצה
`customer_info` → `list_servers` → `get_alerts` → `rate_limit_status`

### Health check לאפליקציה
`get_app_details` → `get_app_monitoring_summary` → `get_server_services_status` → `get_app_analytics_traffic`

### חידוש SSL לפני פג תוקף
`get_app_details` (לבדוק תוקף) → `renew_letsencrypt` (W) → `get_app_details` (verify)

### Audit לקוח חדש
ראה `workflows-onboarding.md`

### Disk full warning
`get_server_disk_usage` → לזהות אפליקציה אשמה → `clear_app_cache` (W) → אם לא מספיק → `optimize_server_disk` (W)

### Cache nuke
`clear_app_cache` (W) + `manage_app_varnish` עם action=purge (W)

### IP whitelist (פיתוח / debugging)
`get_whitelisted_ips_ssh` → `update_whitelisted_ips` (W — וודא שאתה לא מוחק את עצמך)

---

## כלים שלא קיימים (נכון לעכשיו) — workarounds

| מה שחסר | חלופה |
|---------|--------|
| `create_server` | UI של Cloudways בלבד. ה-API public לא חושף provisioning דרך MCP נכון לעכשיו. |
| `delete_server` | UI / API ישיר עם curl, לא דרך MCP. בטיחות. |
| `transfer_app` בין שרתים | UI בלבד. |
| `staging_url` toggle | חלק מ-`get_app_details` (תכונה existing). |
| Application creation על שרת קיים | חלקית — בדוק האם `add_app` קיים בגרסה שלך. |

אם משימה דורשת פעולה שאינה ב-MCP, אפשר עדיין להשתמש ב-Cloudways API REST ישירות עם curl + ה-credentials. ה-MCP הוא wrapper נוח, לא היחיד.
