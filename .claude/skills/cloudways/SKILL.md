---
name: cloudways
description: Manage Cloudways cloud hosting via the Cloudways MCP server — list servers and applications, inspect server/app details and settings, read monitoring metrics, app credentials, SSH keys, team members, alerts, projects, and account info. Use when the user asks about their Cloudways servers, apps, hosting infrastructure, deployment environment, or mentions Cloudways. Tools are exposed as mcp__cloudways__* and are currently read-only.
---

# Cloudways MCP

Operate a user's [Cloudways](https://www.cloudways.com/) managed hosting account
through the Cloudways MCP server. The server wraps the Cloudways API and exposes
read-only tools for inspecting servers, applications, monitoring, and account
configuration.

Official reference: [How to Use Cloudways MCP Server](https://support.cloudways.com/en/articles/14654372-how-to-use-cloudways-mcp-server-for-ai-based-server-management)

## Before you start

1. **Confirm the MCP server is connected.** The tools appear as
   `mcp__cloudways__*` (the prefix depends on the server name in the MCP config;
   it may differ if the user named it something else). If no such tools are
   available, the server isn't connected — see `references/setup.md` to set it
   up, and stop until it's connected. Do not guess answers about a user's
   account; everything must come from a tool call.
2. **Verify connectivity and auth first.** Call `ping` and then `customer_info`
   before any other operation. `ping` confirms the server is reachable;
   `customer_info` confirms the Cloudways credentials are valid and shows whose
   account you're acting on. If either fails, it's almost always an
   authentication problem — point the user to `references/setup.md`.
3. **Read-only today.** The server currently supports read/list operations only.
   It cannot create, restart, scale, deploy, or delete anything. If the user
   asks for a write action (restart server, deploy app, scale, change settings),
   tell them this isn't supported yet and offer to gather the relevant
   read-only context instead.

## Core workflow

Cloudways is hierarchical: **account → servers → applications**. Most questions
resolve by walking down that tree.

1. **Find the server.** Call `list_servers` to enumerate servers. Match the
   user's description (label, public IP, provider, region) to a server and note
   its server ID. Use `get_server` for full detail on one server.
2. **Find the application.** Applications live on a server. Use the app listing
   for that server, then `get_app_details` for a specific app. Note that app IDs
   are scoped to their server, so most app tools need both the server ID and the
   app ID.
3. **Answer the question.** Pick the narrowest tool that satisfies the request
   (see `references/tools.md` for the full catalog) rather than dumping
   everything. For metrics use the monitoring tools; for connection info use the
   credentials tools.

## Conventions

- **Always pass the IDs you discovered.** Server and app tools require numeric
  IDs from `list_servers` / the app listing — never invent them. If you can't
  uniquely identify the server or app the user means, list the candidates and
  ask which one.
- **Respect rate limits.** The server enforces per-customer rate limiting. If a
  call fails with a rate-limit error, check `rate_limit_status` and wait rather
  than retrying in a tight loop. Don't fan out dozens of calls at once.
- **Treat credentials as sensitive.** `get_app_credentials` and SSH key tools
  return secrets (DB passwords, SSH/SFTP logins). Surface them only when the
  user explicitly asked, never log them casually, and don't echo them into
  files or commit them.
- **Read-only data is safe to summarize.** Server lists, monitoring, regions,
  providers, and sizes can be freely presented and compared.

## When the user wants to set up or run the MCP server

See `references/setup.md` for prerequisites (Python 3.11+, Redis), environment
variables, the local endpoint (`http://127.0.0.1:7000/mcp`), how to get
Cloudways API credentials, and how to register the server with the MCP client.

## Tool catalog

`references/tools.md` lists the available tools grouped by category (basic /
account, server management, application management, security & access). Consult
it to choose the right tool. The live server exposes 40+ tools — if a tool you
need isn't documented there, inspect the actual `mcp__cloudways__*` tools
available in the session, since the deployed version may add or rename tools.
