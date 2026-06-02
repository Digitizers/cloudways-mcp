---
name: cloudways-mcp
description: |
  Operational guide for managing Cloudways servers and applications, across one or several Cloudways accounts, via the Cloudways MCP server (Cloudways' official MCP / Remote MCP per their support docs; or the community self-hosted cw-mcp implementation).
  Use whenever the user mentions Cloudways, a Cloudways server or app, server monitoring, app monitoring, bandwidth, disk usage, PHP/MySQL/traffic analytics, SSH/MySQL IP whitelisting, Let's Encrypt or SSL on Cloudways, Varnish cache, app cloning, backups/restore on Cloudways, Git deployments on Cloudways, or running an audit/onboarding on a Cloudways-hosted client site.
  Also use for self-hosting the MCP server itself (Python + Redis setup, mcp-remote configuration for Claude Desktop/Code).
  Any write operation (start/stop/restart server, backup, restore, rollback, install/revoke SSL, update CNAME, change whitelist, clear cache, change service state, git pull) requires explicit confirmation of target server/app and intended action before execution.
---

# Cloudways MCP — Skill תפעולי

ניהול תשתית Cloudways דרך MCP server של Cloudways.

> **שתי דרכים, וצריך לדעת באיזו אתה:**
> 1. **רשמי — Cloudways (Remote) MCP** *(מומלץ, default)*: MCP מתארח על ידי Cloudways (סופק ב-Q2 2026). מתחברים אליו ישירות בלי self-hosting. מקור האמת לחיבור הוא **המאמר הרשמי**: `support.cloudways.com/en/articles/14654372`. ראה `references/installation.md` סעיף "Official".
> 2. **קהילתי — self-hosted (`cw-mcp`)**: שרת Python+Redis שמריצים מקומית. ⚠️ **הריפו `github.com/aphraz/cw-mcp` כרגע מחזיר 404** (נמחק/הוסתר). אם זו הדרך שלך — צריך fork/עותק זמין; ראה `references/installation.md` סעיף "Self-hosted".
>
> אם לא ידוע באיזו דרך משתמשים — שאל את המשתמש, או זהה לפי אופן החיבור: אם ה-tools מופיעים ב-Claude בלי שהרצת שרת מקומי → זה הרשמי.

> **הקשר:** הסקיל בנוי לעבודה יומיומית של ניהול לקוחות/סביבות על Cloudways — monitoring, תחזוקה שגרתית, onboarding/audit ללקוחות חדשים, ואוטומציות. כל ערכים כספיים שמדווחים על ידי ה-API הם $ (USD), לא ₪.

---

## Quick Route

| כוונה | טען |
|--------|------|
| התקנה/הגדרה ראשונית של ה-MCP server | `references/installation.md` |
| לא יודע איזה כלי קיים / חיפוש כלי לפי שם | `references/tools-catalog.md` |
| ניטור, status check, bandwidth, analytics | `references/workflows-monitoring.md` |
| Cache clear, SSL, backup, restart, IP whitelist | `references/workflows-maintenance.md` |
| Audit חדש / onboarding לקוח חדש | `references/workflows-onboarding.md` |
| בניית workflow ל-n8n/Make/Claude Code אוטומטי | `references/workflows-automation.md` |
| כמה חשבונות Cloudways / הגדרת multi-account | `references/installation.md` (סעיף Multi-account) |

**טען רק את מה שנדרש.** מקסימום 2-3 references למשימה. אם המשתמש רק שואל "תראה לי את השרתים שלי", אל תטען את כל הקטלוג — קרא ישירות `list_servers`.

---

## Safety rules (קריא לפני כל פעולה)

1. **בחירת חשבון נכון — לפני כל דבר אחר (multi-account).** קיימים **כמה חשבונות Cloudways**, כל אחד כ-MCP connection נפרד עם prefix משלו (למשל `mcp__cloudways-clientA__*`). לפני כל קריאה — ודא לאיזה חשבון היא שייכת. אם יותר מחשבון אחד מחובר ולא ברור מההקשר על איזה מדובר — **עצור ושאל**, אל תנחש. server/app IDs **לא ניתנים להחלפה בין חשבונות** — ID 1234567 בחשבון A הוא משאב אחר לגמרי (או לא קיים) בחשבון B. אל תיקח ID מתשובה של חשבון אחד ותריץ אותו מול חשבון אחר. ראה הסעיף "Multi-account" למטה.

2. **Write operations דורשות אישור מפורש.** לפני כל קריאה ל-tool ששייכת לקטגוריה Write (ראה רשימה למטה), הצג למשתמש: **החשבון**, שם הכלי, השרת/אפליקציה היעד (ID + שם), הפרמטרים, ההשפעה הצפויה. חכה לתשובת אישור לפני ביצוע. אל תניח שאישור לפעולה אחת מקנה אישור לפעולות נוספות — וגם לא שאישור בחשבון אחד חל על חשבון אחר.

3. **Backup לפני שינוי משמעותי.** לפני `restore_app`, `rollback_app_restore`, `reset_app_file_permissions`, `manage_server_varnish`, או כל שינוי קונפיגורציה — בדוק עם המשתמש אם יש backup עדכני. אם לא, הצע להריץ `backup_app` / `backup_server` קודם.

4. **כפל שירות = כפל סיכון.** Cloudways מארח לרוב **כמה אפליקציות על אותו שרת**. `stop_server` או `restart_server` משפיע על **כל** האפליקציות. תמיד תוודא שהמשתמש מודע לרשימת האפליקציות על השרת לפני פעולה ברמת השרת.

5. **`revoke_letsencrypt` ו-`delete_app_cname` = הרס מיידי בייצור.** דורש אישור כפול: גם של הפעולה וגם של הדומיין/האפליקציה הספציפיים.

6. **Credentials.** לכל חשבון API key + email משלו, שנחשפים דרך headers ב-MCP. אל תדפיס אותם בתגובות. אל תערבב credentials בין חשבונות. אם המשתמש מבקש לראות אותם, הפנה ל-Cloudways Platform → Account → API.

7. **Read-only by default.** אם המשתמש רק מבקש "תראה לי / תבדוק / תנטר" — בחר תמיד את הכלי ה-read-only שמתאים. אל תציע פעולה destructive אלא אם המשתמש ביקש במפורש.

---

## Write operations — רשימה מלאה לקטגוריה

לכל אחת מהפעולות הבאות **חובה לקבל אישור מפורש לפני ביצוע**:

**Server level (משפיע על כל האפליקציות בשרת):**
- `start_server`, `stop_server`, `restart_server`
- `backup_server`
- `optimize_server_disk`
- `change_service_state` (Apache/Nginx/Memcached/MySQL/Varnish)
- `manage_server_varnish`

**App level:**
- `clone_app` (יוצר עותק חדש — צורך משאבים)
- `backup_app`, `restore_app`, `rollback_app_restore`
- `reset_app_file_permissions`
- `enforce_app_https`
- `update_app_cname`, `delete_app_cname` ⚠️ (יכול לשבור production)
- `clear_app_cache`, `manage_app_varnish`

**Security & SSL:**
- `install_ssl_certificate`, `remove_ssl_certificate` ⚠️
- `install_letsencrypt`, `renew_letsencrypt`, `set_letsencrypt_auto_renewal`
- `revoke_letsencrypt` ⚠️⚠️ (הרס מיידי)
- `update_whitelisted_ips`, `allow_ip_siab`, `allow_ip_adminer`

**Git deployment:**
- `git_clone`, `git_pull` (יכול לשבור production אם יש conflict)
- `generate_git_ssh_key`

---

## פטרן confirmation לפעולה destructive

לפני ביצוע, הצג בלוק כזה:

```
🔒 מאשר ביצוע פעולה?
   חשבון: clientA (mcp__cloudways-clientA)
   כלי: stop_server
   שרת: production-shop-il (ID: 1234567)
   אפליקציות שיושפעו: woocommerce-prod, staging-clone, admin-tools
   השפעה: כל 3 האפליקציות יהיו offline עד restart ידני
   המשך? (כן / לא / השהה ובדוק backup קודם)
```

המתן לתשובה מפורשת. "כן" מילולי בלבד = אישור. הסכמה משתמעת לא מספיקה. שורת **החשבון חובה** כשמחובר יותר מחשבון אחד — היא מונעת ביצוע פעולה על החשבון הלא נכון.

---

## Authentication — מבט מהיר

ה-MCP server רץ **אצלך מקומית** (לא שירות hosted של Cloudways). הוא מקבל credentials דרך HTTP headers:

```bash
# סודות שאתה מייצא לסביבה (לא commit ל-git)
export CLOUDWAYS_EMAIL="your-account@example.com"
export CLOUDWAYS_API_KEY="your-cloudways-api-key"
```

ה-API key מופק ב-Cloudways Platform → **Account → API**.

לקונפיגורציה מלאה של Claude Desktop / Claude Code עם `mcp-remote`, ראה `references/installation.md`.

---

## Multi-account — עבודה עם כמה חשבונות Cloudways

לרוב יש **כמה חשבונות Cloudways** (לקוחות שונים / סביבות שונות). כל חשבון מחובר כ-MCP connection **נפרד** עם credentials משלו, ולכן מופיע ב-Claude עם **prefix משלו**:

```
mcp__cloudways-clientA__list_servers
mcp__cloudways-clientB__list_servers
mcp__cloudways-internal__list_servers
```

> ההגדרה (איך מחברים כמה חשבונות — connection אחד לכל חשבון, עם headers שונים, או instance-per-port) מתועדת ב-`references/installation.md` סעיף **Multi-account configuration**. כללי ה-runtime כאן.

### כלל הזהב: לזהות חשבון לפני כל פעולה

1. **חשבון בודד מחובר** → השתמש בו, אין צורך לשאול.
2. **כמה חשבונות מחוברים** → קבע לאיזה חשבון הבקשה שייכת **לפני** שאתה קורא ל-tool:
   - אם המשתמש ציין במפורש לקוח/חשבון ("תבדוק את prod של clientB") → השתמש ב-connection התואם.
   - אם שם השרת/הדומיין מזהה חד-משמעית חשבון אחד → אפשר להסיק, אבל ציין במפורש באיזה חשבון אתה פועל.
   - אם **לא ברור** → עצור ושאל: "על איזה חשבון? (clientA / clientB / internal)". אל תנחש, ואל תריץ על כולם "ליתר ביטחון".

### בידוד מוחלט בין חשבונות

- **IDs לא חוצים חשבונות.** server_id / app_id שקיבלת מ-`mcp__cloudways-clientA` תקפים **רק** מול clientA. לעולם אל תיקח ID מתשובה של חשבון אחד ותעביר לכלי של connection אחר.
- **אישור פר-חשבון.** אישור write בחשבון אחד לא חל על אחר. כל פעולת write על חשבון חדש = בלוק confirmation חדש (כולל שורת חשבון).
- **Credentials לא מתערבבים.** לכל connection ה-email + API key שלו. אל תניח שאותם credentials עובדים על חשבון אחר.

### חיפוש חוצה-חשבונות (read בלבד)

כשהמשתמש מבקש משהו רוחבי — "באיזה חשבון יושב הדומיין shop.example.co.il?", "תן לי סקירת disk לכל החשבונות" — מותר ולגיטימי **לקרוא (read-only)** מכל ה-connections, אבל:
- הרץ את אותו רצף reads על כל connection **בנפרד**, ותייג כל תוצאה עם שם החשבון.
- סכם בטבלה עם עמודת "חשבון" ברורה.
- **לעולם אל** תבצע פעולת write רוחבית על כמה חשבונות בלי אישור פרטני לכל אחד.

```
דוגמת תיוג בתשובה:
| חשבון    | שרת              | disk |
|----------|------------------|------|
| clientA  | prod-shop-il     | 87%  |
| clientB  | prod-blog        | 41%  |
| internal | ops-tools        | 63%  |
```

---

## דפוסי שימוש נפוצים (לדוגמה)

### תמונת מצב מהירה של חשבון
```
1. list_servers              → רשימת כל השרתים
2. get_alerts                → התראות פעילות
3. customer_info             → פרטי חשבון + מצב חבילה
```

### Health check ליציאה לסוף שבוע (לקוח production)
```
1. get_server_details        → CPU/RAM/disk
2. get_server_monitoring_detail → metrics
3. get_app_monitoring_summary   → לכל אפליקציה
4. get_alerts                → התראות פתוחות
5. get_server_disk_usage     → אם disk > 80% — דגל אדום
```

### חידוש SSL לפני פג תוקף
```
1. get_app_details           → מה ה-FQDN הנכון
2. renew_letsencrypt          ← write — דורש אישור
3. get_app_details (שוב)      → ודא שה-SSL התעדכן
```

לדפוסים מפורטים יותר, טען את ה-workflows הרלוונטיים.

---

## Versioning ו-source of truth

- **ה-MCP החי הוא source of truth — תמיד.** שמות הכלים והקטגוריות בקטלוג כאן תועדו ב-2026-Q1 מתוך השרת הקהילתי (`cw-mcp`). ה-MCP הרשמי של Cloudways עשוי לחשוף שמות/יכולות שונים. לפני שאתה מצהיר ש-tool קיים/לא קיים — **בדוק את רשימת ה-tools החיה** שמחוברת ב-Claude (`mcp__cloudways*__*`), ועדכן את הקטלוג בהתאם.
- **read-only מול write — לא ודאי, אז התנהג בזהירות.** מקורות שונים סותרים: חלק מתעדים את הקהילתי כ-read-only-בלבד (write "מתוכנן"), והתיעוד כאן הניח שקיימות פעולות write. **אל תניח** — בדוק מול השרת החי אילו כלים זמינים. בכל מקרה, כל כלי שמבצע שינוי **חייב** לעבור את פטרן ה-confirmation; הכלל הזה בטוח גם אם בפועל אין כלי write (אז הוא פשוט לא מופעל).
- **`github.com/aphraz/cw-mcp` כרגע 404.** אם הסתמכת עליו להתקנה — ראה `references/installation.md`; אתרי אגרגציה (glama/lobehub/mcp.so) עדיין מחזיקים עותק מאוחסן, אבל המקור החי איננו.
