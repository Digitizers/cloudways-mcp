# cloudways-mcp

[Claude Code Agent Skill](https://docs.claude.com/en/docs/claude-code/skills) תפעולי לניהול תשתית [Cloudways](https://www.cloudways.com/) דרך ה-**Cloudways MCP server** ([`aphraz/cw-mcp`](https://github.com/aphraz/cw-mcp), המימוש שמופיע ב[תיעוד הרשמי של Cloudways](https://support.cloudways.com/en/articles/14654372-how-to-use-cloudways-mcp-server-for-ai-based-server-management)).

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

- **לא read-only.** בניגוד לחומר השיווק של Cloudways, ה-MCP כולל גם פעולות **write** רבות (restart, backup, restore, SSL, git, IP whitelist). כל פעולת write דורשת **אישור מפורש** לפי הפטרן ב-`SKILL.md`.
- **Multi-account.** כל חשבון מחובר כ-connection נפרד עם prefix משלו (`mcp__cloudways-clientA__*`). הסקיל מחייב לזהות את החשבון הנכון לפני כל פעולה, ואוסר ערבוב IDs/credentials בין חשבונות.
- **Source of truth.** אם הקטלוג כאן סותר את מה שה-MCP החי מחזיר — **ה-MCP החי קובע**; עדכן את הקטלוג בהתאם.

## הפעלה

הסקיל מופעל אוטומטית כשמדברים על Cloudways, כל עוד הסקיל זמין וה-MCP server מחובר (הכלים מופיעים כ-`mcp__cloudways*__*`). להקמה וחיבור — ראה [`installation.md`](.claude/skills/cloudways-mcp/references/installation.md) ואת [`.mcp.json.example`](.mcp.json.example). תזדקק ל-email + API key של כל חשבון (Cloudways Platform → Account → API).

## מקורות

- [How to Use Cloudways MCP Server for AI-Based Server Management](https://support.cloudways.com/en/articles/14654372-how-to-use-cloudways-mcp-server-for-ai-based-server-management) (רשמי)
- [`aphraz/cw-mcp`](https://github.com/aphraz/cw-mcp) (ה-MCP server)
</content>
