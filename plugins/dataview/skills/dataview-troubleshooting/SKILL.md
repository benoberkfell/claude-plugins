---
name: dataview-troubleshooting
description: Debug Dataview queries and metadata issues. Use this skill when queries aren't showing results, fields aren't being indexed, tasks aren't appearing, or you get unexpected behavior. Diagnoses root cause and provides solutions.
---

# Dataview Troubleshooting

When Dataview isn't working as expected, the issue usually falls into one of these categories: metadata isn't indexed, query syntax is wrong, or field names don't match. This skill helps you diagnose and fix it.

## "My Query Shows No Results"

### Step 1: Verify the metadata exists

**Check:** Are your notes actually using the field you're querying?

```markdown
---
author: "Jane Smith"  # This is indexed
---

# My Note

Some content here.  # This is NOT indexed
```

**Solution:** Add metadata explicitly. Use frontmatter or inline fields `[field:: value]`.

### Step 2: Verify field naming

**The problem:** Field names are normalized. `Author Name` becomes `author-name`.

**Check:** Use the exact normalized field name:
- `"Author Name"` → query as `author-name`
- `"Status"` → query as `status`
- `"Book Rating"` → query as `book-rating`

**Solution:** Always use lowercase with hyphens when referring to fields in queries.

### Step 3: Verify the source matches

**The problem:** You're querying a tag or folder that has no matching notes.

```dataview
LIST
FROM #books  # But you never tagged anything with #books
```

**Solution:**
- Check that the tag actually exists in your vault
- Check the exact tag name: `#books` is different from `#book`
- If using folders: `FROM "Book Notes"` must match the exact folder name

### Step 4: Check implicit field availability

**The problem:** Not all implicit fields are available in all query types.

```dataview
LIST
WHERE file.tasks.length > 0  # Works
```

```dataview
TASK
WHERE file.tasks.length > 0  # DOESN'T WORK - use length(rows) instead
```

**Solution:** See the reference guide for which implicit fields work where.

---

## "My Field Isn't Being Indexed"

### Problem: Field is in body text, not metadata

```markdown
# My Note

author: Jane Smith  # This looks like metadata but ISN'T
```

**Solution:** Put it in frontmatter or use inline syntax:

```markdown
---
author: "Jane Smith"
---

# My Note
```

Or inline:

```markdown
# My Note

From [author:: Jane Smith]
```

### Problem: Typo in field name

**Check:** Field names must use hyphens for spaces in inline fields:

```markdown
[book rating:: 4.5]  # WRONG - spaces in inline field names
[book-rating:: 4.5]  # RIGHT
```

**But in YAML frontmatter**, spaces are fine:

```markdown
---
book rating: 4.5  # Fine in YAML
---
```

### Problem: Special characters in field name

```markdown
[my!field:: value]  # WRONG - special characters stripped
[my-field:: value]  # RIGHT
```

**Solution:** Use only letters, numbers, hyphens, underscores.

### Problem: Empty/null values

```markdown
---
author: ""    # Empty string
author:       # Null
---
```

Both are "falsy" but behave differently. Use `default()` in queries:

```dataview
WHERE author != null
```

---

## "Tasks Aren't Appearing"

### Problem: Using TABLE or LIST instead of TASK

```dataview
TABLE file.tasks  # WRONG - shows array, not individual tasks
```

```dataview
TASK  # RIGHT - shows actual tasks
```

**Solution:** Use TASK query type for tasks.

### Problem: Task metadata on wrong line

```markdown
- [ ] My task [due:: 2024-03-15]  # RIGHT
- [ ] My task                      # WRONG - task metadata on next line
[due:: 2024-03-15]
```

**Solution:** Put task metadata `[field:: value]` on the same line as the task checkbox.

### Problem: Filtering on parent task hides children

```dataview
TASK
WHERE priority = "high"  # Only HIGH priority tasks shown
```

If a parent task is low-priority, its high-priority children won't show. This is by design.

**Solution:** If you need child tasks to show regardless of parent, use JS instead.

---

## "Date Queries Aren't Working"

### Problem: Date not recognized as date type

```markdown
---
due: "2024-03-15"  # Text, not date
due: 2024-03-15    # This IS a date
---
```

**Solution:** Remove quotes from dates in frontmatter. Dataview auto-recognizes `YYYY-MM-DD` format.

### Problem: Comparing dates without date() function

```dataview
WHERE due >= 2024-03-15  # WRONG - comparing text to number
WHERE due >= date(2024-03-15)  # RIGHT
```

**Solution:** Wrap dates in `date()` function.

### Problem: Duration math failing

```dataview
WHERE started + dur(30 days) < date(today)  # RIGHT
WHERE started + "30 days" < date(today)     # WRONG
```

**Solution:** Use `dur()` for durations, not strings.

---

## "My Query Is Slow"

### Problem: Querying entire vault with expensive operation

```dataview
TABLE file.outlinks  # Shows all outlinks for every note
```

**Solution:** Add a FROM clause to narrow scope:

```dataview
TABLE file.outlinks
FROM #books
```

### Problem: Complex grouping on large dataset

**Solution:** See `references/performance-tips.md` for optimization strategies.

---

## Quick Diagnostic Checklist

Before reporting an issue:

- [ ] Is the field actually in frontmatter or inline `[field:: value]`?
- [ ] Are you using the normalized field name (lowercase, hyphens)?
- [ ] Does the FROM source (tag/folder) actually exist?
- [ ] Are you using the right query type? (LIST/TABLE for pages, TASK for tasks)
- [ ] For dates: using `date()` function and `YYYY-MM-DD` format?
- [ ] For tasks: is the metadata `[field:: value]` on the same line as the checkbox?
- [ ] Using correct syntax: hyphens in field names, not spaces, in inline fields?

---

## When to Use JavaScript Instead

If your query is too complex or does something "weird", consider using DataviewJS:

- Accessing fields that TASK doesn't support natively
- Custom rendering (beyond list/table/calendar)
- Complex multi-step logic
- Combining data from multiple sources differently

See the `dataview-javascript` skill for details.

---

## Still Stuck?

See `references/error-messages.md` for specific error messages and what they mean.
