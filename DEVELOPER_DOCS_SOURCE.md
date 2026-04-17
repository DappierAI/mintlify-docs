# Dappier Sales Agent MCP — Developer Documentation Source

> Source-of-truth reference for developers integrating with the Dappier Sales Agent MCP server. Everything below is derived from the live server implementation (`src/index.ts` and `src/tools/*.ts`) and is intended to be the raw material for authoring end-user docs.

---

## 0. Audience and scope

This document is for **developers who want to USE the Dappier Sales Agent MCP server** from their own AI client, LLM agent, or custom application. It is **not** a contributor / local-dev guide — there are no sections on cloning the repo, building it, or deploying it.

The reader may be completely new to MCP. Sections 1 and 2 cover the "what" and "why"; section 3 onward is integration detail.

---

## 1. Primer: what is MCP (for developers who've never seen it)

**Model Context Protocol (MCP)** is an open standard that lets an AI application (an "MCP client" — Claude Desktop, Cursor, Claude.ai, a custom LangChain/LangGraph app, etc.) call tools hosted by a remote service (an "MCP server").

Mental model:

```
┌────────────────────────┐       JSON-RPC / SSE       ┌────────────────────────────┐
│  Your AI client / LLM  │  ────────────────────────► │  Dappier Sales Agent MCP   │
│  (Claude, Cursor,      │                            │  (this server, hosted)     │
│  your own app, etc.)   │  ◄────────────────────────  │                            │
└────────────────────────┘        tool results         └────────────────────────────┘
                                                               │
                                                               ▼
                                                        Dappier REST API
                                                      (campaigns, prompts,
                                                       brand agents, etc.)
```

- The **server** advertises a list of tools (name + JSON schema + description).
- The **client** fetches that list and lets the LLM call any tool by name with arguments that match the schema.
- The server validates, executes, and returns a result. The LLM sees the result as a plain text/JSON block and uses it to continue the conversation.

The Dappier Sales Agent MCP server exposes 9 tools that let an AI agent **discover Dappier advertising inventory, create sponsored campaigns, manage creatives, and pull delivery metrics**, all without the developer having to write HTTP wrappers around the Dappier REST API.

---

## 2. What the Dappier Sales Agent MCP server actually is

A remote MCP server implementing the **AdCP (Advertising Context Protocol)** sales-agent surface for Dappier.

Two AdCP protocols are supported:

| Protocol | What it covers |
|---|---|
| `media_buy` | Discover products, create/update/cancel campaigns ("media buys"), pull delivery data. |
| `creative` | List creative formats, create/update Dappier Brand Agent creatives, list existing creatives. |

What you as a developer get by connecting to it:

1. An **AI-callable surface** for Dappier Sponsored Conversations — the branded prompt suggestions that appear inside publisher AI chat widgets across the Dappier network.
2. Campaign lifecycle in a single tool call: create a paused campaign with advertiser details, CTA, targeting, and optional sponsored prompts.
3. Creative lifecycle: attach a configurable **Dappier Brand Agent** (conversational AI creative) to a campaign via `build_creative`.
4. Delivery reporting over a date range.

What the server is **not**:

- It does not quote prices or sell impressions directly — pricing is handled offline by Dappier sales (`sales@dappier.com`).
- It does not activate campaigns. Every campaign is born `paused`; a Dappier reviewer must approve and a sales rep must complete Google Ad Manager line-item setup before it serves.
- It does not honor standard AdCP targeting (geo, device, language, audience). Dappier targeting is `dappier_network` / `my_network` / `individual_agents`.

---

## 3. Base URLs and endpoints

The server is hosted by Dappier. You connect to it using standard MCP transports.

| Transport | Path | When to use |
|---|---|---|
| Streamable HTTP (newer, recommended) | `POST /mcp` | Most modern MCP clients (Claude.ai, Cursor, custom apps). |
| Server-Sent Events (legacy) | `GET /sse` | Older MCP clients (some Claude Desktop configs, `mcp-remote` proxy). |

Health / discovery endpoints (open, no auth):

| Path | Returns |
|---|---|
| `GET /` | An HTML landing page confirming the worker is up. |
| `GET /.well-known/mcp.json` | The AdCP server card (name, version, protocols, tool names). |
| `GET /.well-known/server.json` | Same server card (alias). |

> Production host / URL and any staging host are managed by Dappier operations. Your docs should reference whatever deployed URL Dappier publishes (e.g. `https://sales-agent.dappier.com/mcp`). Do not hard-code internal URLs.

---

## 4. Authentication

Every MCP request (except the `/` and `/.well-known/*` discovery endpoints) requires a **Dappier API key**.

You can supply it in two ways:

### 4.1 Query parameter

```
https://<host>/mcp?apiKey=YOUR_DAPPIER_API_KEY
https://<host>/sse?apiKey=YOUR_DAPPIER_API_KEY
```

### 4.2 HTTP header

```
dappier-api-key: YOUR_DAPPIER_API_KEY
```

If the key is missing you'll get an HTTP `401` with a plain-text body that looks like:

```
Authentication required

A Dappier API key is required to access this endpoint.

You can provide it in one of two ways:
  1. Query parameter:  ?apiKey=YOUR_DAPPIER_API_KEY
  2. Request header:   dappier-api-key: YOUR_DAPPIER_API_KEY

Don't have a key yet? Create one at:
  https://platform.dappier.com/profile/api-keys
```

### 4.3 Getting an API key

Developers create Dappier API keys at **https://platform.dappier.com/profile/api-keys**. The key is used:

- As a Bearer token on every outbound call the server makes to `api.dappier.com`.
- To authorize the MCP session itself at the edge.

Keep the key server-side / in a secret manager. Do not expose it in browser code.

---

## 5. Connecting from common MCP clients

### 5.1 Claude Desktop / Claude.ai (via `mcp-remote`)

Claude Desktop speaks MCP over stdio. To reach a remote HTTPS MCP server you proxy through `mcp-remote`.

Edit your Claude Desktop config (Settings → Developer → Edit Config):

```jsonc
{
  "mcpServers": {
    "dappier-sales-agent": {
      "command": "npx",
      "args": [
        "mcp-remote",
        "https://<dappier-mcp-host>/sse?apiKey=YOUR_DAPPIER_API_KEY"
      ]
    }
  }
}
```

Then restart Claude Desktop — the 9 Dappier tools will appear in the tool picker.

### 5.2 Claude.ai (Connectors / Remote MCP)

On claude.ai, add a custom connector / remote MCP server pointing at `https://<dappier-mcp-host>/mcp`. Supply the API key via header (`dappier-api-key`) where the UI allows custom headers, or via `?apiKey=...` in the URL otherwise.

### 5.3 Cursor

Add a remote MCP server in Cursor's MCP settings pointing at `https://<dappier-mcp-host>/mcp?apiKey=YOUR_KEY`. Cursor supports the streamable HTTP transport directly.

### 5.4 Cloudflare AI Playground

Go to https://playground.ai.cloudflare.com/ and enter `https://<dappier-mcp-host>/sse?apiKey=YOUR_KEY` as the server URL.

### 5.5 Custom Node / Python client

Any client built on `@modelcontextprotocol/sdk` or the Python `mcp` SDK can connect over streamable HTTP. Minimal Node example:

```ts
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StreamableHTTPClientTransport } from "@modelcontextprotocol/sdk/client/streamableHttp.js";

const transport = new StreamableHTTPClientTransport(
  new URL("https://<dappier-mcp-host>/mcp"),
  { requestInit: { headers: { "dappier-api-key": process.env.DAPPIER_API_KEY! } } }
);

const client = new Client({ name: "my-app", version: "1.0.0" }, { capabilities: {} });
await client.connect(transport);

const tools = await client.listTools();
console.log(tools);

const products = await client.callTool({
  name: "get_products",
  arguments: { buying_mode: "brief", brief: "Launch a new coffee brand in Q3" }
});
console.log(products);
```

### 5.6 Anthropic SDK (native MCP connector, no client SDK)

The Anthropic Messages API can call a remote MCP server directly as a tool-use source:

```python
client.messages.create(
    model="claude-opus-4-7",
    max_tokens=1024,
    mcp_servers=[{
        "type": "url",
        "url": "https://<dappier-mcp-host>/mcp",
        "name": "dappier-sales-agent",
        "authorization_token": "YOUR_DAPPIER_API_KEY"  # sent as dappier-api-key
    }],
    messages=[{"role": "user", "content": "List my active Dappier campaigns."}]
)
```

---

## 6. Server card / discovery payload

`GET /.well-known/mcp.json` (and `/.well-known/server.json`) returns:

```json
{
  "name": "com.dappier/sales-agent",
  "version": "1.0.0",
  "title": "Dappier Sales Agent",
  "description": "AdCP sales agent for Dappier's Sponsored Conversations network — discover inventory, create and manage branded-prompt campaigns across Dappier's publisher AI chat widgets.",
  "tools": [
    { "name": "get_adcp_capabilities" },
    { "name": "list_creative_formats" },
    { "name": "build_creative" },
    { "name": "list_creatives" },
    { "name": "get_products" },
    { "name": "create_media_buy" },
    { "name": "update_media_buy" },
    { "name": "get_media_buys" },
    { "name": "get_media_buy_delivery" }
  ],
  "_meta": {
    "adcontextprotocol.org": { "protocols_supported": ["media_buy", "creative"] }
  }
}
```

---

## 7. Conventions used across every tool

### 7.1 ID prefixes

| Prefix | Meaning | Example |
|---|---|---|
| `cp_` | Dappier campaign / media buy external id | `cp_01HW9...` |
| `am_` | Dappier Brand Agent / creative id | `am_01HW9...` |
| `pm_` | Sponsored prompt id | `pm_01HW9...` |

Schemas reject values that don't match the expected prefix.

### 7.2 Response envelope

Every tool returns an MCP `CallToolResult` with two fields:

- `content[0]` — a `text` block containing pretty-printed JSON (for LLMs reading the text).
- `structuredContent` — the same payload as a machine-readable object (for programmatic clients).

Both contain the same data — use whichever matches your client.

### 7.3 Error shape

Errors follow AdCP conventions:

```json
{
  "errors": [
    { "code": "VALIDATION_ERROR", "message": "...", "field": "cta_link" }
  ]
}
```

Or, for async-style tools, a `status: "failed"` wrapper:

```json
{ "status": "failed", "errors": [{ "code": "INVALID_REQUEST", "message": "..." }] }
```

Consolidated error-code reference:

| Code | Meaning |
|---|---|
| `INVALID_REQUEST` | Generic 400-level rejection on create. |
| `VALIDATION_ERROR` | Update with no fields, or 400-level rejection on update. |
| `INVALID_DATE_RANGE` | Missing, malformed, or unsorted `start_date` / `end_date`. |
| `POLICY_VIOLATION` | 401/403 (bad key, insufficient scope). |
| `AUTH_REQUIRED` | 401 on read tools. |
| `MEDIA_BUY_NOT_FOUND` | Unknown `cp_xxxx`. |
| `CREATIVE_NOT_FOUND` | Unknown `am_xxxx`. |
| `INVALID_STATE` | Campaign in a state that forbids the requested action. |
| `CREATIVE_ID_EXISTS` | 409 conflict (duplicate creative attached). |
| `CONTEXT_REQUIRED` | Default-scope list returned zero; supply `media_buy_ids` or `status_filter`. |
| `VERSION_UNSUPPORTED` | `adcp_major_version` not supported. |
| `INTERNAL_ERROR` | Fallback. |

### 7.4 Context passthrough

Most tools accept `context: Record<string, unknown>`. Whatever you send is echoed back unchanged on the response — useful for correlating tool calls with your own session state.

### 7.5 Typical campaign lifecycle

```
1. get_adcp_capabilities                 — discover what this server supports
2. get_products                          — confirm Sponsored Conversations is the product
3. create_media_buy                      — submit the campaign (returns cp_xxxx, starts paused)
4. list_creative_formats                 — find the dappier_brand_agent format id
5. build_creative (create mode)          — attach a Brand Agent to the cp_xxxx
6. [offline] Dappier sales sets up GAM, reviewer approves
7. update_media_buy { paused: false }    — resume (may be rejected until sales approves)
8. get_media_buys                        — check operational state
9. get_media_buy_delivery                — pull impressions/clicks over a date range
```

---

## 8. Tools reference

Every tool is documented with: purpose, when to call it, every input field (type, required, semantics), the full shape of a success response, and how the tool surfaces errors.

---

### 8.1 `get_adcp_capabilities`

**Title:** Get AdCP Capabilities

**Purpose:** The first call a buyer should make. Tells you which AdCP protocols Dappier implements, which features inside those protocols are honored vs. ignored, the auth model, and which creative capabilities are available.

**Network behavior:** In-memory lookup. No outbound call. Returns instantly.

**Inputs (all optional):**

| Field | Type | Description |
|---|---|---|
| `adcp_major_version` | integer | AdCP major version the buyer's payloads conform to. If provided and unsupported, returns `VERSION_UNSUPPORTED`. Currently supported: `1`. |
| `protocols` | array of `"media_buy" \| "signals" \| "governance" \| "sponsored_intelligence" \| "creative" \| "compliance_testing"` | Filter which protocols you want capability info for. If omitted, you get everything Dappier supports. |

**Success response:**

```json
{
  "adcp": { "major_versions": [1] },
  "supported_protocols": ["media_buy", "creative"],
  "account": {
    "supported_billing": ["operator"],
    "require_operator_auth": false,
    "required_for_products": false,
    "account_financials": false,
    "sandbox": false
  },
  "media_buy": {
    "features": {
      "inline_creative_management": false,
      "property_list_filtering": false,
      "content_standards": false,
      "audience_targeting": false
    },
    "execution": {
      "creative_specs": { "vast_versions": [], "mraid_versions": [], "vpaid": false, "simid": false },
      "targeting": { /* every geo/device/audience field is false */ }
    },
    "portfolio": {
      "publisher_domains": ["dappier.com"],
      "primary_channels": ["native"],
      "primary_countries": ["US"],
      "description": "Dappier operates a network of publishers running AI chat experiences...",
      "advertising_policies": "All campaigns require manual review..."
    }
  },
  "creative": {
    "has_creative_library": true,
    "supports_generation": false,
    "supports_transformation": false,
    "supports_compliance": false
  },
  "last_updated": "2026-04-10T00:00:00Z"
}
```

**Error response:**

```json
{ "errors": [{ "code": "VERSION_UNSUPPORTED", "message": "AdCP major version 2 is not supported..." }] }
```

**Key points to tell readers:**

- Targeting dimensions declared `false` will return a validation error if sent — they are not silently dropped.
- `media_buy` and `creative` are the only protocols you'll see.

---

### 8.2 `list_creative_formats`

**Title:** List Dappier creative formats

**Purpose:** Discover the creative formats supported by the Dappier sales agent. Call this before `build_creative`.

**Network behavior:** In-memory. Instant.

**Inputs (all optional and all ignored in v1 — accepted for AdCP forward-compat):**

| Field | Type | Notes |
|---|---|---|
| `format_ids` | array of `{ agent_url: string, id: string }` | Ignored. |
| `asset_types` | string[] | Ignored. |
| `max_width`, `max_height`, `min_width`, `min_height` | integer | Ignored. |
| `is_responsive` | boolean | Ignored. |
| `name_search` | string | Ignored. |
| `wcag_level` | `"A" \| "AA" \| "AAA"` | Ignored. |
| `pagination.max_results`, `pagination.cursor` | | Ignored. |

**Success response:** A single format descriptor for `dappier_brand_agent` (the only creative format Dappier exposes). The `format_id.agent_url` returned here is the exact value you must pass into `build_creative.target_format_id`.

**Why call it at all:** to pick up the exact `agent_url` string — don't hard-code it.

---

### 8.3 `build_creative`

**Title:** Build Dappier Brand Agent creative

**Purpose:** Create a new Dappier Brand Agent (conversational AI creative) and attach it to a campaign, or update an existing one.

A Brand Agent is a configurable chat experience branded for an advertiser: name, description, optional persona, optional single knowledge source (RSS feed or webpage), and optional widget overrides (logo, colors, welcome copy, theme).

**Two modes:**

| Mode | How to trigger | Effect |
|---|---|---|
| **CREATE** | Omit `creative_id` | Creates a new Brand Agent, attaches it to `media_buy_id`. Requires `creative_manifest.assets.name` and `...description`. The campaign must not already have a creative attached. |
| **UPDATE** | Supply `creative_id` (`am_xxxx`) | PATCH semantics — only the fields you send change. `source_url` and `source_type` must be supplied together or not at all. |

**Inputs:**

| Field | Type | Required | Notes |
|---|---|---|---|
| `target_format_id` | `{ agent_url: URL, id: "dappier_brand_agent" }` | yes | Must be `dappier_brand_agent`. Get `agent_url` from `list_creative_formats`. |
| `media_buy_id` | `cp_xxxx` string | yes | Campaign to attach the Brand Agent to. |
| `creative_id` | `am_xxxx` string | no | Present → UPDATE. Omit → CREATE. |
| `creative_manifest.format_id` | `{ agent_url, id: "dappier_brand_agent" }` | yes | Must equal `target_format_id`. |
| `creative_manifest.assets.name` | string ≤120 | required on CREATE | Shown as "Ask {name}". |
| `creative_manifest.assets.description` | string ≤500 | required on CREATE | Brand description used inside the conversation. |
| `creative_manifest.assets.persona` | string | no | Free-form tone instruction. |
| `creative_manifest.assets.source_url` | URL | no | RSS feed or webpage. Must pair with `source_type`. |
| `creative_manifest.assets.source_type` | `"rss" \| "webpage"` | no | Must pair with `source_url`. |
| `creative_manifest.assets.logo_url` | URL | no | Widget logo override. |
| `creative_manifest.assets.primary_color` | string | no | Hex color, e.g. `#8353E2`. |
| `creative_manifest.assets.welcome_title` | string | no | Title above the Ask-AI widget. |
| `creative_manifest.assets.welcome_description` | string | no | Subtitle above the widget. |
| `creative_manifest.assets.placeholder_text` | string | no | Placeholder text inside the input box. |
| `creative_manifest.assets.theme_mode` | `"light" \| "dark"` | no | Widget theme. |
| `context` | `Record<string, unknown>` | no | Echoed back. |

**Success response** contains at minimum:

```json
{
  "creative_id": "am_...",
  "media_buy_id": "cp_...",
  "widget_id": "...",
  "target_format_id": { "agent_url": "...", "id": "dappier_brand_agent" }
}
```

**Example — CREATE:**

```json
{
  "target_format_id": {
    "agent_url": "https://sales-agent.dappier.com/.well-known/adcp/sales",
    "id": "dappier_brand_agent"
  },
  "media_buy_id": "cp_01HW9ZAB...",
  "creative_manifest": {
    "format_id": {
      "agent_url": "https://sales-agent.dappier.com/.well-known/adcp/sales",
      "id": "dappier_brand_agent"
    },
    "assets": {
      "name": "Acme Pet Food",
      "description": "Premium, science-backed nutrition for dogs and cats.",
      "persona": "Friendly, expert, concise.",
      "source_url": "https://acmepetfood.com/blog/feed.xml",
      "source_type": "rss",
      "primary_color": "#8353E2",
      "welcome_title": "Ask Acme Pet Food",
      "theme_mode": "light"
    }
  }
}
```

**Example — UPDATE (change logo only):**

```json
{
  "target_format_id": { "agent_url": "...", "id": "dappier_brand_agent" },
  "media_buy_id": "cp_01HW9ZAB...",
  "creative_id": "am_01HW9CD...",
  "creative_manifest": {
    "format_id": { "agent_url": "...", "id": "dappier_brand_agent" },
    "assets": { "logo_url": "https://cdn.acme.com/logo-new.png" }
  }
}
```

---

### 8.4 `list_creatives`

**Title:** List Dappier Brand Agent creatives

**Purpose:** Browse the tenant's Brand Agents, or look up specific ones.

**Three filter paths (mutually exclusive on the primary id filters):**

| Path | How to trigger | Behavior |
|---|---|---|
| **By creative ids** | `filters.creative_ids` | Look up specific Brand Agents. Missing ids → `CREATIVE_NOT_FOUND` entries in `errors[]`. Pagination ignored. |
| **By media buy ids** | `filters.media_buy_ids` | List Brand Agents attached to specific campaigns. Missing campaigns → `MEDIA_BUY_NOT_FOUND`. Pagination ignored. |
| **Default** | omit both | Paginated list of every tenant Brand Agent. |

(If both `creative_ids` and `media_buy_ids` are supplied, `creative_ids` wins.)

`filters.name_contains` (case-insensitive) and `filters.statuses` (`approved` / `pending_review` / `archived`) narrow any of the three paths.

**Inputs:**

| Field | Type | Notes |
|---|---|---|
| `filters.creative_ids` | `am_xxxx[]` | Lookup by id. |
| `filters.media_buy_ids` | `cp_xxxx[]` | Lookup by campaign. |
| `filters.name_contains` | string | Case-insensitive substring. |
| `filters.statuses` | `("approved" \| "pending_review" \| "archived")[]` | |
| `pagination.max_results` | int 1-100 (default 50) | Honored only on default path. |
| `pagination.cursor` | string | Opaque — round-trip verbatim. |
| `context` | object | Echoed back. |

---

### 8.5 `get_products`

**Title:** Get Products

**Purpose:** Discover Dappier's advertising inventory. Always returns the single **Sponsored Conversations** product (or an empty list if your filters exclude it).

**Network behavior:** In-memory. No API call.

**Inputs:**

| Field | Type | Required | Notes |
|---|---|---|---|
| `buying_mode` | `"brief" \| "wholesale" \| "refine"` | yes | How you want results selected. |
| `brief` | string | required iff `buying_mode === "brief"` | Natural-language campaign description. Must be omitted otherwise. |
| `brand.domain` | string | no | E.g. `"acmepetfood.com"`. |
| `brand.brand_id` | string | no | Optional stable id. |
| `filters.delivery_type` | `"guaranteed" \| "non_guaranteed"` | no | Dappier only offers `non_guaranteed`; anything else returns `[]`. |
| `filters.channels` | string[] | no | Must include `"native"` (or be omitted) for the product to match. |
| `filters.format_ids` | `{agent_url, id}[]` | no | Must include Dappier's format to match. |

**Success response:**

```json
{
  "products": [{
    "product_id": "sponsored_conversations",
    "name": "Sponsored Conversations",
    "description": "A branded prompt suggestion...",
    "publisher_properties": [{ "publisher_domain": "dappier.com", "property_tags": ["dappier_network"] }],
    "format_ids": [{
      "agent_url": "https://sales-agent.dappier.com/.well-known/adcp/sales",
      "id": "sponsored_prompt_standard"
    }],
    "delivery_type": "non_guaranteed",
    "pricing_options": [{
      "pricing_option_id": "contact_sales",
      "pricing_model": "flat_rate",
      "currency": "USD",
      "min_spend_per_package": 0
    }],
    "brief_relevance": "Strong fit — ..."  // present only if brief was supplied
  }]
}
```

---

### 8.6 `create_media_buy`

**Title:** Create Media Buy

**Purpose:** Create a sponsored campaign on the Dappier network. Campaign is born `paused`. It will not serve until (1) Dappier sales sets up the GAM line item offline and (2) a Dappier reviewer approves it.

You can optionally pass `sponsored_prompts` to create the clickable prompt content atomically with the campaign.

**Inputs:**

| Field | Type | Required | Notes |
|---|---|---|---|
| `brand.name` | string | yes | → Dappier `details.advertiser_name`. |
| `brand.domain` | string | no | |
| `buyer_ref` | string | no | Your correlation id, echoed in the response. |
| `packages` | `Package[]` | defaulted | Defaults to a single `sponsored_conversations` package. Only set it to pass `targeting_overlay.product_ids` for individual-agent targeting. |
| `packages[].product_id` | literal `"sponsored_conversations"` | defaulted | Only allowed value. |
| `packages[].pricing_option_id` | literal `"contact_sales"` | defaulted | Only allowed value. |
| `packages[].format_ids[]` | `{ agent_url, id: "sponsored_prompt_standard" }` | defaulted | |
| `packages[].targeting_overlay.product_ids` | string[] | no | `am_xxxx` agent ids. Only honored when `targeting_type === "individual_agents"`. |
| `start_time` | ISO 8601 datetime | no | |
| `end_time` | ISO 8601 datetime | no | |
| `campaign_name` | string | **yes** | → Dappier `details.name`. |
| `cta_link` | URL | **yes** | Where clicks land. **Must be supplied by the user — never infer from brand name / domain.** |
| `cta_button_text` | string | **yes** | E.g. `"Shop now"`, `"Learn more"`. **Must be supplied by the user — never invent.** |
| `targeting_type` | `"dappier_network" \| "my_network" \| "individual_agents"` | defaulted | Default: `dappier_network`. |
| `sponsored_prompts[]` | `{ prompt_text (≤200), thumbnail?, is_active (default true) }` | no | Any caller-supplied `prompt_id` is silently dropped on create — the server assigns new `pm_xxxx` ids. |
| `context` | object | no | Echoed back. |

**Success response:**

```json
{
  "status": "submitted",
  "media_buy_id": "cp_01HW9ZAB...",
  "task_id": "cp_01HW9ZAB...",
  "buyer_ref": "...",
  "message": "Campaign created in paused state. Dappier sales will contact you to complete GAM setup and activate the campaign.",
  "confirmed_at": "2026-04-18T14:00:00Z",
  "packages": [{
    "package_id": "cp_01HW9ZAB...",
    "product_id": "sponsored_conversations",
    "status": "pending_activation"
  }],
  "context": { /* your context, echoed */ }
}
```

**Error response:**

```json
{ "status": "failed", "errors": [{ "code": "INVALID_REQUEST", "message": "..." }] }
```

Error code mapping: 400 → `INVALID_REQUEST`, 401/403 → `POLICY_VIOLATION`, 409 → `CREATIVE_ID_EXISTS`, else → `INTERNAL_ERROR`.

**Example:**

```json
{
  "brand": { "name": "Acme Pet Food", "domain": "acmepetfood.com" },
  "campaign_name": "Q3 Puppy Launch",
  "cta_link": "https://acmepetfood.com/puppy",
  "cta_button_text": "Shop the launch",
  "targeting_type": "dappier_network",
  "start_time": "2026-07-01T00:00:00Z",
  "end_time":   "2026-09-30T23:59:59Z",
  "sponsored_prompts": [
    { "prompt_text": "Best food for new puppies?", "is_active": true },
    { "prompt_text": "Is grain-free food worth it?" }
  ]
}
```

---

### 8.7 `update_media_buy`

**Title:** Update Media Buy

**Purpose:** Modify an existing Dappier campaign. PATCH semantics — omitted fields are preserved. Atomic — either all changes apply or none do.

The handler routes to one of three backend operations based on what you send:

| Request shape | Backend action |
|---|---|
| `canceled: true` (any other field optional) | Soft-delete the campaign. Irreversible. Overrides other changes. |
| Only `paused` set, no other update fields | Status-only flip (pause/resume). |
| Any update field (optionally plus `paused`) | Full update (all provided fields). |

**Key inputs (all optional except `media_buy_id`):**

| Field | Type | Notes |
|---|---|---|
| `media_buy_id` | `cp_xxxx` | **required.** |
| `paused` | boolean | `true` = pause, `false` = resume. Resume may be rejected until Dappier sales approves — see below. |
| `canceled` | literal `true` | Soft-delete. Irreversible. |
| `cancellation_reason` | string | Echoed in the response message. |
| `start_time`, `end_time` | ISO 8601 | New flight dates. |
| `campaign_name`, `advertiser_name`, `brand_description`, `contextual_keywords` | string | Map to `details.*`. |
| `targeting_type` | `"dappier_network" \| "my_network" \| "individual_agents"` | |
| `product_ids` | `am_xxxx[]` | Replaces `details.targeting.agent_list`. |
| `cta_link` | URL | **Only include if the user explicitly asked to change it.** Otherwise omit (existing value is preserved). |
| `cta_button_text` | string | **Only include if the user explicitly asked.** |
| `custom_followup_prompts` | string[] (max 3) | Replaces existing list. |
| `sponsored_prompts[]` | `{ prompt_id?, prompt_text, thumbnail?, is_active }` | Full-replacement semantics — see below. |
| `context` | object | Echoed back. |

**Sponsored-prompts replacement semantics:** pass the **full desired set** after the update.

- Entries with `prompt_id` (`pm_xxxx`) → updated.
- Entries without `prompt_id` → newly created.
- Existing prompts not present in the array → soft-deleted.

**Success response:**

```json
{
  "status": "completed",
  "media_buy_id": "cp_01HW9...",
  "implementation_date": "2026-04-18T14:02:11Z",
  "affected_packages": [],
  "context": { /* echoed */ }
}
```

**Resume-pending special case:** if you send `paused: false` and the backend rejects with `INVALID_STATE`, `NOT_APPROVED`, or `PENDING_APPROVAL`, the tool returns `status: "submitted"` (not an error) with the message:

> Resume requires Dappier sales to complete GAM setup and a reviewer to approve. The campaign remains paused.

**Validation error — empty request:** if you send no `canceled`, no `paused`, and no update fields, you get:

```json
{ "status": "failed", "errors": [{ "code": "VALIDATION_ERROR", "message": "..." }] }
```

**Examples:**

_Pause a campaign:_
```json
{ "media_buy_id": "cp_01HW9...", "paused": true }
```

_Cancel with a reason:_
```json
{ "media_buy_id": "cp_01HW9...", "canceled": true, "cancellation_reason": "Client request" }
```

_Replace the prompt list:_
```json
{
  "media_buy_id": "cp_01HW9...",
  "sponsored_prompts": [
    { "prompt_id": "pm_01ABC...", "prompt_text": "Updated text", "is_active": true },
    { "prompt_text": "New prompt", "is_active": true }
  ]
}
```
_(Existing prompts not listed here are soft-deleted.)_

---

### 8.8 `get_media_buys`

**Title:** Get Media Buys

**Purpose:** Retrieve the current **operational state** of Dappier campaigns — configuration, sponsored-prompt approval status, and which campaigns are waiting on Dappier sales / GAM setup.

Use this for "what's the current state of my campaigns?". For performance over a date range, use `get_media_buy_delivery` instead.

**Inputs (all optional):**

| Field | Type | Notes |
|---|---|---|
| `media_buy_ids` | `cp_xxxx[]` | Fetch specific campaigns in parallel. When set, no implicit status filter is applied. |
| `status_filter` | one of / array of `"pending_creatives" \| "pending_start" \| "active" \| "paused" \| "completed" \| "rejected" \| "canceled"` | Defaults to `["active"]` when neither `media_buy_ids` nor `status_filter` is provided. |
| `include_snapshot` | boolean (default false) | Accepted for AdCP compliance. Always returns `snapshot_unavailable_reason: "SNAPSHOT_UNSUPPORTED"`. |
| `pagination.max_results` | int 1-100 (default 50) | |
| `pagination.cursor` | string | Opaque. Currently encoded as `page:<n>`. |
| `context` | object | Echoed back. |

**Success response:**

```json
{
  "media_buys": [
    {
      "media_buy_id": "cp_01HW9...",
      "status": "pending_creatives" | "active" | "paused" | "canceled",
      "currency": "USD",
      "total_budget": null,
      "creative_deadline": null,
      "confirmed_at": "2026-04-18T14:00:00Z",
      "revision": 1,
      "valid_actions": ["pause", "cancel", "update_budget", "update_dates", "update_media_buy"],
      "packages": [{
        "package_id": "cp_01HW9...",
        "paused": false,
        "canceled": false,
        "creative_approvals": [
          {
            "creative_id": "pm_01ABC...",
            "approval_status": "approved" | "pending_review" | "rejected",
            "rejection_reason": "..."
          }
        ],
        "format_ids_pending": [],
        "snapshot_unavailable_reason": "SNAPSHOT_UNSUPPORTED"
      }],
      "cancellation": null
    }
  ],
  "pagination": { "has_more": false },
  "errors": []
}
```

**Status derivation rules:**

- `is_deleted === true` → `canceled`
- else `status === "active"` → `active`
- else if creative_ids exist → `paused`
- else → `pending_creatives`

**Valid actions by status:**

| Status | Actions |
|---|---|
| `pending_creatives` | `cancel`, `update_media_buy` |
| `active` | `pause`, `cancel`, `update_budget`, `update_dates`, `update_media_buy` |
| `paused` | `resume`, `cancel`, `update_budget`, `update_dates`, `update_media_buy` |
| `canceled` | — |

**Empty-result nudge:** if you pass neither `media_buy_ids` nor `status_filter`, the default filter is `["active"]`, and if that produces no campaigns, the response includes:

```json
{ "errors": [{ "code": "CONTEXT_REQUIRED", "message": "No campaigns matched the default scope..." }] }
```

---

### 8.9 `get_media_buy_delivery`

**Title:** Get Media Buy Delivery

**Purpose:** Retrieve delivery metrics (impressions, clicks, etc.) for Dappier campaigns over a date range or campaign lifetime.

Use this for "how did my campaigns perform over a period?". For current state, use `get_media_buys`.

**Inputs:**

| Field | Type | Notes |
|---|---|---|
| `media_buy_ids` | `cp_xxxx[]` | When set, no implicit status filter is applied. |
| `status_filter` | one of / array of `"pending_creatives" \| "pending_start" \| "active" \| "paused" \| "completed"` | Defaults to `["active"]` (applied server-side) when neither `media_buy_ids` nor `status_filter` is provided. |
| `start_date` | `YYYY-MM-DD` | Inclusive. Must be paired with `end_date`. |
| `end_date` | `YYYY-MM-DD` | Exclusive. Must be paired with `start_date`. |
| `context` | object | Echoed back. |

**Date range rules:**

- Send both dates or neither.
- Both must match `YYYY-MM-DD`.
- `start_date < end_date` (strictly).
- Violations return `INVALID_DATE_RANGE` with `field` set to the offender — no backend call is made.

**Omitting both dates** returns lifetime-to-date data.

**Unsupported in v1:**

- Reporting dimensions (geo, device, audience, placement).
- Spend, ROAS, CPM, conversion value — pricing is handled offline in GAM, not tracked in Dappier.

**Success response:** whatever the backend returns for the AdCP delivery shape is passed through verbatim, plus your `context` echo. Expect at minimum per-campaign impressions/clicks and a `reporting_period` object.

**Error response:**

```json
{ "errors": [{ "code": "INVALID_DATE_RANGE", "message": "end_date must be after start_date", "field": "end_date" }] }
```

Error mapping: backend `code`/`error_code` passed through if provided, else 404 → `MEDIA_BUY_NOT_FOUND`, 401 → `AUTH_REQUIRED`, 400 → `INVALID_DATE_RANGE`, else → `INTERNAL_ERROR`.

---

## 9. End-to-end recipes

### 9.1 "Launch a campaign" (minimum viable flow)

```
1. get_adcp_capabilities
   → confirm media_buy + creative protocols are available.

2. get_products { buying_mode: "brief", brief: "launch new coffee brand in Q3" }
   → confirm Sponsored Conversations is the product.

3. create_media_buy {
     brand: { name, domain },
     campaign_name: "...",
     cta_link: "<user-supplied>",
     cta_button_text: "<user-supplied>",
     sponsored_prompts: [ { prompt_text }, ... ]  // optional
   }
   → returns cp_xxxx (paused).

4. list_creative_formats
   → pick up dappier_brand_agent's agent_url.

5. build_creative {
     target_format_id: { agent_url, id: "dappier_brand_agent" },
     media_buy_id: "cp_xxxx",
     creative_manifest: {
       format_id: { ... },
       assets: { name, description, persona?, source_url?, source_type?, ... }
     }
   }
   → returns am_xxxx creative + widget_id.

6. [Dappier sales + reviewer complete setup offline]

7. update_media_buy { media_buy_id: "cp_xxxx", paused: false }
   → campaign goes live (or comes back as status: "submitted" if still pending approval).
```

### 9.2 "What's happening with my campaigns right now?"

```
get_media_buys { status_filter: ["active", "paused", "pending_creatives"] }
```

Or for specific campaigns:

```
get_media_buys { media_buy_ids: ["cp_...","cp_..."] }
```

### 9.3 "How did last month perform?"

```
get_media_buy_delivery {
  media_buy_ids: ["cp_..."],
  start_date: "2026-03-01",
  end_date:   "2026-04-01"
}
```

### 9.4 "Edit the prompts on an existing campaign"

```
update_media_buy {
  media_buy_id: "cp_...",
  sponsored_prompts: [
    { prompt_id: "pm_EXISTING_ONE", prompt_text: "Updated text", is_active: true },
    { prompt_text: "Brand-new prompt" }
  ]
}
```
(Any pre-existing prompts not listed are soft-deleted.)

### 9.5 "Attach a different Brand Agent to a campaign"

Call `build_creative` in UPDATE mode with the existing `creative_id`. Only send the asset fields you want to change — the rest are preserved.

---

## 10. Guardrails the server enforces

These are quiet-but-strict rules callers often trip over.

| Rule | Where |
|---|---|
| Never infer `cta_link` or `cta_button_text` from the brand. Ask the user. The schema descriptions say so explicitly and the backend has no way to recover. | `create_media_buy`, `update_media_buy` |
| `creative_id` starts with `am_`, `media_buy_id` starts with `cp_`, prompt ids with `pm_`. | Everywhere. |
| `source_url` and `source_type` must be supplied together. | `build_creative` |
| `brief` is required iff `buying_mode === "brief"`; forbidden otherwise. | `get_products` |
| `start_date` and `end_date` must both be present or both absent; `start_date < end_date`. | `get_media_buy_delivery` |
| `update_media_buy` with no fields at all returns `VALIDATION_ERROR`. | `update_media_buy` |
| `custom_followup_prompts` max 3 items. | `update_media_buy` |
| Any targeting dimension declared `false` in capabilities will reject rather than silently drop. | `create_media_buy`, `update_media_buy` |

---

## 11. FAQ

**Q: Do I need to call `get_adcp_capabilities` on every request?**
No. Call it once per session to discover the surface; cache the result.

**Q: Why is my new campaign still paused an hour later?**
Every Dappier campaign is born paused and requires (a) Dappier sales to configure the GAM line item offline and (b) a reviewer to approve. Calling `update_media_buy { paused: false }` before those steps complete returns `status: "submitted"` with an explanatory message, not an error.

**Q: Can I set a budget?**
Pricing is offline. Budget/CPM/ROAS fields are not honored. `pricing_option_id` is always `"contact_sales"`.

**Q: Can I target by geo / device / audience?**
No. Dappier targeting is `dappier_network` (whole network), `my_network` (your own publishers), or `individual_agents` (list of `am_xxxx` ids).

**Q: What happens if I pass extra fields the server doesn't understand?**
Fields defined by AdCP but unused by Dappier (`idempotency_key`, `budget`, `include_snapshot`, `include_history`, `reporting_dimensions`, format-filter arguments, etc.) are accepted for forward compatibility but ignored. Fields that are targeting dimensions Dappier doesn't support will cause validation errors.

**Q: How do I correlate a tool call with my own request id?**
Pass `context: { your_request_id: "..." }` on any tool — it's echoed back unchanged.

**Q: Is the cursor format stable?**
`get_media_buys` currently uses `page:<n>` cursors. Treat it as opaque — don't parse or construct it yourself.

**Q: Can I create multiple creatives on one campaign?**
No. On CREATE, the target campaign must not already have a creative attached. Use UPDATE mode to modify the existing Brand Agent.

---

## 12. Glossary

| Term | Meaning |
|---|---|
| **AdCP** | Advertising Context Protocol — an open spec for AI-callable ad-platform tools. |
| **MCP** | Model Context Protocol — the transport/framing standard this server speaks. |
| **Media buy** | An AdCP term for a campaign. One Dappier campaign = one media buy. |
| **Package** | An AdCP subdivision of a media buy. Dappier has no native package model — each campaign is returned with a single synthetic package. |
| **Sponsored Conversations** | Dappier's only product. A branded prompt pill that opens into a full Brand Agent conversation. |
| **Brand Agent** | The conversational AI creative attached to a campaign. Id prefix `am_`. |
| **Sponsored prompt** | The clickable prompt text shown to users. Id prefix `pm_`. |
| **GAM** | Google Ad Manager — where Dappier sales configures line items offline before a campaign can serve. |
| **Dappier network** | The set of publishers running Dappier AI chat widgets. |
