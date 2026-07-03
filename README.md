# Cloudways MCP — Claude Code & OpenClaw Skill

[![CI](https://github.com/Digitizers/cloudways-mcp/actions/workflows/ci.yml/badge.svg)](https://github.com/Digitizers/cloudways-mcp/actions/workflows/ci.yml)
![Claude Code Skill](https://img.shields.io/badge/Claude_Code-Skill-d97757)
![OpenClaw Skill](https://img.shields.io/badge/OpenClaw-Skill-purple)
![Cloudways](https://img.shields.io/badge/Cloudways-MCP-2b6cb0)
![License: MIT](https://img.shields.io/badge/License-MIT-green)
![Version](https://img.shields.io/badge/version-1.2.1-blue)

A production-grade **Claude Code & OpenClaw skill** for managing [Cloudways](https://digitizer.li/cloudways) infrastructure through the official **Cloudways (Remote) MCP** — connect directly to the hosted server, run one or many accounts, with a safety rule on every write.

This is not just a tool reference. It is an operational playbook for running Cloudways infrastructure responsibly: server and app monitoring, routine maintenance, client onboarding/audit, and automation — with write-confirmation guardrails on anything that changes state. The connection source of truth is the [official Cloudways article](https://support.cloudways.com/en/articles/14654372-how-to-use-cloudways-mcp-server-for-ai-based-server-management).

## Part of the Aura Design Engine

These are the free skills behind [**Aura**](https://my-aura.app) — one AI web-agency lifecycle you can run standalone or orchestrate across a whole client fleet from a single dashboard.

| Stage | Skill | Role |
| --- | --- | --- |
| 🎨 Build | [claude-elementor-pro](https://github.com/Digitizers/claude-elementor-pro) | Design & build sites inside Elementor |
| 🔎 Audit + Content | [wordpress-api-pro](https://github.com/Digitizers/wordpress-api-pro) | REST content ops, SEO & site audits |
| 🖥 Host | [**cloudways-mcp** ← you are here](https://github.com/Digitizers/cloudways-mcp) · [hostinger-mcp](https://github.com/Digitizers/hostinger-mcp) | Provision & operate the infrastructure |

**→ Orchestrate all of it across your client fleet with [Aura](https://my-aura.app)** — governed agent ops with approvals and a full audit trail on top of these skills.

## Solo, or with Aura

This skill works standalone — point it at your own Cloudways account and run it from Claude Code. That's the **solo path**: you drive one account, and the write-confirmation guardrail on every state change keeps you safe.

The **Aura path** is the same skill orchestrated across a whole client fleet — [Aura](https://my-aura.app) proxies these provider ops through its governed MCP gateway, adds queued human approvals and a full audit trail, and runs them across every server and app you manage from one dashboard, no per-account token juggling.

| | Solo (this skill) | With Aura |
| --- | --- | --- |
| Scope | one account you hold keys for | every client account in your org |
| Approvals | your own write-confirmation | queued admin approval + audit trail |
| Best for | your own infra, quick ops | agencies operating many clients |

Use it solo for your own boxes; reach for Aura when you operate a fleet on clients' behalf.

## Features

- ✅ **Official Cloudways (Remote) MCP** — connect directly to the hosted server at `mcp.cloudways.com`; no self-hosting, no proxy to maintain.
- ✅ **Complete tool catalog** — the full official toolset across servers, apps, services, DNS, CDN, Git, SSH keys, and analytics, tagged R / W / W!.
- ✅ **Write-confirmation safety** — every state-changing operation requires explicit target + action confirmation; destructive operations double-confirm.
- ✅ **Multi-account** — one connection per Cloudways account, each with its own prefix; no cross-account ID or credential mixing.
- ✅ **Operational playbooks** — monitoring, maintenance, client onboarding/audit, and n8n / Make / headless automation.
- ✅ **Honest gaps** — SSL, IP-whitelisting, and team management are flagged as Cloudways UI / direct-API (not MCP tools), never faked.

## Structure

```
.claude/skills/cloudways-mcp/
├── SKILL.md                          # core: safety rules, multi-account, confirmation, patterns
└── references/
    ├── installation.md               # Claude connection (official MCP) + multi-account config
    ├── tools-catalog.md              # official tool catalog, tagged R / W / W!
    ├── workflows-monitoring.md       # monitoring scenarios (read-only)
    ├── workflows-maintenance.md      # maintenance scenarios (write — requires confirmation)
    ├── workflows-onboarding.md       # audit/onboarding for a new client
    └── workflows-automation.md       # n8n / Make / Claude Code headless / multi-account
```

## Key points

- **Write safety.** Any tool that changes state requires **explicit confirmation** of the target and action (`SKILL.md`); destructive operations (delete, restore, scale) double-confirm. The tool catalog is verified against the official article — see `tools-catalog.md` for the R / W / W! tags.
- **Multi-account.** Each account is connected as a separate connection with its own prefix (`mcp__cloudways-clientA__*`). The skill requires identifying the correct account before any operation, and prohibits mixing IDs/credentials across accounts.
- **Source of truth.** If the catalog here conflicts with what the live MCP returns — **the live MCP wins**; update the catalog accordingly.

## Activation

The skill activates automatically when Cloudways is discussed, as long as the skill is available and the MCP server is connected (the tools appear as `mcp__cloudways*__*`). For setup and connection — see [`installation.md`](.claude/skills/cloudways-mcp/references/installation.md) and [`.mcp.json.example`](.mcp.json.example).

The official hosted MCP is at `https://mcp.cloudways.com/mcp/`. Authenticate with per-account request headers: `X-CW-Email`, `X-CW-Api-Key`, and `X-Mcp-Host` (the client name, e.g. `claude-code`). Generate the API key at [platform.cloudways.com](https://platform.cloudways.com/) → API Integration. One connection per account, named `cloudways-<client>`.

## Sources

- [How to Use Cloudways MCP Server for AI-Based Server Management](https://support.cloudways.com/en/articles/14654372-how-to-use-cloudways-mcp-server-for-ai-based-server-management) (official — the source of truth for the connection)
- [Cloudways Roadmap](https://www.cloudways.com/en/roadmap.php) (lists "Cloudways Remote MCP", Q2 2026)
- [Cloudways API reference](https://developers.cloudways.com/)

> The tool catalog is verified against the official support article. If it ever conflicts with what the live `mcp__cloudways*__*` tools return, the live MCP is authoritative.

## Links

- **Repository:** https://github.com/Digitizers/cloudways-mcp
- **OpenClaw:** https://openclaw.ai
- **Cloudways MCP article:** https://support.cloudways.com/en/articles/14654372-how-to-use-cloudways-mcp-server-for-ai-based-server-management
- **Digitizer:** https://www.digitizer.studio
- **Getting started with Cloudways:** https://www.digitizer.studio/cloudways-beginners-guide

## License

MIT

---

Built with ❤️ for OpenClaw by [Digitizer](https://www.digitizer.studio)
