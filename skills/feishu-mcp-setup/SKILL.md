---
name: feishu-mcp-setup
description: "Set up the Feishu (Lark) MCP Connector from scratch for Claude Desktop / Cowork / Claude Code. Use this skill whenever the user wants to: connect Claude to Feishu, set up Feishu/Lark API access, create a Feishu custom app for MCP, configure lark-mcp, troubleshoot Feishu MCP authentication errors (99991672, 99991663, 20029), fix user_access_token or tenant_access_token issues, add Feishu permission scopes, or anything related to initially establishing the Feishu MCP connection. Also trigger when the user mentions 'lark-mcp', '@larksuiteoapi/lark-mcp', or asks about connecting to Feishu wiki/docs/bitable via API. Do NOT use this skill for day-to-day Feishu MCP operations after setup is complete — use feishu-mcp-usage instead."
---

# Feishu (Lark) MCP Connector — Setup Skill

This skill guides you through setting up the Feishu MCP Connector from zero to a fully working connection. It covers app creation, permission scopes, OAuth login, MCP config, and all known pitfalls.

## When to Use This Skill

Use this skill when the agent needs to:

- Create a new Feishu custom app and connect it to Claude
- Configure permission scopes (tenant and user token scopes)
- Set up OAuth login for user_access_token
- Write or fix the MCP server config in `claude_desktop_config.json`
- Troubleshoot authentication or permission errors during initial setup

Once the MCP connection is established and working, switch to the **feishu-mcp-usage** skill for day-to-day operations.

## Quick Setup Flow

```
1. Create custom app → get App ID + Secret
2. Add scopes (BOTH tenant AND user token tabs)
3. Register redirect URL: http://localhost:3000/callback
4. Run OAuth login with --scope offline_access
5. Configure MCP in claude_desktop_config.json
   → For user token: add --oauth + --token-mode user_access_token
6. Restart Claude Desktop → verify connection
```

## Detailed Instructions

Read the full setup guide for step-by-step instructions with all pitfalls and solutions:

```
references/setup_guide.md
```

This reference file contains:

- Architecture diagram and authentication modes
- Token lifecycle (tenant ~2hrs auto-refresh, user ~2hrs with ~30-day refresh token)
- Step-by-step setup (6 steps with detailed substeps)
- 8 documented pitfalls with symptoms, root causes, and fixes
- MCP config templates for both tenant-only and user token modes
- API capabilities matrix
- Quick reference card with commands and example tokens

## Critical Pitfalls Summary

These are the most common issues — check them first when debugging:

| # | Problem | Quick Fix |
|---|---------|-----------|
| 1 | Wrong scope (`docs:` vs `docx:`) | Add `docx:document:readonly` |
| 4 | user_access_token "invalid" | Add `--oauth` + `--token-mode user_access_token` to MCP config |
| 8 | Scope works for tenant but not user | Add scopes under "User token scopes" tab separately |

## Config Templates

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

## Verification

After setup, run these three health checks:

```python
# Tenant token — read document
mcp__lark__docx_v1_document_rawContent(path={"document_id": "<obj_token>"}, useUAT=False)

# Tenant token — resolve wiki node
mcp__lark__wiki_v2_space_getNode(params={"token": "<node_token>"}, useUAT=False)

# User token — search wiki
mcp__lark__wiki_v1_node_search(data={"query": "test"}, useUAT=True)
```

If all three succeed, setup is complete. Switch to **feishu-mcp-usage** for operations.
