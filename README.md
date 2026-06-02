# cloudways-mcp

A [Claude Code Agent Skill](https://docs.claude.com/en/docs/claude-code/skills)
for managing [Cloudways](https://www.cloudways.com/) managed hosting through the
**Cloudways MCP server**.

The skill teaches Claude how to drive the Cloudways MCP tools: discovering
servers and applications, reading monitoring metrics, fetching app credentials
and settings, and inspecting account-level configuration (projects, team
members, SSH keys, providers, regions, sizes). All operations are currently
**read-only**.

## Contents

```
.claude/skills/cloudways/
├── SKILL.md                 # main skill: workflow, conventions, safety
└── references/
    ├── tools.md             # catalog of available MCP tools by category
    └── setup.md             # run the server + connect a client
```

## Using the skill

The skill activates automatically when you ask Claude about your Cloudways
servers, apps, or hosting — as long as the `cloudways` skill is available and
the Cloudways MCP server is connected (tools appear as `mcp__cloudways__*`).

## Connecting the MCP server

See [`.claude/skills/cloudways/references/setup.md`](.claude/skills/cloudways/references/setup.md)
and the example [`.mcp.json.example`](.mcp.json.example). You'll need your
Cloudways account email and API key (Cloudways Platform → Account → API Keys).

## References

- [How to Use Cloudways MCP Server for AI-Based Server Management](https://support.cloudways.com/en/articles/14654372-how-to-use-cloudways-mcp-server-for-ai-based-server-management) (official)
- [`aphraz/cw-mcp`](https://github.com/aphraz/cw-mcp) (upstream MCP server)
