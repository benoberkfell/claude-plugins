---
name: create-plugin-from-repo
description: Clone an OSS repository, analyze its documentation and code, and generate a Claude Code plugin with appropriate skills for that library/framework.
---

# Create Plugin from Repository

Generate a Claude Code plugin by analyzing an open source repository.

## Usage

```
/create-plugin-from-repo <repo-url> [plugin-name]
```

- `repo-url`: GitHub URL or git clone URL for the repository
- `plugin-name`: Optional name for the plugin (defaults to repo name)

## Process

### Step 1: Clone Repository

Clone the repository to a temporary directory:

```bash
TEMP_DIR=$(mktemp -d)
git clone --depth 1 <repo-url> "$TEMP_DIR/repo"
cd "$TEMP_DIR/repo"
```

### Step 2: Analyze Repository Structure

Explore the repository to understand its purpose and structure:

1. **Read primary documentation**:
   - README.md (or README.rst, README.txt)
   - docs/ directory if present
   - Any GETTING_STARTED, QUICKSTART, or TUTORIAL files

2. **Identify the technology**:
   - Check build files (build.gradle, package.json, Cargo.toml, pyproject.toml, etc.)
   - Determine the language and ecosystem
   - Note any framework dependencies

3. **Examine source structure**:
   - Main source directories
   - Example or sample directories
   - Test directories (often contain usage patterns)

4. **Look for API surface**:
   - Public APIs, annotations, decorators
   - Configuration options
   - Extension points

### Step 3: Identify Skill Categories

Based on analysis, determine what skills would help users. Common categories:

| Category | When to Create | Examples |
|----------|----------------|----------|
| **Installation** | Library requires setup/configuration | Dependencies, build config, initialization |
| **Usage** | Library has APIs/patterns to learn | Core APIs, common patterns, best practices |
| **Debugging** | Library has known issues or complex errors | Error messages, troubleshooting, common mistakes |
| **Migration** | Library has version upgrades or alternatives | Upgrade guides, migration from other tools |
| **Advanced** | Library has complex/power-user features | Performance tuning, customization, plugins |

### Step 4: Generate Plugin Structure

Create the plugin in the target marketplace:

```
plugins/<plugin-name>/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   ├── <plugin-name>-install/
│   │   └── SKILL.md
│   ├── <plugin-name>-usage/
│   │   └── SKILL.md
│   └── <plugin-name>-debugging/
│       └── SKILL.md
└── README.md (optional)
```

### Step 5: Write Skill Content

For each skill, create a SKILL.md with:

1. **Frontmatter**: name and description
2. **Overview**: What the skill helps with
3. **Key Information**: Extracted from docs/code
4. **Examples**: Real examples from the repo
5. **Common Patterns**: Frequently used patterns
6. **Troubleshooting**: Known issues and solutions (for debugging skills)

#### Skill Writing Guidelines

- **Be specific**: Include actual code patterns, not generic advice
- **Use repo examples**: Pull real examples from docs, tests, or samples
- **Include versions**: Note which versions the information applies to
- **Link to sources**: Reference official docs where appropriate
- **Progressive disclosure**: Start simple, add complexity gradually

### Step 6: Update Marketplace

Add the plugin entry to `.claude-plugin/marketplace.json`:

```json
{
  "name": "<plugin-name>",
  "source": "./plugins/<plugin-name>",
  "description": "<description from README>",
  "version": "1.0.0"
}
```

### Step 7: Cleanup

Remove the temporary directory:

```bash
rm -rf "$TEMP_DIR"
```

## Output

After completion, report:
- Plugin location
- Skills created with brief descriptions
- Any manual steps needed (e.g., adding to marketplace.json)
- Suggestions for additional skills that could be added later

## Example

```
/create-plugin-from-repo https://github.com/square/retrofit retrofit
```

This would:
1. Clone the Retrofit repository
2. Analyze its documentation and source
3. Create a `plugins/retrofit/` directory
4. Generate skills like `retrofit-install`, `retrofit-usage`, `retrofit-debugging`
5. Update marketplace.json
