# Common Query Patterns

Real-world examples for different use cases.

## Date Range Queries

### "Show me all tasks due this week"
```dataview
TASK
WHERE !completed AND due >= date(today) AND due < date(today) + dur(7 days)
SORT due ASC
```

### "Books I finished this year"
```dataview
TABLE author, rating, completed
FROM #book
WHERE status = "completed" AND completed >= date(2024-01-01)
SORT completed DESC
```

### "Notes created in the last 30 days"
```dataview
LIST file.ctime
WHERE file.ctime >= date(today) - dur(30 days)
SORT file.ctime DESC
```

---

## Filtering with Multiple Conditions

### "High-priority incomplete tasks from work"
```dataview
TASK
WHERE !completed AND priority = "high" AND contains(tags, "#work")
SORT due ASC
```

### "Books rated 4+ stars by favorite authors"
```dataview
TABLE author, rating, completed
FROM #book
WHERE rating >= 4 AND (author = "Neil Gaiman" OR author = "Brandon Sanderson")
SORT rating DESC
```

### "Active projects not blocked"
```dataview
LIST status, progress
FROM "Projects"
WHERE status = "active" AND !blocked
SORT progress DESC
```

---

## Grouping and Nesting

### "Uncompleted tasks grouped by file"
```dataview
TASK
WHERE !completed
GROUP BY file.link
```

### "Books grouped by author showing count"
```dataview
LIST "(" + length(rows) + " books)"
FROM #book
GROUP BY author
SORT length(rows) DESC
```

### "Projects grouped by status with progress"
```dataview
TABLE rows.file.link AS "Projects", avg(rows.progress) AS "Avg Progress"
FROM "Projects"
GROUP BY status
```

---

## Using Computed Fields

### "Calculate how long you've been reading a book"
```dataview
TABLE author, started, completed, completed - started AS "Duration"
FROM #book
WHERE status = "completed"
SORT "Duration" DESC
```

### "Show books and days until they're due (for loans)"
```dataview
TABLE author, due-date, due-date - date(today) AS "Days Left"
FROM #book
WHERE due-date >= date(today)
SORT "Days Left" ASC
```

### "Books with title + rating in one column"
```dataview
LIST file.link + " ⭐" + string(rating)
FROM #book
WHERE rating
SORT rating DESC
```

---

## Implicit Field Queries

### "Show me notes by folder"
```dataview
LIST
GROUP BY file.folder
```

### "Notes with no tags"
```dataview
LIST
WHERE length(file.tags) = 0
```

### "Backlinked notes (what links to me)"
```dataview
LIST length(file.inlinks)
SORT length(file.inlinks) DESC
```

### "Most linked-to notes"
```dataview
LIST length(file.outlinks) AS "Links"
SORT "Links" DESC
```

---

## Array/List Field Queries

### "Tasks with specific tag"
```dataview
TASK
WHERE contains(tags, "#urgent")
```

### "Filter by list membership"
```dataview
TABLE topic, difficulty
FROM #learning
WHERE contains(subjects, "JavaScript")
```

### "Multiple tags (any match)"
```dataview
LIST tags
WHERE contains(tags, "#favorite") OR contains(tags, "#toread")
```

---

## Task-Specific Queries

### "Group tasks by due date"
```dataview
TASK
WHERE !completed
GROUP BY due
```

### "Tasks with metadata on the task itself"
```dataview
TASK
WHERE priority = "high" AND completed = false
```

### "Recurring tasks (no due date)"
```dataview
TASK
WHERE due = null
```

---

## List Without ID (Cleaner Output)

### "Just show author names, not file links"
```dataview
LIST WITHOUT ID author
FROM #book
GROUP BY author
```

### "Show only computed values"
```dataview
LIST WITHOUT ID file.link + ": " + string(rating) + "/5"
FROM #book
WHERE rating
SORT rating DESC
```

---

## Calendar Examples

### "Show when you created notes"
```dataview
CALENDAR file.ctime
```

### "Show when books were finished"
```dataview
CALENDAR completed
FROM #book
WHERE typeof(completed) = "date"
```

### "Show upcoming events"
```dataview
CALENDAR event-date
WHERE event-date >= date(today)
```

---

## Common Operators Reference

| Operator | Meaning | Example |
|----------|---------|---------|
| `=` | Equals | `status = "completed"` |
| `!=` | Not equals | `status != "archived"` |
| `>` | Greater than | `rating > 3` |
| `<` | Less than | `rating < 4` |
| `>=` | Greater or equal | `due >= date(today)` |
| `<=` | Less or equal | `pages <= 300` |
| `+` | Add/concatenate | `"Book: " + file.link` |
| `-` | Subtract | `due - date(today)` |
| `*` | Multiply | `rating * 2` |
| `/` | Divide | `pages / 2` |
| `AND` | Both conditions | `rating >= 4 AND !archived` |
| `OR` | Either condition | `type = "book" OR type = "article"` |
| `!` | Not/negate | `!completed` |

---

## Common Functions

| Function | Use | Example |
|----------|-----|---------|
| `contains(list, value)` | Check if value in list | `contains(tags, "#favorite")` |
| `length(array)` | Count items | `length(file.outlinks)` |
| `date(value)` | Parse/format date | `date(today)` |
| `dur(value)` | Parse duration | `dur(3 days)` |
| `default(value, fallback)` | Use fallback if null | `default(rating, 0)` |
| `upper(text)` | Uppercase | `upper(author)` |
| `lower(text)` | Lowercase | `lower(author)` |
| `typeof(value)` | Check type | `typeof(due) = "date"` |
| `typeof(value)` | Supported types | `"string"`, `"number"`, `"date"`, `"array"`, `"object"`, `"link"` |

