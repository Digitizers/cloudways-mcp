# Cloudways MCP — tool catalog

The Cloudways MCP server exposes 40+ read-only tools, grouped below. In a Claude
client they are surfaced with the server-name prefix, e.g.
`mcp__cloudways__list_servers`. Names here follow the upstream
[`aphraz/cw-mcp`](https://github.com/aphraz/cw-mcp) server; a deployed instance
may add, rename, or version tools, so always defer to the actual tools available
in the session when there's a conflict.

> All operations are currently **read-only**. There are no create/update/delete
> tools yet.

## Basic / account & connectivity

| Tool | Purpose |
| --- | --- |
| `ping` | Health check — confirm the MCP server is reachable. |
| `customer_info` | Authenticated account details; confirms credentials are valid. |
| `rate_limit_status` | Current rate-limit budget and remaining requests for your account. |
| `list_projects` | Projects configured under the account. |
| `list_providers` | Supported cloud/hosting providers (DigitalOcean, AWS, GCP, Vultr, Linode, …). |
| `list_regions` | Regions/data-center locations where resources can be provisioned. |
| `list_server_sizes` | Available server size / plan options. |

## Server management

| Tool | Purpose |
| --- | --- |
| `list_servers` | List all servers in the account (IDs, labels, IPs, provider, region, status). Start here. |
| `get_server` | Full detail for one server by ID. |
| (monitoring) | Server-level monitoring / performance metrics (CPU, memory, disk, bandwidth). |

Server tools key off the numeric **server ID** returned by `list_servers`.

## Application management

| Tool | Purpose |
| --- | --- |
| (app listing) | Applications hosted on a given server. |
| `get_app_details` | Detailed info for one application (server ID + app ID). |
| `get_app_credentials` | App access credentials — DB, SFTP/SSH logins. **Sensitive.** |
| `get_app_settings` | Application configuration / settings. |
| `get_app_monitoring_summary` | Application-level monitoring summary. |

Application tools are scoped to a server, so most require **both** the server ID
and the app ID.

## Security & access control

| Tool | Purpose |
| --- | --- |
| `get_team_members` | Team members and their access for the account. |
| `list_ssh_keys` | SSH keys configured for servers. **Sensitive.** |
| `get_alerts` | Configured alerts / notifications. |

## Notes

- Counts and exact names vary by server version; the upstream project advertises
  43+ tools across basic operations, server management, application management,
  and security/access control.
- When unsure which tool returns a given field, prefer the most specific
  detail/summary tool over a broad list call.
