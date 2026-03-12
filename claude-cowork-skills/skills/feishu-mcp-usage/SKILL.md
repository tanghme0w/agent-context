---
name: feishu-mcp-usage
description: "Day-to-day operations with the Feishu (Lark) MCP Connector. Use this skill for: creating, reading, editing, searching Feishu cloud documents (docx); managing Base/bitable tables and records; working with wiki spaces and nodes; importing markdown as Feishu docs; searching across all document types. Do NOT use for initial setup — use feishu-mcp-setup instead."
---

# Feishu (Lark) MCP Connector — Usage Skill

This skill provides the reference for day-to-day Feishu MCP operations after setup is complete. It covers all available APIs, their required scopes, token types, and usage patterns.

## Prerequisites

Before using this skill, ensure the Feishu MCP connection is set up and working. If not, use the **feishu-mcp-setup** skill first.

### Required Permission Scopes (Full Automation)

**Tenant token scopes:**

| Scope | Purpose |
|-------|---------|
| `docx:document:readonly` | Read document content |
| `docx:document` | Create and edit documents |
| `docs:doc` | Legacy Docs CRUD |
| `drive:drive` | Manage all files in My Space |
| `drive:drive:readonly` | Download files in My Space |
| `wiki:node:read` | View wiki node info |
| `wiki:node:retrieve` | List wiki nodes |
| `wiki:wiki:readonly` | View wiki content |
| `wiki:space:retrieve` | View wiki space list |
| `bitable:app` | Manage Base (bitable) |
| `sheets:spreadsheet` | Manage Sheets |
| `docs:document.content:read` | Read doc content (legacy) |

**User token scopes:**

| Scope | Purpose |
|-------|---------|
| `docx:document:readonly` | Read document content |
| `docx:document` | Create and edit documents |
| `docs:doc` | Legacy Docs CRUD |
| `drive:drive` | Manage all files in My Space |
| `drive:drive:readonly` | Download files in My Space |
| `wiki:wiki:readonly` | View wiki content |
| `wiki:wiki` | Edit and manage wiki |
| `search:docs:read` | Search documents |
| `bitable:app` | Manage Base (bitable) |
| `sheets:spreadsheet` | Manage Sheets |

---

## API Quick Reference

### 1. Document Operations

#### Create a new document (from Markdown)

```python
# User token ONLY — requires drive:drive + docs:doc scopes
mcp__lark__docx_builtin_import(
    data={"markdown": "# Title\n\nContent here...", "file_name": "My Document"},
    useUAT=True
)
# Returns: { "result": { "token": "<doc_token>", "url": "https://my.feishu.cn/docx/<token>" } }
```

#### Read document content

```python
# Either token — requires docx:document:readonly
mcp__lark__docx_v1_document_rawContent(
    path={"document_id": "<OBJ_TOKEN>"},
    useUAT=False  # or True
)
# Returns: { "content": "plain text content..." }
# NOTE: Returns plain text only (no tables, images, or formatting)
```

#### Search documents

```python
# User token ONLY — requires search:docs:read
mcp__lark__docx_builtin_search(
    data={
        "search_key": "keyword",
        "count": 10,              # max 50
        "docs_types": ["docx", "sheet", "bitable"]  # optional filter
    },
    useUAT=True
)
# Returns: { "docs_entities": [...], "has_more": bool, "total": int }
```

---

### 2. Wiki Operations

#### Search wiki nodes

```python
# User token ONLY — requires wiki:wiki:readonly + search:docs:read
mcp__lark__wiki_v1_node_search(
    data={"query": "search keyword"},
    useUAT=True
)
# Returns: { "items": [{ "node_token": "...", "obj_token": "...", "title": "..." }] }
```

#### Get wiki node info (resolve node_token → obj_token)

```python
# Either token — requires wiki:node:read
mcp__lark__wiki_v2_space_getNode(
    params={"token": "<NODE_TOKEN>"},
    useUAT=False
)
# Returns: { "node": { "obj_token": "...", "obj_type": "docx", "title": "..." } }
```

#### Read wiki document content

```
# Two-step process:
# Step 1: Resolve node_token → obj_token
node_info = wiki_v2_space_getNode(token=NODE_TOKEN)
obj_token = node_info["node"]["obj_token"]

# Step 2: Read content using obj_token
content = docx_v1_document_rawContent(document_id=obj_token)
```

---

### 3. Base (Bitable / Multi-Dimensional Table) Operations

#### Create a new Base app

```python
# Either token — requires bitable:app
mcp__lark__bitable_v1_app_create(
    data={"name": "My Base App"},
    useUAT=True
)
```

#### List tables in a Base

```python
mcp__lark__bitable_v1_appTable_list(
    path={"app_token": "<APP_TOKEN>"},
    useUAT=True
)
```

#### Create a table in Base

```python
mcp__lark__bitable_v1_appTable_create(
    path={"app_token": "<APP_TOKEN>"},
    data={
        "table": {
            "name": "My Table",
            "fields": [
                {"field_name": "Name", "type": 1},       # 1 = Text
                {"field_name": "Status", "type": 3},      # 3 = SingleSelect
                {"field_name": "Priority", "type": 2},    # 2 = Number
                {"field_name": "Due Date", "type": 5}     # 5 = Date
            ]
        }
    },
    useUAT=True
)
```

#### Field type reference

| Type | Name | Description |
|------|------|-------------|
| 1 | Text | Multi-line text |
| 2 | Number | Numeric value |
| 3 | SingleSelect | Single option |
| 4 | MultiSelect | Multiple options |
| 5 | DateTime | Date/time |
| 7 | Checkbox | Boolean |
| 11 | User | Person field |
| 13 | PhoneNumber | Phone |
| 15 | Url | Hyperlink |
| 17 | Attachment | File attachment |
| 18 | Link | One-way link |
| 20 | Formula | Calculated |
| 21 | DuplexLink | Two-way link |
| 22 | Location | Geographic |

#### Search records

```python
mcp__lark__bitable_v1_appTableRecord_search(
    path={"app_token": "<APP_TOKEN>", "table_id": "<TABLE_ID>"},
    data={
        "field_names": ["Name", "Status"],  # optional: filter fields
        "filter": {
            "conjunction": "and",
            "conditions": [
                {"field_name": "Status", "operator": "is", "value": ["Active"]}
            ]
        },
        "sort": [{"field_name": "Name", "desc": False}]
    },
    params={"page_size": 100},
    useUAT=True
)
```

#### Create a record

```python
mcp__lark__bitable_v1_appTableRecord_create(
    path={"app_token": "<APP_TOKEN>", "table_id": "<TABLE_ID>"},
    data={
        "fields": {
            "Name": "New Item",
            "Status": "Active",
            "Priority": 1
        }
    },
    useUAT=True
)
```

#### Update a record

```python
mcp__lark__bitable_v1_appTableRecord_update(
    path={"app_token": "<APP_TOKEN>", "table_id": "<TABLE_ID>", "record_id": "<RECORD_ID>"},
    data={
        "fields": {
            "Status": "Completed"
        }
    },
    useUAT=True
)
```

---

### 4. List fields in a Base table

```python
mcp__lark__bitable_v1_appTableField_list(
    path={"app_token": "<APP_TOKEN>", "table_id": "<TABLE_ID>"},
    useUAT=True
)
```

---

## Token Type Decision Guide

| Operation | Use `useUAT=True` | Use `useUAT=False` | Notes |
|-----------|-------------------|---------------------|-------|
| Create document | **Yes** | No | `docx_builtin_import` requires user token |
| Read document | Either | **Preferred** | Tenant token auto-refreshes |
| Search documents | **Yes** | No | Search APIs are user-token-only |
| Search wiki | **Yes** | No | `wiki_v1_node_search` is user-token-only |
| Resolve wiki node | Either | **Preferred** | `wiki_v2_space_getNode` works with both |
| Base CRUD | Either | Either | Both work with `bitable:app` scope |

---

## Common Patterns

### Pattern 1: Find and read a wiki document

```
1. wiki_v1_node_search(query="keyword", useUAT=True)  → get node_token
2. wiki_v2_space_getNode(token=node_token, useUAT=False) → get obj_token
3. docx_v1_document_rawContent(document_id=obj_token, useUAT=False) → get text
```

### Pattern 2: Create a structured work document

```
1. docx_builtin_import(markdown="# Title\n...", useUAT=True) → get doc URL
```

### Pattern 3: Manage project tasks in Base

```
1. bitable_v1_appTable_list(app_token=...) → list tables
2. bitable_v1_appTableField_list(app_token=..., table_id=...) → understand fields
3. bitable_v1_appTableRecord_search(filter=...) → query records
4. bitable_v1_appTableRecord_create(fields=...) → add new records
5. bitable_v1_appTableRecord_update(record_id=..., fields=...) → update existing
```

---

## Error Handling

| Error Code | Meaning | Fix |
|------------|---------|-----|
| `99991672` | Missing permission scope | Add the required scope in Feishu developer console (both tenant AND user tabs) |
| `99991663` | Invalid access token | Use `useUAT=True` for user-token-only APIs; check token expiry |
| `20029` | Invalid redirect URI | Add `http://localhost:3000/callback` to app's Security Settings |
| Empty content | Wrong token type (node vs obj) | Resolve node_token → obj_token via `wiki_v2_space_getNode` first |

If you encounter persistent auth errors, refer to **feishu-mcp-setup** skill for troubleshooting.
