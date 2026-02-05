# Error Messages and Solutions

What specific errors mean and how to fix them.

## "Query could not be parsed"

**Cause:** Syntax error in your query.

**Common issues:**
- Missing query type: `FROM #tag WHERE ...` (need LIST, TABLE, etc first)
- Typo in command: `SORT` vs `SORTS`
- Unmatched quotes: `WHERE author = "John`

**Solution:** Check query syntax. Query must be: `[TYPE] [FROM ...] [WHERE ...] [GROUP BY ...] [SORT ...]`

---

## "Field not found" or "Undefined field"

**Cause:** Field doesn't exist on the page.

**Why it happens:**
- Typo in field name
- Field name not normalized correctly (spaces, punctuation)
- Field is null/empty on this page

**Solution:**
- Check field name spelling
- Use normalized form (lowercase, hyphens): `book-rating` not `Book Rating`
- Use `default()` for optional fields: `default(rating, "unrated")`

---

## "Cannot iterate over null"

**Cause:** Trying to use a field operation on something that doesn't exist.

```dataview
WHERE contains(tags, "#work")  # Fails if tags is null
```

**Solution:** Add null check:

```dataview
WHERE tags && contains(tags, "#work")
```

Or use `default()`:

```dataview
WHERE contains(default(tags, []), "#work")
```

---

## "Invalid date format"

**Cause:** Date not in recognized format.

**Dataview recognizes:**
- `2024-03-15` (ISO format - YYYY-MM-DD)
- `2024/03/15` (with slashes)

**Doesn't recognize:**
- `03-15-2024` (wrong order)
- `March 15, 2024` (written out)
- `"2024-03-15"` (quoted in frontmatter)

**Solution:** Convert to YYYY-MM-DD format without quotes.

---

## "Cannot compare different types"

**Cause:** Comparing incompatible types.

```dataview
WHERE due >= 2024-03-15  # Comparing date to number
WHERE rating > "4"       # Comparing number to text
```

**Solution:** Ensure both sides of comparison are same type:

```dataview
WHERE due >= date(2024-03-15)
WHERE rating > 4
```

---

## "Operator not recognized"

**Cause:** Using wrong operator symbol.

| Wrong | Right |
|-------|-------|
| `==` | `=` |
| `<>` | `!=` |
| `&&` | `AND` |
| `\|\|` | `OR` |

**Solution:** Use correct operators for DQL: `=`, `!=`, `>`, `<`, `>=`, `<=`, `AND`, `OR`

---

## Task-specific Issues

### "Task query returns pages instead of tasks"

**Cause:** Using LIST or TABLE instead of TASK.

```dataview
LIST  # Returns pages
TASK  # Returns tasks (correct for tasks)
```

**Solution:** Use `TASK` query type.

### "Task metadata not being recognized"

**Cause:** Metadata on wrong line or wrong syntax.

```markdown
- [ ] Task name
[due:: 2024-03-15]  # WRONG - on separate line
```

```markdown
- [ ] Task name [due:: 2024-03-15]  # RIGHT - on same line
```

**Solution:** Put task-level metadata on the same line as the checkbox.

### "Task is showing but I didn't filter for it"

**Cause:** Child tasks belong to parent. If parent matches filter, children show too.

```markdown
- [ ] Parent task [priority:: high]
  - [ ] Child task [priority:: low]
```

```dataview
TASK
WHERE priority = "high"
```

Both parent AND child show because child belongs to high-priority parent.

**Solution:** If you only want matching tasks (not children), use JS instead.

---

## GROUP BY Issues

### "GROUP BY returns unexpected structure"

**When grouping LIST:**
```dataview
LIST
GROUP BY author  # Shows author name, with file links nested
```

**When grouping TABLE:**
```dataview
TABLE file.name
GROUP BY author  # Shows author name in first column, then table columns
```

**Solution:** Different query types group differently. This is expected.

### "Can't access grouped values"

When using GROUP BY, access results with:
- `key` - the grouping value (e.g., author name)
- `rows` - the array of matching pages
- `rows.fieldname` - array of field values from each page

```dataview
LIST rows.file.link AS "Books"
GROUP BY author
```

---

## Implicit Field Issues

### "file.tasks is undefined"

**Cause:** Using `file.tasks` outside TASK query.

**In LIST/TABLE:**
```dataview
TABLE file.tasks  # Shows task array, not individual tasks
```

**In TASK:**
```dataview
TASK  # Already operating on task level
```

**Solution:** Use TASK query type if you need task-level filtering.

### "file.inlinks/outlinks not showing"

**Cause:** These are implicit fields that may be empty.

```dataview
WHERE file.inlinks.length > 0  # Only pages with incoming links
```

**Solution:** Filter for non-empty: `length(file.inlinks) > 0`

---

## Calendar-Specific Issues

### "Calendar shows no results"

**Cause:**
- Date field is null/missing
- Date field contains non-date values
- No WHERE clause filtering to valid dates

**Solution:**

```dataview
CALENDAR due
WHERE typeof(due) = "date"  # Only include pages with valid date
```

---

## Performance Issues

### "Query is slow"

**Causes:**
- Querying entire vault with expensive operations
- Complex sorting/grouping on huge dataset
- File operations (inlinks, outlinks) on many pages

**Solution:** Add FROM clause to narrow scope:

```dataview
TABLE file.outlinks
FROM #books  # Only process books, not entire vault
```

---

## DataviewJS Errors

### "dv.table is not a function"

**Cause:** Inside wrong code block type.

```
dataview  ← WRONG - regular query block
dataviewjs ← RIGHT - JavaScript block
```

**Solution:** Use backticks with `dataviewjs`, not `dataview`:

````markdown
```dataviewjs
dv.table(...)
```
````

### "Cannot read property X of undefined"

**Cause:** Trying to access field that doesn't exist.

```javascript
dv.pages("#book").forEach(b => dv.paragraph(b.nonexistent.field));
```

**Solution:** Check before accessing:

```javascript
dv.pages("#book").forEach(b => {
  if (b.author) {
    dv.paragraph(b.author);
  }
});
```

