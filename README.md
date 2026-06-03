# cloudways-mcp

An operational [Claude Code Agent Skill](https://docs.claude.com/en/docs/claude-code/skills) for managing [Cloudways](https://www.cloudways.com/) infrastructure through the **official Cloudways (Remote) MCP server** — a Cloudways-hosted MCP you connect to directly (listed as "Cloudways Remote MCP", Q2 2026, on the [Cloudways roadmap](https://www.cloudways.com/en/roadmap.php)). The connection source of truth is the [official article](https://support.cloudways.com/en/articles/14654372-how-to-use-cloudways-mcp-server-for-ai-based-server-management).

The skill is built for day-to-day infrastructure management: monitoring, maintenance, client onboarding/audit, and automations — with **support for multiple Cloudways accounts** and safety rules for write operations.

## Structure

```
.claude/skills/cloudways-mcp/
├── SKILL.md                          # core: safety rules, multi-account, confirmation, patterns
└── references/
    ├── installation.md               # Claude connection (official MCP) + multi-account config
    ├── tools-catalog.md              # catalog of tools (illustrative), tagged R / W / W!
    ├── workflows-monitoring.md       # monitoring scenarios (read-only)
    ├── workflows-maintenance.md      # maintenance scenarios (write — requires confirmation)
    ├── workflows-onboarding.md       # audit/onboarding for a new client
    └── workflows-automation.md       # n8n / Make / Claude Code headless / multi-account
```

## Key points

- **read-only vs. write — uncertain.** Sources conflict on whether the current version includes write operations or read only. The skill errs on the side of caution: any tool that changes state requires **explicit confirmation** (`SKILL.md`) — a rule that is safe even if no write tool actually exists. Always verify against the live server.
- **Multi-account.** Each account is connected as a separate connection with its own prefix (`mcp__cloudways-clientA__*`). The skill requires identifying the correct account before any operation, and prohibits mixing IDs/credentials across accounts.
- **Source of truth.** If the catalog here conflicts with what the live MCP returns — **the live MCP wins**; update the catalog accordingly.

## Activation

The skill activates automatically when Cloudways is discussed, as long as the skill is available and the MCP server is connected (the tools appear as `mcp__cloudways*__*`). For setup and connection — see [`installation.md`](.claude/skills/cloudways-mcp/references/installation.md) and [`.mcp.json.example`](.mcp.json.example). You will need the email + API key for each account (Cloudways Platform → Account → API).

## Sources

- [How to Use Cloudways MCP Server for AI-Based Server Management](https://support.cloudways.com/en/articles/14654372-how-to-use-cloudways-mcp-server-for-ai-based-server-management) (official — the source of truth for the connection)
- [Cloudways Roadmap](https://www.cloudways.com/en/roadmap.php) (lists "Cloudways Remote MCP", Q2 2026)
- [Cloudways API reference](https://developers.cloudways.com/)

> Tool names in the catalog are illustrative — compiled from an earlier community MCP and not verified against the official server. The live `mcp__cloudways*__*` tools are authoritative.
</content>
