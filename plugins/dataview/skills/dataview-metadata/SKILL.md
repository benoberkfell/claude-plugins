---
name: dataview-metadata
description: Annotate notes with metadata for Dataview queries. Use this skill when setting up metadata in your vault, adding fields to notes, or structuring data for Dataview queries. Covers YAML frontmatter, inline fields, implicit fields, data types, and Obsidian-specific patterns (daily notes, books, projects, tasks).
---

# Dataview Metadata

Metadata is how Dataview reads your notes. Without it, queries have nothing to work with. This skill covers the three ways to add metadata: frontmatter, inline fields, and built-in implicit fields.

## Three Types of Metadata

### 1. Frontmatter (YAML at top of file)

Frontmatter lives at the very top of your note, between `---` markers:

```markdown
---
author: "Jane Smith"
published: 2024-03-15
rating: 9
tags: [productivity, writing]
---

# My Note

Content goes here...
```

**Use frontmatter for:** Page-level metadata that applies to the whole note. Best for structured data like publication dates, authors, ratings, categories.

### 2. Inline Fields

Inline fields embed metadata within your note content using `[key:: value]` syntax:

```markdown
# My Book Review

From [author:: Stephen King], published [published:: 2023-09-12]

I gave this book [rating:: 4.5] stars. Finished [completed:: true].
```

**Use inline fields for:** Metadata mixed into your narrative. Best when the data makes sense inline with your writing.

### 3. Implicit Fields

Dataview automatically extracts certain fields without you doing anything:

- **Tags**: `#productivity #work` → available as `tags` field
- **Links**: `[[Note Title]]` → available as `file.inlinks` and `file.outlinks`
- **Tasks**: `- [ ] Task text [due:: 2024-03-20]` → available in TASK queries
- **File properties**: `file.name`, `file.path`, `file.folder`, `file.ctime`, `file.mtime`

**Use implicit fields for:** Data you're already writing. No extra work needed.

## Field Names and Data Types

See `references/field-reference.md` for:
- Field naming rules (spaces, punctuation, case sensitivity)
- All supported data types with examples (dates, durations, links, objects, lists)
- Type conversions and edge cases

## Metadata Patterns in Obsidian

Common setups for different note types. See `references/obsidian-patterns.md` for:

- Daily notes (tracking sleep, mood, habits)
- Book tracking (author, rating, status)
- Project management (status, due dates, assignee)
- Task tracking (priority, due date, completion)
- Recipe collection (ingredients, prep time, tags)
- Learning notes (source, review date, difficulty)

Each pattern shows frontmatter + inline field examples and how to query them.

## Quick Start

1. **Decide what data matters** - What do you want to query later?
2. **Choose where to store it** - Frontmatter (structured) or inline (narrative)?
3. **Name your field** - Use lowercase, hyphens for spaces: `book-rating`, `author-name`
4. **Pick a data type** - Date, number, text, link, etc.
5. **Query it** - Use the field in a Dataview query

Example:

```markdown
---
type: book
author: "N.K. Jemisin"
---

# The Broken Earth Series

Started [started:: 2024-01-15], finished [completed:: 2024-03-10].
Gave it [rating:: 5] stars.
```

Then query:

```dataview
TABLE author, started, completed, rating
FROM #book
WHERE rating >= 4
```

## Common Gotchas

- **Metadata not appearing in queries?** Make sure you're using the field name correctly. Dataview auto-normalizes names to lowercase with hyphens.
- **Task metadata not working?** Task-level fields go `[field:: value]` inside the task line itself.
- **Dates not showing as dates?** Use the `date()` function in queries, or prefix with `date()` when defining. Dataview recognizes `2024-03-15` format automatically.
- **Need to query implicit fields?** Use `file.*` prefix: `file.name`, `file.tags`, `file.inlinks`, etc.
