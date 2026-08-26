---
name: levitate-connect-the-mcp-server
description: Connect an AI assistant to Levitate's hosted remote MCP server over OAuth, and know what the MCP surface reaches that the REST API does not.
api: Levitate MCP Server
base_url: https://mcp.levitate.ai/mcp
operations: []
generated: '2026-08-25'
method: generated
source: https://help.levitate.ai/article/589-levitate-mcp-server + live probe of https://mcp.levitate.ai/mcp (401 + RFC 9728 challenge, 2026-08-25)
---

# Connect an AI assistant to Levitate over MCP

Levitate runs a hosted **remote** MCP server. There is nothing to install — it is a URL an MCP client
POSTs to, authenticated with OAuth.

**Endpoint:** `https://mcp.levitate.ai/mcp` · **Transport:** streamable HTTP · **Auth:** OAuth 2.0
authorization code + PKCE against `https://login.levitate.ai`. No API keys.

## Prerequisite

The MCP server must be enabled on the account by a Levitate Success Specialist. Once it is, **any**
Levitate user may connect — no admin step per user.

Levitate's own warning, repeated because it matters: a Levitate account holds client PII. Do not
connect from a free tier of any AI assistant.

## Connecting

| Client | How |
|---|---|
| Claude Desktop | Settings → Connectors → search Levitate (verified app) |
| Claude Code | `claude mcp add --transport http levitate https://mcp.levitate.ai/mcp` |
| ChatGPT | Settings → Plugins → Browse plugins → Levitate (verified app) |
| Cursor | Settings → MCP → Add new MCP server, type **URL**, `https://mcp.levitate.ai/mcp` |
| VS Code (Copilot) | `.vscode/mcp.json` → `{"servers":{"levitate":{"type":"http","url":"https://mcp.levitate.ai/mcp"}}}` |
| Windsurf | Settings → MCP → Add Server, transport **HTTP** |
| Anything else | Remote server, transport HTTP, the same URL |

On first connect you are redirected to log in with your existing Levitate credentials and to authorize
the client. The session stays active until you disconnect or revoke it.

## What the discovery chain looks like

An unauthenticated POST to the endpoint returns `401` with:

```
WWW-Authenticate: Bearer resource_metadata="https://mcp.levitate.ai/.well-known/oauth-protected-resource/mcp"
```

That RFC 9728 document names `https://login.levitate.ai` as the authorization server, whose RFC 8414
metadata advertises PKCE (`S256`) and a dynamic client registration endpoint — so a compliant MCP
client negotiates the whole flow without hand configuration. Copies of every one of these documents
are saved under `well-known/`.

## What MCP reaches that REST does not

This is the important part. The REST Public API covers **only** contacts, key facts, companies and
notes. The MCP server additionally reaches:

- action items (by assignee, due date, completion status)
- donations and campaigns
- opportunity boards, stages and deals
- insurance policies
- contact timeline (emails, notes, calls, key dates)
- email drafts (needs Microsoft `Mail.ReadWrite` for O365/Outlook users)
- email campaigns and templates — create, update, schedule, send, list *(July 2026)*
- social posts — create, update, schedule, get, list, delete *(July 2026)*
- file upload via a two-step pre-signed URL flow *(July 2026)*
- live queries against 11+ connected integrations (HousecallPro results are live, not a daily snapshot)

None of those have a public REST endpoint. Conversely, key-fact writes, note creation and company
management are REST-only. See `mcp/levitate-tool-crosswalk.yml` for the operation-level mapping.

## Behaviour to expect

- Contact edit tools are **additive**: they update or add, but cannot remove or clear existing data.
- Email drafts land in Levitate as drafts. Nothing sends until a human approves it.
- Tool names are mostly undocumented. Only `contacts_search` and `companies_search` are named in
  Levitate's docs; the rest are described by capability. Call `tools/list` after authenticating to get
  the real inventory and input schemas.
- A credential scoped to the MCP server will **not** authenticate against the REST Public API. They are
  separate surfaces with separate credentials.
