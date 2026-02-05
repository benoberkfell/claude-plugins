# Claude Code Plugin Marketplace

A personal collection of Claude Code plugins, skills, and agents.

## Available Plugins

### Showkase
Skills for setting up and using Airbnb's Showkase library in Jetpack Compose projects.
- `showkase-install` - Install and configure Showkase in Android/Compose projects
- `showkase-usage` - Use Showkase annotations to document Compose UI elements

### Dataview
Skills for using Dataview in Obsidian - query syntax, metadata handling, troubleshooting, and JavaScript API.
- `dataview-metadata` - Add and structure metadata fields in Obsidian notes
- `dataview-queries` - Write DQL queries for listing, filtering, grouping data
- `dataview-javascript` - Write DataviewJS code for custom logic and rendering
- `dataview-troubleshooting` - Debug Dataview issues and common problems

## Adding This Marketplace

To add this marketplace to Claude Code, use the `/plugin marketplace add` command:

```bash
/plugin marketplace add https://github.com/benoberkfell/claude-plugins
```

Or if working locally:

```bash
/plugin marketplace add ./path/to/claude-plugins
```

## Installing Plugins

Once the marketplace is added, install individual plugins:

```bash
/plugin install plugin-name@ben-plugins
```

## Adding New Plugins

1. Create a new directory under `plugins/`:
   ```bash
   mkdir -p plugins/your-plugin-name/.claude-plugin
   ```

2. Add your plugin files (commands, skills, agents, etc.)

3. Create `.claude-plugin/plugin.json`:
   ```json
   {
     "name": "your-plugin-name",
     "description": "Description of your plugin",
     "version": "1.0.0"
   }
   ```

4. Update `.claude-plugin/marketplace.json` to include your plugin:
   ```json
   {
     "name": "your-plugin-name",
     "source": "./plugins/your-plugin-name",
     "description": "Description of your plugin",
     "version": "1.0.0"
   }
   ```

## Structure

```
.claude-plugin/
  marketplace.json      # Marketplace configuration
plugins/
  plugin-name/
    .claude-plugin/
      plugin.json       # Plugin manifest
    commands/           # Custom commands
    skills/             # Skills
    agents/             # Agents
```

## Validation

Validate your marketplace before committing:

```bash
claude plugin validate .
```

Or from within Claude Code:

```
/plugin validate .
```
