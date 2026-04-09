# GC Company Claude Plugins

Shared Claude Code plugins for GC Company engineering teams.

**[한국어 사용 가이드 →](USAGE_GUIDE.md)**

## Setup

Add this marketplace (one-time):

    /plugin marketplace add kunsumer/gc-claude-plugins

Install plugins:

    /plugin install gc-korea-compliance@gc-claude-plugins
    /plugin install gc-safe-migration@gc-claude-plugins
    /plugin install gc-yeogibrain-guide@gc-claude-plugins

## Available plugins

| Plugin | Description |
|---|---|
| gc-korea-compliance | Korea OTA security, privacy, and regulatory compliance review |
| gc-safe-migration | Schema migration safety patterns for Spring Boot + MyBatis + MySQL |
| gc-yeogibrain-guide | Effective usage of the yeogibrain MCP server |

## Contributing

1. Create a branch
2. Add or edit a plugin under `plugins/`
3. Bump `package.json` version for existing plugins
4. Update `marketplace.json` if adding a new plugin
5. Open a PR

### Quality gates

- **Org-wide:** Must be useful across 2+ projects
- **Self-contained:** Must not depend on a specific repo's files or paths
- **Clear triggers:** SKILL.md `description` must clearly state when the skill activates
- **References bundled:** Include reference files, don't assume external docs exist

## Plugin structure

Each plugin follows:

```
plugins/my-plugin/
├── package.json           # { "name": "my-plugin", "version": "1.0.0" }
├── CLAUDE.md              # (optional) context injected when plugin is active
├── skills/
│   └── my-skill/
│       ├── SKILL.md       # skill entry point
│       └── references/    # (optional) reference files loaded on demand
├── commands/              # (optional) slash commands
└── agents/                # (optional) agent definitions
```
