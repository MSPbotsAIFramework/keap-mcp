# keap-mcp

Keap (formerly Infusionsoft) MCP Service — a stateless HTTP MCP server wrapping the [Keap CRM REST API v1](https://developer.keap.com/docs/rest/), covering Contacts, Tags, Opportunities, Notes, Tasks, Campaigns, and Users.

**Tech stack:** Python 3.12 + uv + FastMCP (Starlette/Uvicorn)

It follows the MSPbots **Vendor MCP Service SOP**: stateless, no stored credentials, per-request header authentication only (no environment-variable fallback).

<div align="center">

## MSPbots Connector

**The MCP Gateway built for MSPs: visible, controlled, and your data stays where it belongs.**

[**Get started at connect.mspbots.ai →**](https://connect.mspbots.ai)

</div>

---

MSPbots Connector gives your AI clients one MCP URL to reach your entire MSP stack (RMM, PSA, documentation, security, finance), and gives you control over who can call which tool and a record of everything that was called.

**Hosted service: [connect.mspbots.ai](https://connect.mspbots.ai)** · Free to use.

### Why Connector

![MSPbots Connector](https://stgoss.mspbots.ai/files/mcp/mspbots-mcp.png)

| | |
|---|---|
| **Visible** | Every tool call is logged: who, which vendor, when. Usage dashboard, CSV export, SIEM streaming. |
| **Controlled** | Role-based, per-tool allowlist. Deny by default. |
| **Protected** | Credentials encrypted per record. AI clients never see your vendor passwords or tokens. |
| **MSP-native** | Built for the MSP stack only: RMM, PSA, documentation, security, finance. |

### Quick start

1. **Sign up** at [connect.mspbots.ai](https://connect.mspbots.ai).
2. **Connect your vendors.** Pick from 150+ connectors and authorize each one on the vendor's own page, or bring your own API keys. No extra vendor accounts needed.
3. **Set access.** Create roles and decide which tools each role may call. Nothing is allowed until you allow it.
4. **Add your MCP URL to your AI client.** Copy the URL from your Connector dashboard into Claude, ChatGPT, Cursor, VS Code, Gemini, Codex, or any MCP-capable client.
5. **Ask your AI to do real work**, and watch every call appear in the audit log.

### Features

**Unified gateway**
- One MCP URL for all your AI clients
- 150+ connectors across the MSP stack
- Custom MCP servers supported, free
- Authorization happens on each vendor's own page; the AI never sees passwords or tokens
- Bring your own credentials: use the vendor API keys you already have
- Multiple accounts per app; connect once, reuse across AI clients
- Tool search: finds the right action for the task
- One sign-in; vendor credentials are injected behind the scenes and OAuth tokens refresh automatically

**Visibility**
- Audit log for every tool call (who, which vendor, when)
- Usage dashboard: call volume, top vendors, error rate, per-user activity
- Export audit logs to CSV, or stream to Loki, Graylog, or LogScale (SIEM)
- Optional recording of tool arguments and result summaries (off by default, enabled per organization)

**Control**
- Per-role, per-tool allowlist, deny by default
- owner / admin / member roles
- SCIM provisioning with Entra ID, Okta, JumpCloud, and Google Workspace

**Security**
- Credentials encrypted one record at a time (AES-256-GCM, per-record key)
- AI clients authenticate with OAuth 2.1 + PKCE
- Vendor responses are scanned for prompt-injection patterns and returned as structured data. This is one layer of defense, not a guarantee against every attack.
- Static egress IP for customer firewall and API allowlisting

### Pricing

**Free.**

Links: [connect.mspbots.ai](https://connect.mspbots.ai) · [mspbots.ai](https://mspbots.ai)

---

## Scope

Out of Keap's full REST API v1, this service implements **15 tools** aligned to the resource categories MSPbots has historically synced (Contact, Tag Contact, Tags, Opportunities, Notes, Task, Campaigns, User).

## Authentication

Keap's REST API v1 is `oauth2` on paper (`authorization_code` grant is the only flow in its official OpenAPI spec's `securitySchemes`), but Keap also offers a documented, self-service alternative that needs no redirect: **Personal Access Tokens (PAT)** and **Service Account Keys (SAK)** — see [developer.infusionsoft.com/pat-and-sak](https://developer.infusionsoft.com/pat-and-sak/). Either is generated inside the Keap app itself (Settings → API Settings, no developer-app registration or client secret needed) and used as a plain bearer token:

```
Authorization: Bearer <PAT or SAK>
```

This service uses that path — a single static token per tenant, no login/token-exchange step, no redirect. A Service Account Key (admin-level) is recommended over a Personal Access Token (single-user-scoped) for server-to-server use.

Every request to `/mcp` must include the following HTTP header. The token is read **only** from this header on every request (via a request-scoped `contextvar`) — it is never cached, never falls back to an environment variable, and is discarded as soon as the request completes, so concurrent requests from different tenants can never see each other's credentials.

## Quick Start

### Docker (recommended)

```bash
docker compose up --build
```

The server starts on `http://localhost:8080`.

### Local (uv)

```bash
uv sync
python -m keap_mcp
```

## Health Check

```bash
curl http://localhost:8080/health
# {"status": "ok"}
```

No credentials are required for the health endpoint.

## HEADER Authorization Parameters

| Header | Type | Required | Default | Allowed values | Description | Example |
|---|---|---|---|---|---|---|
| `X-Keap-Access-Token` | string | Yes | None | None | Keap Personal Access Token or Service Account Key (generated in the Keap admin under Settings → API Settings); this service forwards it as-is as `Authorization: Bearer` | `KeapAK-a1b2c3d4e5f6g7h8i9j0` |

Missing the header returns `401 Unauthorized` with the required header name listed in the response body.

## Environment Variables

| Variable | Default | Description |
|---|---|---|
| `MCP_HTTP_PORT` | `8080` | Listening port |
| `MCP_HTTP_HOST` | `0.0.0.0` | Listening host |

No credential fields exist in configuration — the access token is never read from an environment variable in this service; it comes exclusively from the `X-Keap-Access-Token` request header (see above).

## MCP Endpoint

```
POST http://localhost:8080/mcp
```

Connect your MCP client with:
- Transport: `http` (Streamable HTTP)
- Headers: `X-Keap-Access-Token: <token>` (required)

## Available Tools (15)

`limit` parameters below default to 20 and are clamped server-side to a hard cap of 200 (Keap's own API accepts up to 1000 per page, but this service applies the stricter MSPbots token-economy ceiling).

| Tool | Description | API |
|---|---|---|
| `keap_list_contacts` | List contacts | `GET /contacts` |
| `keap_get_contact` | Retrieve a contact by ID | `GET /contacts/{id}` |
| `keap_create_contact` | Create a contact | `POST /contacts` |
| `keap_list_contact_tags` | List tags applied to a contact | `GET /contacts/{contactId}/tags` |
| `keap_apply_tags_to_contact` | Apply tags to a contact | `POST /contacts/{contactId}/tags` |
| `keap_list_tags` | List all tags defined in the app | `GET /tags` |
| `keap_list_opportunities` | List sales opportunities (deals) | `GET /opportunities` |
| `keap_get_opportunity` | Retrieve an opportunity by ID | `GET /opportunities/{opportunityId}` |
| `keap_create_opportunity` | Create a sales opportunity | `POST /opportunities` |
| `keap_list_notes` | List notes | `GET /notes` |
| `keap_create_note` | Create a note on a contact | `POST /notes` |
| `keap_list_tasks` | List tasks | `GET /tasks` |
| `keap_create_task` | Create a task | `POST /tasks` |
| `keap_list_campaigns` | List marketing/automation campaigns | `GET /campaigns` |
| `keap_list_users` | List users of the Keap app | `GET /users` |

Tool descriptions in the MCP protocol itself are intentionally short (business-facing "what it does" summary only); full per-parameter documentation lives in each parameter's JSON-schema `description` (visible via `tools/list`), not in this table.

## Test Example

```bash
curl -X POST http://localhost:8080/mcp \
  -H "Content-Type: application/json" \
  -H "X-Keap-Access-Token: <token>" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": { "name": "keap_list_contacts", "arguments": { "limit": 5 } }
  }'
```

## Known Gaps

- **⚠️ Not yet tested against a live Keap account.** All 15 tools have been checked structurally only (MCP handshake, tools-list, schema validity, `/health`, gateway 401 credential-gating) — no real PAT/SAK was available at build time. A free-trial Keap account application has been submitted separately; once approved, generate a Service Account Key and test end-to-end.
- Endpoint paths/params were verified directly against Keap's official OpenAPI v1 spec (`https://crm.infusionsoft.com/app/v3/api-docs/V1`, linked from the "Download OpenAPI specification" button on the docs site) — not guessed.
- `PUT /contacts` (upsert) documents a `duplicate_option` field in prose but it does not appear in the actual request schema — a documentation/schema mismatch on Keap's side, not implemented here since it's unconfirmed. Not built as a tool in this 15-tool scope anyway (only `POST`/`GET` contact operations were included).
- Scope is limited to the 15 operations above (Contacts/Tags/Opportunities/Notes/Tasks/Campaigns/Users read+core-write), not Keap's full REST API v1 surface (which also includes Companies, Products, Orders/Subscriptions, Affiliates, Files, Email sending, custom fields management, and webhooks).
- Keap's list endpoints accept `limit` up to 1000 server-side; this service clamps to 200 (the SOP's hard cap) to bound response size and token cost — a value between 200 and 1000 will silently be reduced to 200 rather than rejected.

## API Reference

- [Keap REST API v1 Documentation](https://developer.keap.com/docs/rest/)
- [Personal Access Tokens & Service Account Keys](https://developer.infusionsoft.com/pat-and-sak/)
