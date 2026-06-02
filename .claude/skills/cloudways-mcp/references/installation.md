# Installation — Cloudways MCP Server

מדריך התקנה והגדרה מלאים של ה-MCP server `aphraz/cw-mcp` עבור Claude Desktop / Claude Code.

> **חשוב להבין:** Cloudways לא מפעילים את ה-MCP server עבורך. ה-server רץ **אצלך** (לוקאלית או על VPS) ופונה ל-Cloudways API בשמך. המשמעות: אתה אחראי על האבטחה, ה-credentials, ועל זה שה-server רץ ברציפות כשאתה רוצה להשתמש בו.

---

## Prerequisites

- **Python 3.11+** (לא 3.10 — ה-`asyncio.TaskGroup` שה-MCP משתמש בו דורש 3.11)
- **Redis** רץ (לוקאלית או remote). שימושים: storage מוצפן של tokens, rate limiting, session isolation.
- **חשבון Cloudways** עם API access מופעל
- **Node.js 18+** עבור `npx mcp-remote` (אם מתחברים מ-Claude Desktop)

---

## שלב 1 — הוצאת API credentials מ-Cloudways

1. התחבר ל-Cloudways Platform
2. למעלה מימין: **Account → API**
3. צור/העתק את `API Key`
4. רשום את ה-email של החשבון

> ה-API key מאפשר **כל** הפעולות שהחשבון יכול לבצע ב-UI. שמור אותו כמו סיסמה.

---

## שלב 2 — Clone והתקנת ה-MCP server

```bash
# Clone
git clone https://github.com/aphraz/cw-mcp.git
cd cw-mcp

# Virtual env
python3 -m venv venv
source venv/bin/activate    # macOS/Linux
# או: venv\Scripts\activate  # Windows

# Dependencies
pip install -r requirements.txt
```

---

## שלב 3 — Redis

### לוקאלית (macOS):
```bash
brew install redis
brew services start redis
# בדיקה:
redis-cli ping   # מצופה: PONG
```

### לוקאלית (Linux):
```bash
sudo apt install redis-server
sudo systemctl enable --now redis-server
redis-cli ping
```

### Docker (כל פלטפורמה):
```bash
docker run -d --name cw-mcp-redis -p 6379:6379 redis:7-alpine
```

---

## שלב 4 — Environment variables

צור קובץ `.env` ב-root של ה-repo (וודא שהוא ב-`.gitignore`):

```bash
# Encryption key — חובה. ייצור פעם אחת ושמור:
ENCRYPTION_KEY=$(python -c 'from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())')

# Redis URL
REDIS_URL=redis://localhost:6379/0

# Performance
HTTP_POOL_SIZE=500
REDIS_POOL_SIZE=500

# Rate limiting (requests per minute, per customer)
RATE_LIMIT_REQUESTS=90
RATE_LIMIT_WINDOW=60

# Logging
LOG_LEVEL=INFO
LOG_FORMAT=console
```

**שמור את `ENCRYPTION_KEY` במקום בטוח.** אם תאבד אותו, כל ה-tokens השמורים ב-Redis לא יהיו ניתנים לפענוח. אם תחליף אותו, כל המשתמשים יצטרכו לבצע auth מחדש.

> **הקשר Digitizer:** הקשר ישיר ל-Infisical / OpenBao שאתה בודק — `ENCRYPTION_KEY` הוא בדיוק הסוג של secret שמתאים להישמר שם, לא ב-`.env` plain text.

---

## שלב 5 — הפעלת ה-server

```bash
source venv/bin/activate
python cw-mcp.py
```

מצופה לראות:
```
==================================================
🚀 Cloudways MCP Server
==================================================
INFO: Started server process [XXXX]
INFO: Uvicorn running on http://127.0.0.1:7000
```

ה-endpoint הוא `http://127.0.0.1:7000/mcp`.

### בדיקת חיים מהירה:
```bash
curl -v "http://127.0.0.1:7000/mcp/"
# צריך להחזיר 200 או 405 (לא 404/connection refused)
```

---

## שלב 6 — חיבור ל-Claude Desktop / Claude Code

ה-MCP server החי הוא HTTP. כדי לחבר אותו ל-Claude (שמדבר stdio), משתמשים ב-`mcp-remote` כפרוקסי.

### Claude Desktop config

מיקום הקובץ:
- **macOS:** `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Windows:** `%APPDATA%\Claude\claude_desktop_config.json`
- **Linux:** `~/.config/Claude/claude_desktop_config.json`

ערוך והוסף ל-`mcpServers`:

```json
{
  "mcpServers": {
    "cloudways": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "http://127.0.0.1:7000/mcp",
        "--header",
        "x-cloudways-email: ${CLOUDWAYS_EMAIL}",
        "--header",
        "x-cloudways-api-key: ${CLOUDWAYS_API_KEY}"
      ],
      "env": {
        "CLOUDWAYS_EMAIL": "your-account@example.com",
        "CLOUDWAYS_API_KEY": "your-cloudways-api-key"
      }
    }
  }
}
```

אחרי שמירה — **Quit & Reopen** ל-Claude Desktop (לא רק לסגור חלון).

### Claude Code config

עבור Claude Code, הוסף את אותו blob ל-`~/.claude.json` או דרך `claude mcp add` (תלוי בגרסה).

גרסאות Claude Code חדשות תומכות גם ב-transport `http` ישיר (בלי `mcp-remote` כפרוקסי), עם headers:

```bash
claude mcp add --transport http cloudways http://127.0.0.1:7000/mcp \
  --header "x-cloudways-email: your-account@example.com" \
  --header "x-cloudways-api-key: your-cloudways-api-key"
```

---

## Multi-account configuration — כמה חשבונות Cloudways

ל-Digitizer יש **כמה חשבונות Cloudways**. ה-MCP server של `aphraz/cw-mcp` הוא **multi-tenant** — הוא קורא את ה-credentials מתוך ה-headers של **כל request**, ומבודד sessions ב-Redis לפי customer. המשמעות: לרוב **לא צריך כמה server instances** — מספיק להגדיר ב-Claude **connection אחד לכל חשבון**, כולם מצביעים לאותו URL, רק עם headers שונים.

### גישה A — connection לכל חשבון (מומלצת)

server instance אחד (`http://127.0.0.1:7000/mcp`), וכמה entries ב-`mcpServers`, אחד לכל חשבון. שמות תיאוריים לפי לקוח — הם הופכים ל-prefix של ה-tools (`mcp__cloudways-clientA__*`):

```json
{
  "mcpServers": {
    "cloudways-clientA": {
      "command": "npx",
      "args": [
        "-y", "mcp-remote", "http://127.0.0.1:7000/mcp",
        "--header", "x-cloudways-email: ${CW_A_EMAIL}",
        "--header", "x-cloudways-api-key: ${CW_A_API_KEY}"
      ],
      "env": { "CW_A_EMAIL": "clientA@example.com", "CW_A_API_KEY": "..." }
    },
    "cloudways-clientB": {
      "command": "npx",
      "args": [
        "-y", "mcp-remote", "http://127.0.0.1:7000/mcp",
        "--header", "x-cloudways-email: ${CW_B_EMAIL}",
        "--header", "x-cloudways-api-key: ${CW_B_API_KEY}"
      ],
      "env": { "CW_B_EMAIL": "clientB@example.com", "CW_B_API_KEY": "..." }
    }
  }
}
```

(ב-Claude Code עם transport `http` ישיר — אותו רעיון, entry לכל חשבון עם headers משלו.)

### גישה B — instance לכל חשבון (בידוד חזק יותר)

אם אתה רוצה הפרדת תהליכים מלאה (credentials מ-Infisical projects נפרדים, או rate-limit/לוגים נפרדים לכל לקוח), הפעל כמה server instances על פורטים שונים — כל אחד עם `.env` משלו:

```bash
PORT=7000 ENV_FILE=.env.clientA  python cw-mcp.py   # clientA
PORT=7001 ENV_FILE=.env.clientB  python cw-mcp.py   # clientB
```

ואז ב-config כל connection מצביע לפורט שלו (`:7000`, `:7001`...). שקול systemd unit / Docker Compose service לכל instance.

### כללי בטיחות ל-multi-account (חובה)

- **שמות עקביים:** השתמש ב-prefix אחיד `cloudways-<client>` כדי ש-Claude (וגם אתה) תזהו מיד לאיזה חשבון שייך כל tool.
- **Secrets בנפרד:** אל תשים את כל ה-keys באותו `.env`. עדיף Infisical/OpenBao project לכל לקוח (ראה checklist האבטחה למטה).
- **אל תערבב:** לעולם אל תשתמש באותו API key לשני חשבונות, ואל תיקח server/app ID מחשבון אחד מול connection של אחר.
- **runtime:** התנהגות Claude בזמן ריצה (זיהוי חשבון, חיפוש חוצה-חשבונות, אישורי write פר-חשבון) מתועדת ב-`SKILL.md` סעיף **Multi-account**.

---

## שלב 7 — בדיקה ב-Claude

פתח שיחה ב-Claude Desktop. בקש:
```
list my Cloudways servers
```

מצופה: רשימת השרתים שלך. אם מקבל error:

| Error | משמעות | פתרון |
|--------|--------|--------|
| `connection refused` | ה-server לא רץ | `python cw-mcp.py` |
| `401 / authentication` | credentials לא נכונים | בדוק email + API key |
| `429 / rate limited` | הגעת ל-cap | חכה דקה / הגדל `RATE_LIMIT_REQUESTS` |
| `redis connection error` | Redis לא רץ | `redis-cli ping` → להפעיל |
| `cryptography invalid token` | `ENCRYPTION_KEY` שונה ממה ששימש לקודינג | החזר את ה-key הישן או נקה Redis |

לבדיקה ישירה של credentials מול ה-API (בלי לעבור דרך MCP):

```bash
curl -X POST "https://api.cloudways.com/api/v1/oauth/access_token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "email=YOUR_EMAIL&api_key=YOUR_API_KEY"
```

---

## הפעלה כשירות (production-grade)

### עם systemd (Linux):

צור `/etc/systemd/system/cw-mcp.service`:

```ini
[Unit]
Description=Cloudways MCP Server
After=network.target redis.service

[Service]
Type=simple
User=cw-mcp
WorkingDirectory=/opt/cw-mcp
EnvironmentFile=/opt/cw-mcp/.env
ExecStart=/opt/cw-mcp/venv/bin/python cw-mcp.py
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable --now cw-mcp
sudo systemctl status cw-mcp
```

### עם Docker Compose (מומלץ אם ה-VPS שלך כבר רץ n8n על Docker):

```yaml
services:
  cw-mcp-redis:
    image: redis:7-alpine
    restart: unless-stopped
    volumes:
      - cw-mcp-redis:/data

  cw-mcp:
    build: .
    restart: unless-stopped
    depends_on: [cw-mcp-redis]
    env_file: .env
    environment:
      REDIS_URL: redis://cw-mcp-redis:6379/0
    ports:
      - "127.0.0.1:7000:7000"

volumes:
  cw-mcp-redis:
```

> **בטיחות:** `ports: 127.0.0.1:7000:7000` — bind רק ל-loopback, לא לאינטרנט. אם אתה רוצה גישה מרחוק, השם תוכל מאחורי Nginx + Cloudflare Access או דומה, ולא תחשוף את הפורט ישירות.

---

## Self-hosting על Cloudways עצמו

אפשרי, ואפילו אלגנטי. ה-VPS שאתה כבר משלם עליו יכול לארח גם את ה-MCP server של עצמו. שיקולים:
- אל תשים את זה על שרת production שמארח אפליקציות לקוח — שים על שרת operational/internal.
- חבר Cloudflare Access (Zero Trust) כדי שרק אתה תוכל לגשת ל-endpoint מרחוק.
- אל תפעיל את זה על שרת shared של פרויקטים שונים — credentials יחשפו בין הסביבות.

---

## אבטחה — checklist לפני שימוש בייצור

- [ ] `ENCRYPTION_KEY` נשמר ב-Infisical / OpenBao / 1Password (לא ב-plain `.env`)
- [ ] `.env` ב-`.gitignore`
- [ ] Redis לא חשוף לאינטרנט (`bind 127.0.0.1` או password protected)
- [ ] ה-port 7000 לא חשוף לאינטרנט (loopback only או מאחורי auth proxy)
- [ ] API key של Cloudways — נפרד לכל סביבה אם רלוונטי, ניתן לשנן/לבטל
- [ ] לוגים לא מודפסים עם credentials (`LOG_LEVEL=INFO` ולא `DEBUG` בייצור)
- [ ] Rate limit מוגדר — לא להשאיר default אם אתה משרת מספר משתמשים

---

## Troubleshooting

```bash
# האם ה-process רץ?
ps aux | grep cw-mcp

# האם הפורט תפוס?
lsof -i :7000

# האם Redis זמין?
redis-cli ping

# Logs (אם רץ ב-systemd):
journalctl -u cw-mcp -f

# Logs (אם רץ idiomatically):
# stdout — תפנה אותו ל-file אם אתה רץ ב-nohup
```

---

## Issues / contribute

ה-repo: `https://github.com/aphraz/cw-mcp`

לפני שאתה מדווח באג — נסה לחזור על הבעיה עם `LOG_LEVEL=DEBUG` ושמור את הלוג. ה-MCP server עדיין צעיר ובאופן יחסי לא יציב; ייתכן שתיתקל ב-edge cases.
