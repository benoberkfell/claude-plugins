# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is a Claude Code plugin marketplace - a catalog for distributing plugins, skills, and agents to users. The marketplace is defined by `.claude-plugin/marketplace.json` and contains organized plugin directories that can be installed via Claude Code CLI commands.

## Maintaining This File

This CLAUDE.md should be kept up to date as the marketplace evolves:
- When adding a new plugin, update the "Current Plugins" section
- When changing plugin structure or adding new patterns, update the "Architecture" or "Common Tasks" sections
- When updating marketplace.json schema or Claude Code plugin features become relevant, update "References"
- Use this as the source of truth for how the marketplace is organized and how to maintain it

## Architecture

### Marketplace Structure

- **`.claude-plugin/marketplace.json`** - The marketplace definition file. This is the entry point that Claude Code reads to discover available plugins. It defines the marketplace name, owner, and references all plugins.
- **`plugins/`** - Directory containing individual plugins
  - Each plugin is a self-contained directory with its own `.claude-plugin/plugin.json` manifest
  - Plugins can contain multiple skills, commands, agents, hooks, or MCP/LSP servers

### Plugin Organization

Each plugin under `plugins/` follows this structure:

```
plugins/plugin-name/
├── .claude-plugin/
│   └── plugin.json          # Plugin manifest with metadata and configuration
├── skills/                  # SKILL.md files for skills
│   └── skill-name/
│       └── SKILL.md
├── commands/                # Command definition files
├── agents/                  # Agent definition files
├── hooks/                   # Hook configuration files
└── README.md                # Plugin documentation (optional)
```

### How Plugins Are Referenced

In `marketplace.json`, each plugin entry includes:
- `name` - Kebab-case identifier (public-facing, used in install commands). Note: Cannot contain "claude", "anthropic", or official-sounding combinations as these are reserved for Anthropic
- `source` - Relative path to plugin directory (relative paths only work for git-based distribution)
- `description` - Brief description users see when discovering plugins
- `version` - Semantic version for the plugin

## Adding New Plugins

1. Create plugin directory: `mkdir -p plugins/plugin-name/.claude-plugin`
2. Create manifest: `plugins/plugin-name/.claude-plugin/plugin.json` with name, description, version
3. Add plugin content (skills, commands, agents, etc.) to appropriate subdirectories
4. Add entry to `.claude-plugin/marketplace.json` with name, source path, description, and version
5. Validate with `claude plugin validate .`
6. Commit and push changes

### Skills in Plugins

Skills are added as SKILL.md files in `plugins/plugin-name/skills/skill-name/SKILL.md`. Each skill should have:
- Frontmatter with `name` and `description`
- Clear, progressive disclosure structure
- Related references in subdirectories if needed

## Validation and Distribution

### Local Testing

Test a plugin locally before committing:
```bash
/plugin marketplace add ./path/to/claude-plugins
/plugin install plugin-name@claude-plugins
```

### Validation

Validate marketplace structure:
```bash
claude plugin validate .
```

or from Claude Code:
```
/plugin validate .
```

### Distribution

This marketplace is distributed via GitHub at https://github.com/benoberkfell/claude-plugins. Users add it with:
```bash
/plugin marketplace add https://github.com/benoberkfell/claude-plugins
```

Marketplace updates are pulled with:
```bash
/plugin marketplace update
```

## Common Tasks

### Add a new skill to an existing plugin

1. Create `plugins/plugin-name/skills/skill-name/SKILL.md`
2. Add frontmatter with name and description
3. No changes needed to marketplace.json (skills are discovered from plugin directory)
4. Validate and commit

### Create a new plugin with multiple skills

1. Create plugin structure: `mkdir -p plugins/new-plugin/.claude-plugin/skills`
2. Create `plugins/new-plugin/.claude-plugin/plugin.json`
3. Add SKILL.md files under `plugins/new-plugin/skills/`
4. Add plugin entry to `.claude-plugin/marketplace.json`
5. Validate and commit

### Version a plugin

Update version in both:
- `plugins/plugin-name/.claude-plugin/plugin.json`
- `.claude-plugin/marketplace.json` entry for that plugin

## Current Plugins

### Showkase
- **Description**: Skills for setting up and using Airbnb's Showkase library in Jetpack Compose projects
- **Skills**: showkase-install, showkase-usage
- **Location**: `plugins/showkase/`

### Dataview
- **Description**: Skills for using Dataview in Obsidian - query syntax, metadata handling, troubleshooting, and JavaScript API
- **Skills**: dataview-javascript, dataview-metadata, dataview-queries, dataview-troubleshooting
- **Location**: `plugins/dataview/`
- **Documentation**: `plugins/dataview/README.md` contains decision tree and workflow examples

### Metro
- **Description**: Skills for Metro, a compile-time dependency injection framework for Kotlin Multiplatform
- **Skills**: metro-install, metro-usage, metro-debugging
- **Location**: `plugins/metro/`
- **Repository**: https://github.com/ZacSweers/metro

## References

- [Claude Code Plugin Documentation](https://code.claude.com/docs/plugins)
- [Marketplace Schema Reference](https://code.claude.com/docs/plugins/distributing-plugins)
- [Plugin Settings Documentation](https://code.claude.com/docs/settings#plugin-settings)
