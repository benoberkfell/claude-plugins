---
name: dataview-queries
description: Write Dataview Query Language (DQL) queries to search, filter, group, and display vault data. Use this skill when building queries with LIST, TABLE, TASK, or CALENDAR types, filtering with WHERE, grouping with GROUP BY, or sorting with SORT.
---

# Dataview Query Language (DQL)

Dataview queries turn your indexed metadata into views. There are four query types, each with its own output format. Combine them with data commands (FROM, WHERE, GROUP BY, SORT) to get what you need.

## The Four Query Types

### LIST - Bullet point lists

Simplest output. Shows file links or group names, plus one optional piece of extra info.

```dataview
LIST
FROM #books
```

Output:
```
- [The Fifth Season]
- [Educated]
- [Thinking, Fast and Slow]
```

With extra info (shown after the colon):

```dataview
LIST rating
FROM #books
WHERE rating >= 4
```

Output:
```
- [The Fifth Season]: 5
- [Educated]: 4.5
```

**Use LIST when:** You just need a simple list of files with maybe one piece of context.

### TABLE - Tabular data

Show multiple columns of metadata. More structured than LIST.

```dataview
TABLE author, rating, started
FROM #books
WHERE status = "completed"
```

Output:
```
| File | author | rating | started |
|------|--------|--------|---------|
| [The Fifth Season] | N.K. Jemisin | 5 | Jan 15, 2024 |
| [Educated] | Tara Westover | 4.5 | Feb 02, 2024 |
```

**Use TABLE when:** You want to compare multiple fields side-by-side.

### TASK - Interactive task lists

Special query type that works on tasks, not pages. Only query type that can modify your notes (checking tasks).

```dataview
TASK
WHERE !completed AND priority = "high"
GROUP BY file.link
```

Output:
```
[Daily Notes - 2024-03-15] (2)
- [ ] Finish report [due:: 2024-03-16]
- [ ] Review code [priority:: high]

[Projects - Website] (1)
- [ ] Deploy frontend [due:: 2024-03-17]
```

**Use TASK when:** You want to see all unchecked tasks across your vault, filtered by status or metadata.

### CALENDAR - Monthly calendar view

Shows results as dots on a calendar, keyed to a date field. Requires a date field as argument.

```dataview
CALENDAR file.ctime
```

Output:
```
Calendar view with dots showing when files were created
```

**Use CALENDAR when:** You want to visualize when things happened (birthdays, events, creation dates).

## Data Commands

Commands you chain with query types to refine results.

### FROM - Pick which notes to search

```dataview
LIST
FROM #books
```

Can use tags, folders, or specific file names:

```dataview
TABLE author, rating
FROM "Projects" OR "Archive"
```

Multiple sources (folders, tags, files):

```dataview
LIST
FROM #work AND "Active Projects"
```

**Skip FROM to search entire vault.**

### WHERE - Filter by conditions

Only include rows that match.

```dataview
TABLE author, rating
WHERE rating >= 4 AND status = "completed"
```

Common conditions:

```dataview
WHERE !archived                    # not archived
WHERE completed = true
WHERE contains(tags, "#favorite")  # has tag
WHERE author = "Stephen King"
WHERE started >= date(2024-01-01)  # on or after this date
WHERE length(file.inlinks) > 0     # has at least one backlink
```

**Combine with AND / OR:**

```dataview
WHERE (genre = "fiction" OR genre = "scifi") AND rating >= 4
```

### GROUP BY - Group results by a field

Collapses results into groups.

```dataview
LIST rows.file.link
GROUP BY author
```

Output (LIST):
```
- Stephen King
  - [The Shining]
  - [It]
- Neil Gaiman
  - [American Gods]
```

**Access grouped data:** Use `rows.fieldname` and `key` (the group value).

### SORT - Order results

Sort ascending or descending.

```dataview
TABLE author, rating
SORT rating DESC
```

Multiple sort fields:

```dataview
TABLE author, status, rating
SORT status ASC, rating DESC
```

Sorts by status first (A→Z), then by rating within each status (highest first).

## Query Structure

Every query has this order (only query type is required):

```
[QUERY TYPE] [extra info]
[FROM source]
[WHERE condition]
[GROUP BY field]
[SORT field direction]
```

Example:

```dataview
TABLE author, rating, completed
FROM #books
WHERE status = "completed" AND rating >= 4
GROUP BY genre
SORT completed DESC
```

## Inline Queries

Get a single value anywhere in your note using backticks:

```
The current date is `= date(today)`.
I've read `= length(file.where(x => x.tags.contains("read")))` books.
```

These are mini-queries that execute inline. Useful for single values.

## Next Steps

1. **Know your metadata** - Review metadata skill first. Queries only work on indexed fields.
2. **Start simple** - Build queries step by step. Add one filter at a time.
3. **Reference patterns** - See `references/query-patterns.md` for real examples by use case.
