# Claude Skills

Custom skills for [Claude Cowork](https://claude.ai) — reusable prompt & reference bundles that extend Claude's capabilities.

## Skills

| Skill | Description |
|-------|-------------|
| **feishu-mcp-setup** | Set up the Feishu (Lark) MCP Connector — custom app creation, permission scopes (with batch JSON import), OAuth login, MCP config |
| **feishu-mcp-usage** | Day-to-day Feishu MCP operations — document CRUD, wiki search, Base/bitable management |

## Installation

Each skill folder can be packaged as a `.skill` file (a zip archive) and installed into Claude Cowork via the "Copy to your skills" button.

## Structure

```
skills/
├── feishu-mcp-setup/
│   ├── SKILL.md                    # Skill definition & instructions
│   └── references/
│       └── setup_guide.md          # Detailed setup runbook with permission JSON
└── feishu-mcp-usage/
    └── SKILL.md                    # API reference & usage patterns
```

## License

MIT
