# Workflows — Automation & Integration

איך לבנות אוטומציות מסביב ל-Cloudways MCP — חיבור ל-n8n, Make.com, Claude Code, או cron jobs. בנוי במיוחד ל-Digitizer stack.

> **עיקרון בסיסי:** ה-MCP server הוא קודם כל interface ל-Claude. לאוטומציה ללא human-in-the-loop, לרוב **עדיף לקרוא ישירות ל-Cloudways API** (curl/n8n HTTP node) ולא דרך MCP. ה-MCP מוסיף overhead, ובאוטומציה הוא לא נחוץ.

---

## מתי להשתמש ב-MCP vs API ישיר?

| תרחיש | בחר |
|--------|-----|
| שיחה חיה ב-Claude / Claude Code | MCP |
| Daily report שמיוצר אוטומטית | API ישיר (n8n/Make) |
| Alerting → action אוטומטית | API ישיר |
| Audit one-off | MCP |
| CI/CD trigger (post-deploy backup, etc.) | API ישיר |
| Claude Code headless שמרץ pipelines | MCP (אם Claude הוא ה-orchestrator) |

ה-MCP חסך זמן בהקשר אינטראקטיבי. באוטומציה — overhead.

---

## 1. Cloudways API direct (לאוטומציה)

### Authentication flow

```bash
# שלב 1: get access token
TOKEN=$(curl -sX POST "https://api.cloudways.com/api/v1/oauth/access_token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "email=$CLOUDWAYS_EMAIL&api_key=$CLOUDWAYS_API_KEY" \
  | jq -r '.access_token')

# שלב 2: השתמש בו
curl -sH "Authorization: Bearer $TOKEN" \
     "https://api.cloudways.com/api/v1/server"
```

ה-token תקף לזמן מוגבל (בערך שעה). באוטומציה ארוכה, ייצר מחדש.

### Endpoints נפוצים

| Endpoint | Method | מה זה |
|----------|--------|--------|
| `/server` | GET | list servers |
| `/server/{id}` | GET | server details |
| `/app/manage/varnish` | POST | Varnish operations |
| `/app/manage/backup` | POST | trigger backup |
| `/app/letsencrypt_renew` | POST | renew SSL |
| `/app/analytics/visitor` | GET | traffic |
| `/app/manage/cache` | POST | clear cache |

תיעוד מלא: `https://developers.cloudways.com/docs/`.

---

## 2. n8n workflows (Digitizer stack)

### Workflow: Daily account health check

**Trigger:** Cron, every day 8:00 AM Israel time

```
┌─ Cron (08:00 Asia/Jerusalem)
├─ HTTP: POST /oauth/access_token  → get TOKEN
├─ HTTP: GET /server                → list of servers
├─ Loop over servers:
│   ├─ HTTP: GET /server/{id}      → details
│   ├─ HTTP: GET /alerts           → alerts
│   ├─ Filter: status != Running OR alerts > 0
│   └─ Continue if filter passed
├─ Aggregate: Build summary message
└─ Slack/Email: Send to team
```

**הערה:** ה-API לא מחזיר alerts ב-endpoint נפרד באופן עקבי — לפעמים זה חלק מה-server response. בדוק עם curl ישיר קודם.

### Workflow: SSL expiry monitoring

**Trigger:** Cron, every Sunday

```
┌─ Cron (Sunday 09:00)
├─ HTTP: get TOKEN
├─ HTTP: GET /server                 → all servers
├─ Loop servers → Loop apps:
│   ├─ HTTP: GET /app/{id}           → including SSL info
│   ├─ Function: parse SSL expiry date
│   ├─ IF expiry < 30 days:
│   │   └─ Add to "needs attention" list
├─ Aggregate
└─ Send report
```

### Workflow: Disk space alerting

**Trigger:** Cron, every 4 hours

```
┌─ Cron (every 4h)
├─ HTTP: get TOKEN
├─ HTTP: GET /server
├─ Loop servers:
│   ├─ HTTP: GET /server/{id}/disk_usage
│   ├─ IF usage > 85%:
│   │   ├─ Slack alert: "[Server X] disk 87% — investigate"
│   │   └─ (אופציונלי) IF usage > 95%: PagerDuty trigger
```

### Workflow: Auto-backup לפני deployment

**Trigger:** Webhook מ-GitHub Actions / GitLab CI

```
┌─ Webhook IN (with: app_id, deployment_sha)
├─ HTTP: get TOKEN
├─ HTTP: POST /app/manage/backup    → trigger backup
├─ Poll: get backup status until complete
├─ Save: backup_id, timestamp → Airtable / DB
└─ Webhook OUT → continue deployment
```

---

## 3. Make.com (Integromat) scenarios

ה-Make.com Custom App של Digitizer (אם בנוי) יכול לעטוף את ה-API ב-modules נוחים יותר. אבל גם בלי app מותאם, HTTP module עובד.

### תבנית בסיסית:

```
[HTTP — Make a request]
  URL: https://api.cloudways.com/api/v1/oauth/access_token
  Method: POST
  Body: email=...&api_key=...
  → extract access_token

[HTTP — Make a request]
  URL: https://api.cloudways.com/api/v1/server
  Method: GET
  Headers: Authorization: Bearer {{1.access_token}}
  → iterate

[Iterator]
  → for each server:

[Router]
  → branch by status / size / region

[Slack / Email / Airtable]
  → output
```

---

## 4. Claude Code headless

באוטומציה דרך `claude -p` (headless mode), ה-MCP server ממשיך לעבוד כרגיל. שימושי כאשר:

- אתה רוצה ש-Claude יחליט מה לעשות (לא just rules-based)
- האוטומציה כוללת analysis טקסטואלי (סיכום report, ניסוח email)
- יש ערך ב-natural language interpretation של הנתונים

**דוגמה: Daily summary בייצור**

```bash
#!/bin/bash
# /opt/cw-mcp/scripts/daily-summary.sh

# כאן Claude קורא ל-MCP tools בעצמו ומייצר summary
claude -p "
Generate today's Cloudways health summary.
Check all servers (list_servers), get alerts, and identify:
1. Any server not in Running status
2. Any disk > 80%
3. Any SSL expiring within 30 days
4. Top 3 apps by traffic in the past 24h

Output in Hebrew, markdown format, sent to /tmp/cw-summary.md
"

# שלח את הסיכום ל-Slack
curl -X POST -H 'Content-type: application/json' \
     --data "{\"text\":\"$(cat /tmp/cw-summary.md)\"}" \
     "$SLACK_WEBHOOK_URL"
```

**הערה:** headless דורש ש-Claude Code יודע איך לפנות ל-MCP server מקומי. תוודא שה-MCP config מוגדר ב-`~/.claude.json` של המשתמש שמריץ את ה-cron.

---

## 5. Airtable as state store

לאוטומציות שמייצרות הרבה נתונים (audit results, alerts log, deployment history), Airtable הוא state store טוב. דפוס:

**Table: cloudways_servers**

| Field | Type |
|-------|------|
| cw_id | Number (PK) |
| label | Text |
| provider | Single select |
| region | Text |
| size | Text |
| status | Single select |
| last_check | DateTime |
| client | Linked to Clients table |
| monthly_cost_usd | Number (formula or manual) |

**Table: cloudways_alerts**

| Field | Type |
|-------|------|
| date | DateTime |
| server | Linked |
| app | Linked |
| severity | Single select (P0/P1/P2) |
| issue | Text |
| status | Single select (open/investigating/resolved) |
| resolution_notes | Long text |

**Sync:** n8n / Make scenario כל שעה: pull state מ-Cloudways → upsert ל-Airtable. הצוות מקבל view חי.

---

## 6. Slack notifications — דפוסים מומלצים

**Slack message types:**

| חומרה | פורמט | תגובה צפויה |
|--------|--------|--------------|
| P0 (server down, SSL expired) | `<!channel>` + 🚨 | תגובה תוך 30 דקות |
| P1 (disk > 90%, SSL < 7d) | `<!here>` + ⚠️ | תגובה תוך 4 שעות |
| P2 (info: backup completed) | רגיל + ✅ | אין צורך בתגובה |

**דוגמת payload:**

```json
{
  "text": "🚨 P0: Cloudways alert",
  "blocks": [
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "*Server:* prod-shop-il (1234567)\n*Issue:* SSL expired 2 hours ago\n*Affected apps:* shop.example.co.il, admin.example.co.il\n*Action:* renew_letsencrypt — pending human approval"
      }
    },
    {
      "type": "actions",
      "elements": [
        {
          "type": "button",
          "text": {"type": "plain_text", "text": "Open Cloudways"},
          "url": "https://platform.cloudways.com/server/1234567"
        }
      ]
    }
  ]
}
```

---

## 7. Multi-account management

אם Digitizer מנהל מספר חשבונות Cloudways (לקוחות שונים, חשבונות נפרדים) — לא להחזיק את כל ה-keys באותו `.env`.

**אסטרטגיה:**

1. **Account-per-client:** כל לקוח ב-Infisical project נפרד
2. **MCP instance per account:** הפעל מספר MCP servers על פורטים שונים (7000, 7001, 7002...) עם credentials שונים
3. **n8n credentials:** הגדר אותם ב-n8n credentials store, לא ב-workflow
4. **Make.com connections:** אותו דבר ב-Make

**דוגמה ל-Claude Desktop config מ multi-account:**

```json
{
  "mcpServers": {
    "cloudways-account1": { "command": "npx", "args": ["-y", "mcp-remote", "http://127.0.0.1:7000/mcp", ...] },
    "cloudways-account2": { "command": "npx", "args": ["-y", "mcp-remote", "http://127.0.0.1:7001/mcp", ...] }
  }
}
```

Claude יראה כל אחד מהם כ-MCP נפרד עם prefix משלו.

---

## 8. Rate limiting considerations

ה-MCP מוגבל ל-`RATE_LIMIT_REQUESTS=90` בדקה (default). באוטומציה — שים לב:

- **Daily summary** של 50 servers + 200 apps = ~250 reads. במצב רגיל — OK.
- **Bulk audit** של חשבון מאסיבי — יכול לחצות מהר. השתמש ב-`rate_limit_status` כדי לבדוק.
- **Cloudways API ישיר** יש לו rate limit נפרד (יותר נדיב — בערך 180/min). אם MCP rate limit מפריע, באוטומציה מסיבית — עבור ל-API ישיר.

---

## דפוסי anti-pattern (אל תעשה)

❌ **Auto-execute write operations** בלי human-in-the-loop. גם אם נראה בטוח, אל תעשה.

❌ **לוגינג של credentials.** וודא שה-n8n / Make logs לא מציגים את ה-API key. השתמש ב-credential store הפנימי.

❌ **MCP server חשוף לאינטרנט.** אם אתה רוצה גישה מרחוק — Cloudflare Access / WireGuard, לא public port.

❌ **Use MCP for monitoring loops.** ל-monitoring continuous (כל 30 שניות), עבור ל-API ישיר. ה-MCP overhead לא שווה.

❌ **One MCP server לכל הצוות.** סוכן אבטחה: כל עובד צריך MCP server משלו עם credentials משלו, או SSO proxy מותאם. שיתוף MCP server = שיתוף credentials.
