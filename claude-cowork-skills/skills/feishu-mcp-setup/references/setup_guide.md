# Feishu (Lark) MCP Connector — Agent Setup Guide

> **Purpose**: Step-by-step runbook for an AI agent (Claude Cowork / Claude Code) to set up and use the Feishu MCP Connector for reading wiki documents via API.
>
> **Last verified**: 2026-03-07 | **MCP package**: `@larksuiteoapi/lark-mcp`

---

## TL;DR — Minimal Happy Path

```
1. Create custom app on https://open.feishu.cn  →  get App ID + App Secret
2. Add scopes via "Batch import/export scopes" button → Import tab → paste the JSON from Step 2 below
   (Or manually add under BOTH "Tenant token scopes" AND "User token scopes" tabs)
3. Security Settings → Redirect URLs → add http://localhost:3000/callback
4. Terminal: npx -y @larksuiteoapi/lark-mcp login -a <APP_ID> -s <APP_SECRET> --scope offline_access docx:document
5. Add MCP config to claude_desktop_config.json → restart Claude Desktop
   ⚠️ For user token mode, add: "--oauth", "--token-mode", "user_access_token" to args
6. Verify: call wiki_v1_node_search (useUAT=true) and docx_v1_document_rawContent
```

---

## 1. Architecture

```
┌─────────────────┐     stdio/MCP      ┌──────────────────────┐     HTTPS     ┌─────────────────┐
│  Claude Desktop  │ ◄────────────────► │  lark-mcp (local)    │ ◄───────────► │  Feishu Open API │
│  (Cowork/Code)   │                    │  npm: @larksuiteoapi  │               │                 │
│                  │                    │  /lark-mcp            │               │                 │
└─────────────────┘                    └──────────────────────┘               └─────────────────┘
        │                                       │
        │  Calls MCP tools like:                │  Uses either:
        │  - docx_v1_document_rawContent        │  - tenant_access_token (app identity)
        │  - wiki_v2_space_getNode              │  - user_access_token (user identity via OAuth)
        │  - wiki_v1_node_search                │
        └───────────────────────────────────────┘
```

### Authentication Modes

| Mode | Identity | Pros | Cons |
|------|----------|------|------|
| `tenant_access_token` | App itself | Auto-refreshes; no user login | App must be published & added to docs; limited API coverage |
| `user_access_token` | User (OAuth) | Full API access; reads all user-accessible docs | Requires OAuth login; token expires |
| **Hybrid (recommended)** | Both | Best coverage | Slightly more complex |

### Token Lifecycle & Refresh

| Token Type | Lifetime | Refresh Mechanism |
|------------|----------|-------------------|
| `tenant_access_token` | ~2 hours | Auto-refreshed by MCP server using App ID + Secret. No user action needed. |
| `user_access_token` | ~2 hours | Refreshed using `refresh_token`. MCP server handles this automatically if `--oauth` is set. |
| `refresh_token` | ~30 days | Obtained during OAuth login with `offline_access` scope. Each refresh produces a new refresh_token. |

**Key implications for agents**:

- `tenant_access_token` is fully automatic — the MCP server refreshes it silently using the App ID and Secret. No human intervention needed.
- `user_access_token` requires an initial OAuth login (Step 4), but afterwards the MCP server uses the `refresh_token` to auto-renew. As long as the agent is used at least once every ~30 days, no re-login is needed.
- If the `refresh_token` expires (>30 days of inactivity), the user must re-run the `lark-mcp login` command.

---

## 2. Prerequisites

- **Node.js ≥ 20.0.0** on the host machine
- A **Feishu account** with access to target documents
- **Claude Desktop** (Cowork) or **Claude Code** installed

---

## 3. Step-by-Step Setup

### Step 1: Create a Feishu Custom App

1. Go to **https://open.feishu.cn** → log in
2. Click **Developer Console** (top-right)
3. Click **Create Custom App**
4. Enter app name (e.g. `"Alphy Agent"`) and description
5. Record the two credentials:
   - **App ID**: e.g. `cli_a926b9902b219bd6`
   - **App Secret**: click 👁 eye icon to reveal, then copy

> **⚠️ CRITICAL**: The App ID and App Secret are needed for BOTH `tenant_access_token` and `user_access_token` modes. The app is the OAuth client. Creating the app is **never wasted** regardless of which auth mode you end up using.

---

### Step 2: Configure Permission Scopes

Navigate to: **Development Configuration → Permissions & Scopes → Add permission scopes to app**

Add the following scopes. **⚠️ Scopes must be added under BOTH "Tenant token scopes" AND "User token scopes" tabs** — they are managed independently.

**Tenant token scopes** (for app-identity access):

| Scope | Purpose | Priority |
|-------|---------|----------|
| `docx:document:readonly` | Read document content (new docx API) | **CRITICAL** |
| `docx:document` | Create and edit documents | **CRITICAL** |
| `docs:doc` | View, comment, edit, and manage Docs (legacy) | **CRITICAL** |
| `drive:drive` | View, comment, edit, and manage all files in My Space | **CRITICAL** |
| `drive:drive:readonly` | View, comment, and download all files in My Space | Required |
| `wiki:node:read` | View wiki node info | Required |
| `wiki:node:retrieve` | List wiki nodes | Required |
| `wiki:wiki:readonly` | View all wiki content | Required |
| `wiki:space:retrieve` | View wiki space list | Recommended |
| `bitable:app` | View, comment, edit and manage Base (bitable) | Required |
| `sheets:spreadsheet` | View, comment, edit, and manage Sheets | Required |
| `docs:document.content:read` | Read doc content (legacy API) | Optional |

**User token scopes** (for user-identity access via OAuth):

| Scope | Purpose | Priority |
|-------|---------|----------|
| `docx:document:readonly` | Read document content | **CRITICAL** |
| `docx:document` | Create and edit documents | **CRITICAL** |
| `docs:doc` | View, comment, edit, and manage Docs (legacy) | **CRITICAL** |
| `drive:drive` | View, comment, edit, and manage all files in My Space | **CRITICAL** |
| `drive:drive:readonly` | View, comment, and download all files in My Space | Required |
| `wiki:wiki:readonly` | View all wiki content | **CRITICAL** |
| `wiki:wiki` | View, edit, and manage wiki | **CRITICAL** |
| `search:docs:read` | Search wiki and documents | **CRITICAL** |
| `bitable:app` | View, comment, edit and manage Base (bitable) | Required |
| `sheets:spreadsheet` | View, comment, edit, and manage Sheets | Required |

#### Quick Method: Batch Import via JSON (Recommended)

Instead of checking scopes one by one, use the **"Batch import/export scopes"** button on the Permissions & Scopes page. Click it, switch to the **Import** tab, paste the following JSON, then click **"Next, Review New Scopes"**:

```json
{
  "scopes": {
    "tenant": [
      "sheets:spreadsheet",
      "bitable:app",
      "docs:doc",
      "docs:document.content:read",
      "docx:document",
      "docx:document:readonly",
      "drive:drive",
      "drive:drive:readonly",
      "wiki:node:read",
      "wiki:node:retrieve",
      "wiki:space:retrieve",
      "wiki:wiki:readonly"
    ],
    "user": [
      "sheets:spreadsheet",
      "bitable:app",
      "docs:doc",
      "docx:document",
      "docx:document:readonly",
      "drive:drive",
      "drive:drive:readonly",
      "search:docs:read",
      "wiki:wiki",
      "wiki:wiki:readonly"
    ]
  }
}
```

This single import covers **both tenant and user scopes** at once. The imported scopes will be incorporated in the upcoming version request and will not affect previously requested or added scopes.

> **💡 TIP**: You can also use the **Export** tab to export your current scopes as JSON for backup or sharing with other apps.

> **🔴 PITFALL #1 — Wrong scope for document reading**
>
> The old scope `docs:document.content:read` does **NOT** cover the new `docx_v1_document_rawContent` API. You **must** add `docx:document:readonly` separately.
>
> - **Symptom**: Error `99991672` — "Access denied. One of the following scopes is required: [docx:document, docx:document:readonly]"
> - **Fix**: Add `docx:document:readonly` in the Permissions & Scopes page.

> **🔴 PITFALL #8 — User token scopes must be activated separately**
>
> Feishu manages **Tenant token scopes** and **User token scopes** as two independent sets. Adding a scope under "Tenant token scopes" does NOT automatically enable it for user tokens. You must click "Add permission scopes to app", switch to the **"User token scopes"** tab, and add the scopes there too.
>
> - **Symptom**: Error `99991672` — "应用尚未开通所需的用户身份权限" (App has not enabled required user identity permissions), even though the same scope works with `useUAT: false`
> - **Fix**: Go to Permissions & Scopes → Add permission scopes to app → switch to "User token scopes" tab → check all needed scopes → Add Scopes. After adding, re-run `lark-mcp login` to obtain a new token that includes the updated scopes.

---

### Step 3: Configure Redirect URL

Navigate to: **Development Configuration → Security Settings → Redirect URLs**

Add:
```
http://localhost:3000/callback
```

> **🔴 PITFALL #2 — OAuth redirect_uri not registered**
>
> The `lark-mcp login` command starts a local HTTP server on port 3000 for the OAuth callback. If this URL isn't registered, authentication fails.
>
> - **Symptom**: Error code `20029` — "redirect_uri is invalid"
> - **Fix**: Add `http://localhost:3000/callback` to Redirect URLs before running login.

---

### Step 4: Run OAuth Login

In a terminal on the **host machine** (not inside Cowork VM):

```bash
# Basic login (minimal)
npx -y @larksuiteoapi/lark-mcp login -a <APP_ID> -s <APP_SECRET>

# Recommended: login with explicit scopes (enables offline refresh)
npx -y @larksuiteoapi/lark-mcp login -a <APP_ID> -s <APP_SECRET> --scope offline_access docx:document
```

The `--scope` parameter requests specific OAuth scopes during login. `offline_access` enables long-lived refresh tokens (~30 days), so you don't need to re-authorize frequently.

Example:
```bash
npx -y @larksuiteoapi/lark-mcp login -a cli_a926b9902b219bd6 -s <secret> --scope offline_access docx:document
```

A browser window opens → click **Authorize** → terminal confirms success.

> **🟡 PITFALL #3 — user_access_token may not reach Cowork MCP Server**
>
> The login stores the token locally, but the Cowork MCP process may run in a sandboxed VM with a different filesystem. Calls with `useUAT: true` may still fail.
>
> - **Symptom**: `"Current user_access_token is invalid or expired"` even after successful login
> - **Workaround**: Use `tenant_access_token` (`useUAT: false`) as the primary mode for basic reads. It auto-refreshes and works for `docx_v1_document_rawContent` and `wiki_v2_space_getNode`.

---

### Step 5: Configure MCP Server in Claude Desktop

Edit the config file:

| OS | Path |
|----|------|
| macOS | `~/Library/Application Support/Claude/claude_desktop_config.json` |
| Windows | `%APPDATA%\Claude\claude_desktop_config.json` |

Access via: **Claude Desktop → Settings → Developer → Edit Config**

Add one of the following configs depending on your authentication mode:

**Mode A — Tenant token only** (simpler, auto-refreshes, limited API coverage):

```json
{
  "mcpServers": {
    "lark": {
      "command": "npx",
      "args": [
        "-y",
        "@larksuiteoapi/lark-mcp",
        "mcp",
        "-a", "<APP_ID>",
        "-s", "<APP_SECRET>"
      ]
    }
  }
}
```

**Mode B — User token mode (recommended)** (full API access, requires prior OAuth login):

```json
{
  "mcpServers": {
    "lark": {
      "command": "npx",
      "args": [
        "-y",
        "@larksuiteoapi/lark-mcp",
        "mcp",
        "-a", "<APP_ID>",
        "-s", "<APP_SECRET>",
        "--oauth",
        "--token-mode", "user_access_token"
      ]
    }
  }
}
```

> **🔴 CRITICAL**: The `--oauth` and `--token-mode user_access_token` flags are **required** to enable user token mode. Without them, the MCP server will only use `tenant_access_token`, even if you have successfully completed OAuth login. This is the most common reason for `"user_access_token is invalid or expired"` errors.

Then **fully quit and restart Claude Desktop** (not just close the window).

> **🔴 PITFALL #4 — Missing `--oauth` and `--token-mode` flags in MCP config**
>
> The MCP server defaults to `tenant_access_token` mode. Even after successful OAuth login, if the config doesn't include `--oauth` and `--token-mode user_access_token`, the server never attempts to use user tokens.
>
> - **Symptom**: `"Current user_access_token is invalid or expired"` despite successful OAuth login
> - **Root Cause**: Missing `--oauth` and `--token-mode user_access_token` in the `args` array
> - **Fix**: Add both flags to the MCP server config and restart Claude Desktop
>
> **Also note**: MCP Servers are loaded at startup. Editing config without a full restart has no effect.

---

### Step 6: Verify the Connection

Run these test calls (in Claude conversation):

**Test A — Read a document** (requires `obj_token`, not `node_token`):
```
docx_v1_document_rawContent(document_id="<OBJ_TOKEN>", useUAT=false)
```

**Test B — Get wiki node info** (accepts `node_token` from URL):
```
wiki_v2_space_getNode(token="<NODE_TOKEN>", useUAT=false)
```

**Test C — Search wiki** (user token only):
```
wiki_v1_node_search(query="keyword", useUAT=true)
```

---

## 4. Token System — Critical Concept

Feishu wiki uses **two token types**. Confusing them is the most common error.

```
Wiki URL:  https://my.feishu.cn/wiki/GtHEw1G1Fi4NwrklS0acbpbInKe
                                       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                                       This is the NODE_TOKEN

wiki_v2_space_getNode(token=NODE_TOKEN) returns:
  → obj_token: "AMygdCOOqojC5Vx0i8acJhoKn5f"   ← This is the OBJ_TOKEN

docx_v1_document_rawContent(document_id=OBJ_TOKEN) returns:
  → The actual document text
```

**Resolution flow**:
```
Wiki URL → node_token → wiki_v2_space_getNode() → obj_token → docx_v1_document_rawContent() → text
```

> **🔴 PITFALL #5 — Using node_token where obj_token is required**
>
> `docx_v1_document_rawContent` takes `obj_token`, NOT `node_token`. Passing the wrong one returns empty or error.
>
> - **Fix**: Always call `wiki_v2_space_getNode` first to resolve `node_token` → `obj_token`.

---

## 5. API Capabilities Matrix

### ✅ Working with `tenant_access_token` (useUAT: false)

| API | Function | Input | Notes |
|-----|----------|-------|-------|
| `docx_v1_document_rawContent` | Read document plain text | `obj_token` | Core capability; returns text only (no tables/images) |
| `wiki_v2_space_getNode` | Get wiki node metadata | `node_token` | Returns obj_token, title, parent, space_id, etc. |
| `bitable_v1_*` | Base (spreadsheet) operations | `app_token` | Full CRUD on base tables |
| `im_v1_*` | Messaging operations | varies | Requires additional `im:` scopes |

### ✅ Working with `user_access_token` only (useUAT: true)

| API | Function | Input | Notes |
|-----|----------|-------|-------|
| `wiki_v1_node_search` | Search wiki nodes by keyword | `query` string | Returns node_token + obj_token for each result; supports pagination |
| `docx_builtin_search` | Full-text document search | `search_key` string | Searches across all accessible docs |
| `docx_builtin_import` | Import markdown as document | `markdown` string | Creates a new Feishu doc from markdown content |

> **Note**: These APIs return `99991663` (invalid access token) with tenant token. You must use `useUAT: true` AND have user token scopes enabled (see Pitfall #8).

### ✅ Full Document Automation (requires both token types with full scopes)

| API | Function | Token | Notes |
|-----|----------|-------|-------|
| `docx_builtin_import` | Create new document from markdown | User | Requires `drive:drive` + `docs:doc` scopes |
| `bitable_v1_app_create` | Create new Base app | Either | Requires `bitable:app` scope |
| `bitable_v1_appTable_create` | Create table in Base | Either | Requires `bitable:app` scope |
| `bitable_v1_appTableRecord_create` | Add records to Base table | Either | Requires `bitable:app` scope |
| `bitable_v1_appTableRecord_search` | Search records in Base | Either | Requires `bitable:app` scope |
| `bitable_v1_appTableRecord_update` | Update records in Base | Either | Requires `bitable:app` scope |

### ⚠️ Known Limitations of `rawContent`

The `docx_v1_document_rawContent` API returns **plain text only**. It does NOT include:
- Table structures (tables appear as empty lines)
- Image references
- Synced block / embedded sub-page content
- Rich formatting (bold, links, etc.)

**Workaround**: For rich content, combine API with browser-based extraction:
1. Navigate to the page in browser
2. Use `get_page_text` or JavaScript to extract formatted content
3. Use `docx_v1_document_rawContent` for sub-documents linked within the page

---

## 6. All Challenges, Pitfalls & Solutions

| # | Problem | Error | Root Cause | Solution |
|---|---------|-------|------------|----------|
| 1 | Document read fails | `99991672` | Wrong scope: `docs:` vs `docx:` | Add `docx:document:readonly` |
| 2 | OAuth login fails | `20029` | Redirect URL not registered | Add `http://localhost:3000/callback` to Security Settings |
| 3 | App not found in wiki member search | "No results" | App not published | Use `user_access_token` mode instead |
| 4 | user_access_token invalid in Cowork | "token invalid or expired" | Missing `--oauth` and `--token-mode user_access_token` in MCP config | Add both flags to config args; also ensure prior `lark-mcp login` was done on host |
| 5 | Wiki search fails with tenant token | `99991663` | API only supports user token | Enable user token mode (Pitfall #4) + user scopes (Pitfall #8) |
| 6 | rawContent returns empty for tables | Empty lines | API is text-only | Combine with browser extraction |
| 7 | node_token vs obj_token | Empty/error | Wrong token type | Always resolve via `wiki_v2_space_getNode` first |
| 8 | user_access_token scope denied | `99991672` + "用户身份权限" | Scopes only added for tenant, not user token | Add scopes under "User token scopes" tab separately; re-run login |

---

## 7. Recommended Agent Workflow

```
                    ┌──────────────────────┐
                    │  1. Search & Discover │
                    │  wiki_v1_node_search  │
                    │  (user token)         │
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │  2. Resolve Tokens    │
                    │  wiki_v2_getNode      │
                    │  (either token)       │
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │  3. Read Content      │
                    │  docx_v1_rawContent   │
                    │  (either token)       │
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │  4. Context Inject    │
                    │  Feed into Agent      │
                    └──────────────────────┘
```

1. **Search & discover**: Use `wiki_v1_node_search` (user token) to find documents by keyword. No browser automation needed — the API returns `node_token`, `obj_token`, `title`, and `url` for each match.

2. **Resolve tokens**: For documents discovered via URL (not search), call `wiki_v2_space_getNode` to resolve `node_token` → `obj_token`.

3. **Read content**: Use `docx_v1_document_rawContent` to fetch plain text. For rich content (tables, images), supplement with browser-based extraction.

4. **Context injection**: Feed relevant document context into ongoing conversations.

---

## 8. Quick Reference

### Commands

```bash
# OAuth login (basic)
npx -y @larksuiteoapi/lark-mcp login -a <APP_ID> -s <APP_SECRET>

# OAuth login (recommended — with offline refresh + document scope)
npx -y @larksuiteoapi/lark-mcp login -a <APP_ID> -s <APP_SECRET> --scope offline_access docx:document

# Start MCP server — tenant token only
npx -y @larksuiteoapi/lark-mcp mcp -a <APP_ID> -s <APP_SECRET>

# Start MCP server — user token mode (requires prior login)
npx -y @larksuiteoapi/lark-mcp mcp -a <APP_ID> -s <APP_SECRET> --oauth --token-mode user_access_token
```

### Config Templates

**Tenant token only:**
```json
{
  "mcpServers": {
    "lark": {
      "command": "npx",
      "args": ["-y", "@larksuiteoapi/lark-mcp", "mcp", "-a", "APP_ID", "-s", "APP_SECRET"]
    }
  }
}
```

**User token mode (recommended):**
```json
{
  "mcpServers": {
    "lark": {
      "command": "npx",
      "args": ["-y", "@larksuiteoapi/lark-mcp", "mcp", "-a", "APP_ID", "-s", "APP_SECRET", "--oauth", "--token-mode", "user_access_token"]
    }
  }
}
```

### Example Tokens (from this setup)

| Item | Value |
|------|-------|
| App ID | `cli_a926b9902b219bd6` |
| Wiki Space ID | `7418172726923657219` |
| Template_7.6 node_token | `GtHEw1G1Fi4NwrklS0acbpbInKe` |
| Template_7.6 obj_token | `AMygdCOOqojC5Vx0i8acJhoKn5f` |
| Redirect URL | `http://localhost:3000/callback` |

### MCP Tool Quick Ref

```python
# Read document (tenant token works)
mcp__lark__docx_v1_document_rawContent(path={"document_id": OBJ_TOKEN}, useUAT=False)

# Get node info (tenant token works)
mcp__lark__wiki_v2_space_getNode(params={"token": NODE_TOKEN}, useUAT=False)

# Search wiki (user token ONLY)
mcp__lark__wiki_v1_node_search(data={"query": "keyword"}, useUAT=True)

# Search docs (user token ONLY)
mcp__lark__docx_builtin_search(data={"search_key": "keyword"}, useUAT=True)
```
