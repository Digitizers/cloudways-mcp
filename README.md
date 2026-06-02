# cloudways-mcp

[Claude Code Agent Skill](https://docs.claude.com/en/docs/claude-code/skills) תפעולי לניהול תשתית [Cloudways](https://www.cloudways.com/) דרך ה-**Cloudways MCP server**. תומך בשתי דרכי חיבור:

1. **רשמי — Cloudways (Remote) MCP** *(מומלץ)*: MCP מתארח על ידי Cloudways (Q2 2026). מקור החיבור הוא ה[מאמר הרשמי](https://support.cloudways.com/en/articles/14654372-how-to-use-cloudways-mcp-server-for-ai-based-server-management).
2. **קהילתי — self-hosted `cw-mcp`**: שרת Python+Redis לוקאלי. ⚠️ הריפו `github.com/aphraz/cw-mcp` **כרגע 404** (נמחק/הוסתר) — דורש fork/עותק זמין.

הסקיל בנוי לעבודה היומיומית של Digitizer: monitoring, תחזוקה, onboarding/audit ללקוחות, ואוטומציות — עם **תמיכה בכמה חשבונות Cloudways** וכללי בטיחות לפעולות write.

## מבנה

```
.claude/skills/cloudways-mcp/
├── SKILL.md                          # ליבה: safety rules, multi-account, confirmation, דפוסים
└── references/
    ├── installation.md               # הקמת השרת (Python+Redis) + חיבור Claude + multi-account config
    ├── tools-catalog.md              # קטלוג 43+ כלים, מתויגים R / W / W!
    ├── workflows-monitoring.md       # תרחישי ניטור (read-only)
    ├── workflows-maintenance.md      # תרחישי תחזוקה (write — דורש confirmation)
    ├── workflows-onboarding.md       # audit/onboarding ללקוח חדש
    └── workflows-automation.md       # n8n / Make / Claude Code headless / multi-account
```

## נקודות מפתח

- **read-only מול write — לא ודאי.** מקורות סותרים אם הגרסה הנוכחית כוללת פעולות write או רק read. הסקיל נוקט זהירות: כל כלי שמשנה מצב חייב **אישור מפורש** (`SKILL.md`) — כלל שבטוח גם אם אין כלי write בפועל. תמיד אמת מול השרת החי.
- **Multi-account.** כל חשבון מחובר כ-connection נפרד עם prefix משלו (`mcp__cloudways-clientA__*`). הסקיל מחייב לזהות את החשבון הנכון לפני כל פעולה, ואוסר ערבוב IDs/credentials בין חשבונות.
- **Source of truth.** אם הקטלוג כאן סותר את מה שה-MCP החי מחזיר — **ה-MCP החי קובע**; עדכן את הקטלוג בהתאם.

## הפעלה

הסקיל מופעל אוטומטית כשמדברים על Cloudways, כל עוד הסקיל זמין וה-MCP server מחובר (הכלים מופיעים כ-`mcp__cloudways*__*`). להקמה וחיבור — ראה [`installation.md`](.claude/skills/cloudways-mcp/references/installation.md) ואת [`.mcp.json.example`](.mcp.json.example). תזדקק ל-email + API key של כל חשבון (Cloudways Platform → Account → API).

## מקורות

- [How to Use Cloudways MCP Server for AI-Based Server Management](https://support.cloudways.com/en/articles/14654372-how-to-use-cloudways-mcp-server-for-ai-based-server-management) (רשמי — מקור האמת לחיבור)
- [Cloudways Roadmap](https://www.cloudways.com/en/roadmap.php) (מציין את "Cloudways Remote MCP", Q2 2026)
- `aphraz/cw-mcp` — השרת הקהילתי (⚠️ הריפו כרגע 404)
</content>
