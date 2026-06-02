# Workflows — Maintenance (write operations)

תרחישי תחזוקה הדורשים פעולות write. **כל פעולה כאן דורשת confirmation מפורש מהמשתמש לפני ביצוע**, לפי הפטרן ב-`SKILL.md`.

> **כלל בסיסי:** Read לפני Write. תמיד תקרא את המצב הנוכחי לפני שאתה משנה אותו. גם כדי לוודא שהפעולה נדרשת, וגם כדי שיהיה לך baseline לחזרה אחורה.

---

## פטרן confirmation סטנדרטי

לפני **כל** קריאה לכלי W, הצג בלוק כזה למשתמש וחכה ל"כן" מפורש:

```
🔒 מאשר ביצוע?
   כלי: <tool_name>
   יעד: <server name + ID או app name + URL>
   פרמטרים: <key params>
   השפעה צפויה: <what happens>
   risks: <what could go wrong>
   המשך? (כן / לא / השהה)
```

אם המשתמש כתב "כן" — בצע. אם משהו אחר — בקש הבהרה.

לפעולות מסוכנות במיוחד (W!): הוסף **שלב שני**: "הקלד את שם השרת/האפליקציה כדי לאשר". זה מוודא שהוא קורא ולא מסכים אוטומטית.

---

## 1. Cache clear — basic

**מתי:** "האתר לא מתעדכן אחרי שינוי" / "מסך אדמין מציג גרסה ישנה"

**רצף:**

1. `get_app_details` — לאשש את ה-target app
2. `get_app_varnish_settings` — לראות אם Varnish פעיל
3. **CONFIRM:** `clear_app_cache` (W)
4. אם Varnish פעיל: **CONFIRM:** `manage_app_varnish` עם action=purge (W)
5. בדוק את האתר בדפדפן (curl או ידני)

**אם הבעיה ממשיכה:**
- בדוק plugin caches (W3 Total Cache, WP Rocket, LiteSpeed) — אלה לא ב-Cloudways, יש למחוק מתוך WP-Admin
- בדוק Cloudflare cache אם בשימוש — `purge everything` ב-Cloudflare UI
- CDN caches

---

## 2. SSL — חידוש Let's Encrypt

**מתי:** SSL מתקרב לפג תוקף ואין auto-renewal / auto-renewal נכשל

**רצף:**

1. `get_app_details` — בדוק את ה-domain ואת תאריך פג תוקף הנוכחי
2. בדוק שה-DNS עדיין מצביע לשרת (קריטי ל-LE validation)
3. **CONFIRM:** `renew_letsencrypt` (W)
4. **CONFIRM:** `set_letsencrypt_auto_renewal` עם enabled=true (W) — אם לא היה פעיל
5. `get_app_details` — verify

**אם renewal נכשל:**
- בעיה הכי נפוצה: DNS לא מצביע נכון, או wildcard domains לא קונפיגורים
- שנייה: rate limit של Let's Encrypt (5 attempts per week per domain)
- שלישית: validation HTTP-01 לא מצליח כי האתר מאחורי Cloudflare proxy → פתור עם DNS-01 או disable proxy זמנית

---

## 3. SSL — התקנת cert מותאם

**מתי:** הלקוח קנה cert מ-CA אחר (DigiCert, Sectigo, וכו'), לא Let's Encrypt

**רצף:**

1. אסוף מהלקוח: certificate, private key, ca bundle
2. `get_app_details` — לאשש target
3. **CONFIRM:** `install_ssl_certificate` (W) — עם cert + key
4. בדוק SSL מהדפדפן (SSL Labs grade A+ עדיף)
5. אם היה Let's Encrypt פעיל — תחליט: לשמור backup או revoke

> אזהרה: התקנת cert מותאם **מבטלת** את ה-Let's Encrypt אם היה פעיל. וודא שיש לך את ה-cert המותאם ביד **לפני** שמתחיל.

---

## 4. Backup לפני שינוי

**מתי:** לפני migration, restore, plugin update משמעותי, או כל "אני לא בטוח מה זה יעשה"

**רצף:**

1. `get_app_details` — מאשש target
2. `get_app_monitoring_summary` — לפני: snapshot של state
3. **CONFIRM:** `backup_app` (W)
4. בדוק שה-backup נוצר (`get_app_details` שוב, או דרך UI)
5. רשום timestamp של backup — תצטרך אותו ל-restore אם משהו ישתבש

**Backup ברמת שרת:**
- `backup_server` (W) — slower, includes everything, יותר יקר
- שימושי לפני server-wide change (PHP upgrade, OS upgrade, package change)

---

## 5. Restore אחרי שגיאה

**מתי:** משהו השתבש (deployment רע, hack, accidental delete)

**רצף — קריטי לעקוב לפי הסדר:**

1. **STOP** — אל תעשה כלום עד שאתה מבין את היקף הבעיה.
2. `get_app_details` — מצב נוכחי
3. בדוק רשימת backups זמינים (דרך Cloudways UI — MCP לא חושף list_backups ישירות)
4. **CONFIRM שלב 1:** "האם backup מתאריך X הוא הנקודה שאתה רוצה לחזור אליה?"
5. **CONFIRM שלב 2:** Type the app name to confirm restore
6. **CONFIRM:** `restore_app` (W!) — דריסה מלאה של state נוכחי
7. בדוק שהאתר עובד
8. אם יש בעיה: **CONFIRM:** `rollback_app_restore` (W) תוך פרק זמן מוגבל

> **חלון rollback מוגבל.** אחרי מספר שעות / יום, אי אפשר לעשות rollback. תוודא שהאתר עובד **באותו יום** של ה-restore.

---

## 6. Restart server / service

**מתי:** memory leak, services stuck, או troubleshooting

**Priority order — נסה את הכי שקט קודם:**

1. `clear_app_cache` (W) per app — לפעמים זה הכל
2. `change_service_state` (W) על service בודד (למשל restart MySQL בלבד)
3. אם לא עוזר: `restart_server` (W) — downtime 1-5 דקות לכל ה-apps

**לפני restart_server:**

1. **חובה:** `get_server_details` → רשימת apps
2. **חובה:** ספור משתמשים פעילים (אם רלוונטי — אתר חנות עם carts פתוחים?)
3. **CONFIRM כפול:** "השרת מארח X אפליקציות — Y, Z, W. כל אחת מהן תהיה offline X דקות. המשך?"
4. בצע
5. **VERIFY:** `get_server_services_status` אחרי restart

---

## 7. IP whitelist — SSH/MySQL

**מתי:** הוספת מפתח לחבר צוות, או הסרת גישה ל-IP ישן

**רצף:**

1. `get_whitelisted_ips_ssh` — מה יש עכשיו
2. תכנן את הרשימה החדשה — **כולל IP שלך**
3. **CONFIRM:** הצג למשתמש: "הרשימה החדשה היא: [...]. ה-IP שלך X.X.X.X נשאר ברשימה? כן/לא"
4. אם המשתמש חסר ברשימה — **עצור והבהר**
5. **CONFIRM:** `update_whitelisted_ips` (W)
6. **VERIFY:** נסה SSH מיידית (אם לא עובד — Cloudways support כדי לשחזר)

> **תרחיש סיוט להימנע ממנו:** עדכון whitelist + הסרת ה-IP של עצמך + אין SSH אלטרנטיבי. הדרך החוצה היחידה — Cloudways UI (פאניקה) או support ticket (זמן). זהיר.

---

## 8. Disk cleanup

**מתי:** disk usage > 80%

**Priority order:**

1. `get_server_disk_usage` — איפה השטח?
2. **CONFIRM:** `clear_app_cache` (W) לכל ה-apps החשודים
3. אם לא מספיק: **CONFIRM:** `optimize_server_disk` (W) — Cloudways magic cleanup
4. אם לא מספיק: שדרוג size (UI בלבד) או SSH ידני לניקוי logs
5. **VERIFY:** `get_server_disk_usage` שוב

> `optimize_server_disk` עלולה למחוק logs. אם הלקוח חייב לשמור logs (compliance, debugging), צא לידנית קודם.

---

## 9. Enforce HTTPS

**מתי:** אתר ישן שרץ עדיין ב-HTTP / לקוח רוצה SEO/security boost

**רצף:**

1. `get_app_details` — בדוק שיש SSL תקין מותקן
2. אם אין SSL: התקן קודם (`install_letsencrypt` או `install_ssl_certificate`)
3. **CONFIRM:** `enforce_app_https` (W)
4. בדוק שה-redirect עובד: `curl -I http://example.com` → 301 ל-HTTPS

> **WordPress quirk:** אחרי enforce HTTPS, ייתכן שתצטרך לעדכן את `WP_HOME` ו-`WP_SITEURL` ב-`wp-config.php` או דרך WP-CLI: `wp option update home https://example.com && wp option update siteurl https://example.com`. אחרת — mixed content errors.

---

## 10. Git pull deployment

**מתי:** deploy של branch לאפליקציה

**רצף:**

1. `get_git_branch_names` — לוודא שה-branch קיים
2. `get_git_deployment_history` — מה ה-state נוכחי
3. **CONFIRM:** `backup_app` (W) — תמיד backup לפני deploy
4. **CONFIRM:** `git_pull` (W) עם branch + commit hash אם רלוונטי
5. **VERIFY:** בדוק את האתר
6. אם משהו נשבר: `restore_app` (W) ל-backup מסעיף 3

> Cloudways Git deploy לא מריץ build steps. אם האתר דורש `npm run build` / `composer install` — תצטרך לעשות את זה ידנית ב-SSH אחרי ה-pull, או להחזיק build artifacts ב-repo.

---

## 11. Varnish — configure/purge

**מתי:** הגדרת cache strategy / טיוב performance

**רצף:**

1. `get_app_varnish_settings` — config נוכחי
2. הבן את ה-policy: cache TTL, exceptions, purge rules
3. **CONFIRM:** `manage_app_varnish` (W) עם action ספציפי (enable/disable/purge/configure)
4. אם purge: בדוק שה-cache נקי (`curl -I` ל-URL → צריך `X-Cache: MISS` בקריאה ראשונה)

> Varnish לא תואם לכל אפליקציה. ל-WooCommerce או אפליקציות עם session-heavy data, צריך exclusion rules מדויקות. תמיד תבדוק אחרי שינוי.

---

## Cleanup לאחר session

לפני שמסיים שיחה שכללה פעולות write:

1. סכם מה בוצע
2. הזכר אם backups נוצרו ומתי תוקפם
3. הזכר אם יש פעולות שעדיין דורשות verification מאוחרת (SSL renewal, cache propagation)
4. אם משהו לא הסתיים — תעד מה נשאר פתוח
