# Workflows — Monitoring (read-only)

תרחישי ניטור בלבד. אף לא אחת מהפעולות פה דורשת אישור — כולן read.

> **כלל בסיסי:** לפני שמדווח למשתמש שמשהו לא בסדר, אסוף מספיק נתונים כדי להיות בטוח. דיווח עם נתון בודד הוא רעש — דיווח עם 3-4 נתונים שמצביעים על אותו דבר הוא אות.

---

## 1. תמונת מצב יומית של חשבון

**מתי:** "תראה לי מה קורה / סקירה כללית / מה המצב היום"

**רצף קריאות:**

1. `customer_info` — לוודא שהחשבון פעיל, חבילה תקינה
2. `list_servers` — רשימה + סטטוס לכל שרת
3. `get_alerts` — מה פתוח עכשיו
4. לכל שרת בסטטוס לא Running: `get_server_details` לבדוק למה

**איך לסכם:**
- כמה servers, כמה apps, כמה פעילים / לא פעילים
- alerts פתוחים — לפי חומרה
- אם הכל נקי: "כל המערכות פעילות, X servers, Y apps, אין alerts פתוחים"
- אל תמלא בטקסט אם הכל בסדר — תהיה תמציתי

---

## 2. Health check לפני שינוי משמעותי

**מתי:** לפני deployment / migration / שינוי DNS / אישור שינוי משמעותי ללקוח

**מטרה:** baseline לפני, baseline אחרי. אם משהו השתבש, יהיה לך נקודת השוואה.

**רצף קריאות:**

1. `get_server_details` (השרת היעד) — מצב נוכחי
2. `get_server_monitoring_detail` — CPU, RAM, disk I/O ב-5 דקות אחרונות
3. `get_server_services_status` — לוודא שכל ה-services רצים
4. `get_server_disk_usage` — שטח פנוי
5. `get_app_monitoring_summary` (לכל אפליקציה רלוונטית) — bandwidth, response time
6. `get_alerts` — אין surprises פעילים
7. `get_app_analytics_traffic` (24h אחרון) — לדעת מה ה-traffic הרגיל

**שמור את הפלט לפני התחלת השינוי.** אחרי השינוי, חזור על אותו רצף והשווה.

---

## 3. Disk usage investigation

**מתי:** alert של disk space, או "השרת איטי"

**רצף:**

1. `get_server_disk_usage` — איפה השטח?
2. אם application folders גדולים: לכל app חשוד `get_app_details` + `get_app_settings`
3. בדוק logs באמצעות ssh ידני (Cloudways MCP לא חושף file system access ישיר): המנהל יצטרך להתחבר ב-SSH ל-`/var/log/`, `/home/master/applications/<app>/logs/`
4. בדוק MySQL slow logs: `get_app_analytics_mysql` — אם יש המון slow queries, ה-bin logs יכולים לתפוח

**דיווח טוב:**
```
שרת prod-shop-il (1234567): disk 87% מלא
פירוט:
  - /home/master/applications/woocommerce-prod/public_html: 18GB
  - /var/log: 4.2GB (לוגים של 30 ימים — אפשר rotation)
  - /tmp: 2.1GB
  - אחר: 6GB
המלצה: clear_app_cache לכל ה-apps + log rotation ידני ב-SSH
פעולה הבאה דורשת אישור: clear_app_cache (W)
```

---

## 4. Performance investigation — "האתר איטי"

**מתי:** לקוח מתלונן על איטיות

**שלב 1 — האם זה באמת איטי, או רק ההרגשה שלהם?**

1. `get_app_analytics_traffic` (שעה אחרונה) — בעיקרון traffic spike?
2. `get_app_monitoring_summary` — response time avg + p95
3. `get_server_monitoring_detail` — CPU/RAM של השרת בכלל

**שלב 2 — איפה הבעיה?**

1. `get_app_analytics_php` — slow scripts? memory exhaustion?
2. `get_app_analytics_mysql` — slow queries? locks?
3. `get_server_services_status` — Varnish/Memcached/Redis פעילים?
4. `get_app_varnish_settings` — cache mode מוגדר?

**מטריצת אבחנה:**

| תופעה | סביר שזה | כלי אבחוני |
|--------|----------|-----------|
| CPU 100% מתמשך | PHP heavy / DB heavy | `get_app_analytics_php` + `get_app_analytics_mysql` |
| RAM 95%+ | memory leak / cache bloat | `get_server_services_status` + restart services |
| Disk I/O גבוה | swap / log writes / DB writes | `get_server_disk_usage` |
| Response time גבוה אבל CPU/RAM סבירים | Varnish לא פעיל / external API איטי | `get_server_services_status` + `get_app_varnish_settings` |
| Spike של traffic | DDoS / viral / bot | `get_app_analytics_traffic` (מקורות) |

---

## 5. SSL expiry monitoring

**מתי:** סקירה שבועית של תוקפי SSL לקוחות

**רצף לכל אפליקציה:**

1. `list_servers`
2. לכל server: `get_server_details` → רשימת apps
3. לכל app: `get_app_details` → שדה `ssl` עם expiry date
4. סנן: SSL שיפוג ב-30 ימים הקרובים → flag לחידוש
5. לכל apps שבדגל: בדוק האם `letsencrypt_auto_renewal=true`. אם לא — flag כפול.

> אם Let's Encrypt auto-renewal פעיל, Cloudways מחדש 30 יום לפני פג תוקף. אם הוא לא מחדש (DNS issue, rate limit) — תקבל alert. עדיין שווה לעבור ידנית פעם בשבועיים.

---

## 6. Traffic anomaly detection

**מתי:** "יש מקפצה ב-traffic" / "המכירות ירדו" / לפני קמפיין

**רצף:**

1. `get_app_analytics_traffic` — bottom line: visitors, pageviews
2. אם spike: מקור ה-traffic? geographic distribution?
3. השווה לאותו יום בשבוע הקודם / בחודש הקודם
4. אם drop: `get_app_monitoring_summary` — error rate עלה?
5. בדוק את `get_alerts` — אולי יש משהו שמפיל את האתר

> Cloudways analytics לא מחליפים GA4/Plausible. הם משלימים עם metrics ברמת השרת (raw bandwidth, requests). שתי הזוויות יחד נותנות תמונה טובה.

---

## 7. Multi-server comparison

**מתי:** "איזה שרת לקוח X על?" / "השוואה בין production לבין staging"

**רצף:**

1. `list_servers` — מסנן לפי label/project
2. לשניים-שלושה שרתים: `get_server_details` + `get_server_monitoring_detail` במקביל
3. השווה: provider, region, size, RAM/CPU usage, אפליקציות

**טיפ:** Cloudways לפעמים מקבץ apps של לקוח אחד על אותו שרת. זה יכול להיות בעיה בייצור: spike באפליקציה אחת משפיע על אחרות. ב-audit ללקוח חדש, זה דבר ראשון לבדוק.
