# Workflows — Onboarding & Audit (agency client takeover)

תרחיש מרכזי: לקוח חדש שמגיע עם אתר על Cloudways, ואתה צריך לבנות תמונה מלאה — בלי קריסות, בלי surprises ב-3 בלילה.

> **עיקרון מנחה:** ב-audit הראשון, **אל תיגע בכלום.** רק קרא. תיעוד מלא של state נוכחי קודם. שינויים — אחרי שהבנת את התמונה ויש תוכנית מסודרת.

---

## שלב 0 — Pre-flight (לפני קריאת MCP)

ודא עם הלקוח:
- [ ] גישה ל-API מאושרת מצדם
- [ ] API key חדש (לא לחלוק עם הצוות הישן שלהם)
- [ ] אם יש team members ישנים שלא צריכים גישה — להוסיף בנקודה הזו (לא דרך MCP — UI בלבד)

---

## שלב 1 — מיפוי חשבון

```
1. customer_info               → מצב חבילה, רמת תמיכה, billing status
2. list_servers                → כל השרתים: count, providers, regions, sizes
3. list_projects               → איך הם מסודרים
4. list_team_members           → מי יש לו גישה ומה הרמה
5. get_alerts                  → הכל פתוח עכשיו
6. get_ssh_keys                → מי יכול להיכנס ב-SSH
```

**תוצרים שאתה רושם:**

| שדה | ערך |
|------|-----|
| Cloudways plan | (Starter/Growth/Enterprise?) |
| מספר שרתים | |
| Providers בשימוש | (DO/AWS/GCP/Vultr) |
| Regions | |
| Total monthly $ Cloudways (אומדן) | |
| Team members + הרשאות | |
| SSH keys (לא לפרסם — רק מספר) | |
| Open alerts | |

---

## שלב 2 — מיפוי שרתים

לכל שרת ברשימה:

```
1. get_server_details          → label, size, IP, master credentials, app list
2. get_server_settings         → PHP timeout, memory, upload limit, custom PHP
3. get_server_services_status  → מה רץ (Apache/Nginx/MySQL/Memcached/Varnish/Redis)
4. get_server_disk_usage       → שטח נוכחי + פירוט
5. get_server_monitoring_detail → CPU/RAM trends 24h אחרון
```

**Red flags שצריך לדגול:**
- [ ] Disk > 80% → לא יכולים להוסיף content בלי risk
- [ ] CPU spike מתמשך → בעיית performance
- [ ] PHP timeout < 60s → עלול לשבור long operations
- [ ] memory_limit < 256M → WordPress יתקע
- [ ] Memcached/Redis off → אין object caching
- [ ] Varnish off → אין page caching
- [ ] שרת מארח 5+ apps של מקורות שונים → blast radius גבוה

---

## שלב 3 — מיפוי אפליקציות

לכל אפליקציה (לפי app list מהשלב הקודם):

```
1. get_app_details             → URL, FQDN, app folder, DB credentials, SSL status
2. get_app_settings            → app-level overrides
3. get_app_credentials         → SFTP/additional access
4. get_app_monitoring_summary  → bandwidth, requests (לקבל תחושת גודל)
5. get_app_analytics_traffic   → visitors הכי פחות 7 ימים
6. get_app_analytics_php       → slow scripts? memory issues?
7. get_app_analytics_mysql     → slow queries?
8. get_app_varnish_settings    → cache configured?
```

**טבלת תוצרים לכל app:**

| שדה | ערך | red flag? |
|------|-----|-----------|
| App name | | |
| Primary domain | | |
| Additional domains/CNAMEs | | |
| SSL provider + expiry | | בודק auto-renew? |
| App type (WP/Magento/PHP/Laravel) | | |
| PHP version | | < 8.1 = upgrade needed |
| WP version (אם רלוונטי) | | < 6.0 = security risk |
| Active plugins (אם WP) | ידני | פלגינים נטושים? |
| DB size (גס) | | |
| Daily traffic (avg) | | |
| Daily bandwidth | | |
| Avg response time | | > 1.5s = problem |
| Backups schedule | | אין = critical risk |
| Varnish enabled | | false = perf gap |
| Object cache | | none = perf gap |
| HTTPS enforced | | false = SEO+security gap |

---

## שלב 4 — Security audit

```
1. get_whitelisted_ips_ssh     → מי יכול ל-SSH?
2. get_whitelisted_ips_mysql   → מי יכול לגשת ל-DB ישירות?
3. get_ssh_keys                → מפתחות שמורים
4. לכל IP חשוד: check_ip_blacklisted
```

**Red flags:**
- [ ] SSH whitelist ריק = פתוח לעולם (קריטי)
- [ ] MySQL whitelist ריק = פתוח לעולם (אסון אבטחה)
- [ ] SSH keys ישנים שלא ברור למי הם שייכים
- [ ] Team member הישן עדיין יש לו גישה
- [ ] גישה לאו רק לעובדי לקוח אלא גם לעובדים שעזבו

---

## שלב 5 — Backups audit

מצערים: Cloudways MCP לא חושף `list_backups` ישירות. צריך לעבור ל-UI (או API call ישיר).

**שאלות לבדוק ידנית:**
- האם backups אוטומטיים פעילים? (Cloudways → Server → Backups)
- תדירות? (כל יום / כל יומיים / כל שבוע)
- Retention? (כמה ימים אחורה?)
- האם יש off-platform backup? (Cloudways backups זמינים רק מתוך Cloudways — אם החשבון נסגר, אובד)

**המלצה סטנדרטית:**
- Daily Cloudways backups, 7-day retention
- Weekly off-platform backup (UpdraftPlus ל-S3 / Wasabi / Cloudways → Drive)

---

## שלב 6 — Reporting ללקוח

מסמך onboarding חייב לכלול (Hebrew):

### 1. תקציר מנהלים
- כמה servers, כמה apps, monthly spend גס
- 3-5 ה-red flags הכי חמורים שמצאת
- המלצה כללית (סדר עדיפות)

### 2. מצב נוכחי מפורט
- טבלאות לכל server + לכל app
- קישורים / IDs ב-Cloudways

### 3. רשימת tasks מומלצת
לפי priority (P0/P1/P2):

**P0 — חייב לבצע בשבוע הקרוב:**
- (קריטיות אבטחה: SSL פג, whitelist פתוח, וכו')

**P1 — חייב לבצע בחודש:**
- (PHP/WP upgrades, backup strategy, performance)

**P2 — שיפורים לטווח ארוך:**
- (CDN integration, Varnish tuning, monitoring setup)

### 4. הצעת מחיר
שעות מוערכות לכל P (₪300/h). הכל מתועד.

### 5. SLA / שיגרת תפעול
- ניטור שבועי (אילו queries יורצו)
- תגובה לalert (זמן יעד)
- חידושי SSL — מי אחראי

---

## תבנית report מהירה

```markdown
# Cloudways Audit — [Client Name]
Date: [YYYY-MM-DD]
Auditor: [your name]

## תקציר
- Servers: X | Apps: Y | Monthly spend (gross): $Z
- Critical findings: [N items]
- Recommendation summary: [one paragraph]

## Red flags
| Severity | Issue | Impact | Effort to fix |
|----------|-------|--------|---------------|
| P0 | ... | ... | ... |

## Inventory
(טבלאות מהשלבים 2-3)

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
