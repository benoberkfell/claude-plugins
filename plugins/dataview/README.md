# Dataview Skills Suite

Four focused Claude Code skills for working with Obsidian Dataview. Each skill handles one piece of the Dataview workflow.

## Skill Overview

### 1. **dataview-metadata** – Add metadata to your notes
Use when: Setting up fields for querying, adding inline fields, structuring Obsidian notes

Topics:
- Types of metadata (frontmatter, inline fields, implicit fields)
- Field naming conventions and data types
- Obsidian-specific patterns (daily notes, books, projects, tasks, recipes, etc.)

**Start here** if you're new to Dataview or setting up a new vault structure.

### 2. **dataview-queries** – Write DQL queries
Use when: Building queries with LIST, TABLE, TASK, or CALENDAR; filtering with WHERE; grouping with GROUP BY

Topics:
- The four query types and when to use each
- Data commands (FROM, WHERE, GROUP BY, SORT)
- Query patterns by use case
- Inline queries

**Go here** once you have metadata set up and want to display/search it.

### 3. **dataview-javascript** – Write DataviewJS code
Use when: DQL isn't powerful enough; need custom logic, rendering, or data transformation

Topics:
- `dv` API methods (pages, table, list, paragraph, header, etc.)
- Data manipulation (filtering, sorting, mapping, grouping)
- Real-world code examples

**Use this** when queries get too complex or you need custom rendering.

### 4. **dataview-troubleshooting** – Debug Dataview issues
Use when: Queries return no results, fields aren't indexed, tasks don't show, or unexpected behavior

Topics:
- Common problems and solutions
- Diagnostic checklist
- Error messages and what they mean
- When to use JavaScript instead

**Consult this** when things aren't working.

## Decision Tree

```
"How do I use Dataview?"
├─ "I need to add fields to my notes"
│  └─ Use: dataview-metadata
│
├─ "I have metadata, now I want to query it"
│  ├─ Simple search/list/filter?
│  │  └─ Use: dataview-queries
│  │
│  └─ Complex logic needed?
│     └─ Use: dataview-javascript
│
└─ "My query/code isn't working"
   └─ Use: dataview-troubleshooting
```

## Usage

Skills are installed in `~/.claude/plugins/` automatically when you load them in Claude Code. Alternatively, you can:

1. Copy `.skill` files to your Claude plugins directory
2. Restart Claude Code
3. Reference the skill by name when working with Dataview

Example:
```
User: "Help me write a query to show all books rated 4+."
Claude: "I'll use the dataview-queries skill..."
```

## Workflow Example

**Scenario:** You want to track books you've read and query them.

1. **dataview-metadata** → Set up field structure
   - YAML frontmatter with author, rating, status
   - Inline fields for inline data
   - Field naming conventions

2. **dataview-queries** → Write a LIST query
   - Query books by rating
   - Sort by completion date
   - Filter by genre

3. **dataview-troubleshooting** (if needed)
   - Query returns no results? Check field names
   - Date filtering acting weird? Check date format

4. **dataview-javascript** (if needed)
   - Want a chart of ratings by genre? Use JS
   - Want custom card layout? Use JS

## When Not to Use These Skills

- For Obsidian plugin development (use plugin documentation)
- For general JavaScript questions (use MDN)
- For plugin architecture questions (not Dataview-specific)

## References Within Skills

Each skill includes detailed references:
- `dataview-metadata` → Field naming rules, Obsidian patterns
- `dataview-queries` → Query patterns by use case
- `dataview-javascript` → Code examples and API reference
- `dataview-troubleshooting` → Error messages and solutions
